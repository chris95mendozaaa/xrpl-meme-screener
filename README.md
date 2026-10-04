render.yaml
"""Minnow Live: non-custodial XRP -> meme coin swap on the XRPL (mainnet by default).
Users sign every transaction in their own Xaman wallet. This server never sees a secret key."""
import json, os
from decimal import Decimal
import httpx
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel
from xrpl.clients import JsonRpcClient
from xrpl.models.requests import BookOffers, AccountLines, AccountInfo
from xrpl.models.currencies import XRP, IssuedCurrency
from xrpl.core.addresscodec import is_valid_classic_address

NETWORKS = {"mainnet": "https://xrplcluster.com", "testnet": "https://s.altnet.rippletest.net:51234"}
NET = os.getenv("XRPL_NETWORK", "mainnet")
client = JsonRpcClient(NETWORKS[NET])
MAX_XRP = float(os.getenv("MAX_XRP_PER_SWAP", "100"))
TF_SELL, TF_IOC, TF_NO_RIPPLE = 0x00080000, 0x00020000, 0x00020000 * 0 + 131072


def hexcode(c: str) -> str:
    return c.encode().hex().upper().ljust(40, "0")

# Issuers from the projects' own pages / XPMarket. VERIFY each on bithomp.com or xpmarket.com before going live.
TOKENS = {
    "XRdoge": dict(issuer="rLqUC2eCPohYvJCEBJ77eCCqVL2uEiczjA", currency=hexcode("XRdoge"),
                   note="Issuer blackholed (no new supply). Listed on several exchanges."),
    "XFLOKI": dict(issuer="rUtXeAXonpFpgKubAa7LxcLd7NFep92T1t", currency=hexcode("XFLOKI"),
                   note="Very small market. Issuer found on one source only: verify before use."),
}
app = FastAPI(title="Minnow Live")


def tok(symbol: str) -> dict:
    if symbol not in TOKENS:
        raise HTTPException(404, "Unknown token.")
    return TOKENS[symbol]


def check_account(a: str):
    if not is_valid_classic_address(a):
        raise HTTPException(400, "Invalid XRPL address.")


def xaman(method, path, body=None):
    key, secret = os.getenv("XAMAN_API_KEY"), os.getenv("XAMAN_API_SECRET")
    if not key or not secret:
        raise HTTPException(503, "Wallet connection is not configured on the server.")
    try:
        r = httpx.request(method, "https://xumm.app/api/v1/platform" + path, timeout=15,
                          headers={"X-API-Key": key, "X-API-Secret": secret}, json=body)
        r.raise_for_status()
        return r.json()
    except httpx.HTTPError:
        raise HTTPException(502, "Xaman did not respond. Try again.")


def make_quote(symbol: str, xrp: float):
    t = tok(symbol)
    try:
        res = client.request(BookOffers(taker_gets=IssuedCurrency(currency=t["currency"], issuer=t["issuer"]),
                                        taker_pays=XRP(), limit=60))
    except Exception:
        raise HTTPException(503, "Could not reach the XRPL. Try again.")
    left, out, best = xrp * 1e6, 0.0, None
    for o in res.result.get("offers", []):
        gets = o.get("taker_gets_funded", o["TakerGets"])
        avail = float(gets["value"])
        rate = int(o["TakerPays"]) / float(o["TakerGets"]["value"])  # drops per token
        best = best or rate
        n = min(avail, left / rate)
        out += n; left -= n * rate
        if left <= 1:
            break
    if out <= 0:
        raise HTTPException(409, "No order-book liquidity for this token right now.")
    spent = xrp * 1e6 - left
    avg = spent / out
    return dict(tokens=out, avg_xrp_per_token=avg / 1e6, impact_pct=round((avg / best - 1) * 100, 2),
                filled_pct=round(spent / (xrp * 1e6) * 100, 1))


def fmt(x: float) -> str:
    return format(Decimal("%.12g" % x), "f")


@app.get("/api/config")
def config():
    return dict(network=NET, max_xrp=MAX_XRP, tokens={k: dict(issuer=v["issuer"], note=v["note"]) for k, v in TOKENS.items()})


@app.get("/api/balance")
def balance(account: str):
    check_account(account)
    try:
        info = client.request(AccountInfo(account=account)).result
        xrp = int(info["account_data"]["Balance"]) / 1e6
        lines = client.request(AccountLines(account=account)).result.get("lines", [])
    except Exception:
        raise HTTPException(404, "Account not found or not activated on this network.")
    held = {k: next((float(l["balance"]) for l in lines if l["account"] == v["issuer"] and l["currency"] == v["currency"]), None)
            for k, v in TOKENS.items()}
    return dict(xrp=xrp, tokens=held)


@app.get("/api/quote")
def quote(symbol: str, xrp: float):
    if not 1 <= xrp <= MAX_XRP:
        raise HTTPException(400, f"Amount must be 1 to {MAX_XRP:g} XRP.")
    return make_quote(symbol, xrp)


@app.post("/api/signin")
def signin():
    p = xaman("POST", "/payload", {"txjson": {"TransactionType": "SignIn"}})
    return dict(uuid=p["uuid"], link=p["next"]["always"], qr=p["refs"]["qr_png"])


@app.get("/api/payload/{uuid}")
def payload(uuid: str):
    p = xaman("GET", "/payload/" + uuid)
    m, r = p["meta"], p.get("response") or {}
    return dict(signed=bool(m.get("signed")), resolved=bool(m.get("resolved")), expired=bool(m.get("expired")),
                account=r.get("account"), txid=r.get("txid"))


class Swap(BaseModel):
    account: str
    symbol: str
    xrp: float
    slippage_pct: float = 5


@app.post("/api/swap")
def swap(s: Swap):
    """Builds the exact transactions and sends them to the user's Xaman for approval."""
    check_account(s.account)
    t = tok(s.symbol)
    if not 1 <= s.xrp <= MAX_XRP:
        raise HTTPException(400, f"Amount must be 1 to {MAX_XRP:g} XRP.")
    if not 0.5 <= s.slippage_pct <= 20:
        raise HTTPException(400, "Slippage must be 0.5% to 20%.")
    q = make_quote(s.symbol, s.xrp)
    min_out = q["tokens"] * (1 - s.slippage_pct / 100)
    lines = client.request(AccountLines(account=s.account, peer=t["issuer"])).result.get("lines", [])
    has_line = any(l["currency"] == t["currency"] for l in lines)
    steps = []
    if not has_line:
        steps.append(("Allow " + s.symbol + " in your wallet (trust line)", {
            "TransactionType": "TrustSet", "Account": s.account, "Flags": TF_NO_RIPPLE,
            "LimitAmount": {"currency": t["currency"], "issuer": t["issuer"], "value": "999999999999999"}}))
    steps.append((f"Swap {s.xrp:g} XRP for at least {min_out:,.2f} {s.symbol}", {
        "TransactionType": "OfferCreate", "Account": s.account, "Flags": TF_SELL | TF_IOC,
        "TakerGets": str(int(s.xrp * 1e6)),
        "TakerPays": {"currency": t["currency"], "issuer": t["issuer"], "value": fmt(min_out)}}))
    out = []
    for text, tx in steps:
        p = xaman("POST", "/payload", {"txjson": tx, "custom_meta": {"instruction": text}})
        out.append(dict(text=text, uuid=p["uuid"], link=p["next"]["always"], qr=p["refs"]["qr_png"]))
    return dict(steps=out, quote=q, min_out=min_out)


def rules(q, symbol):
    flags = []
    if q["impact_pct"] > 5: flags.append(f"price impact {q['impact_pct']}%")
    if q["filled_pct"] < 100: flags.append("order book too thin to fill the full amount")
    if symbol == "XFLOKI": flags.append("very small market")
    return dict(summary="Rule-based check: " + (", ".join(flags) or "no major flags") + ". Meme coins can lose most of their value quickly.", source="rules")


class Risk(BaseModel):
    symbol: str
    xrp: float


@app.post("/api/risk")
def risk(r: Risk):
    q = make_quote(r.symbol, r.xrp)
    if not os.getenv("ANTHROPIC_API_KEY"):
        return rules(q, r.symbol)
    try:
        import anthropic
        m = anthropic.Anthropic().messages.create(
            model="claude-sonnet-5-5", max_tokens=300,
            system="You explain risk to first-time crypto buyers in 3 plain sentences. Mention price impact and liquidity. Never promise returns or give financial advice.",
            messages=[{"role": "user", "content": json.dumps(dict(token=r.symbol, xrp=r.xrp, quote=q, info=TOKENS[r.symbol]["note"]))}])
        return dict(summary=m.content[0].text, source="claude")
    except Exception:
        return rules(q, r.symbol)

app.mount("/", StaticFiles(directory=os.path.join(os.path.dirname(__file__), "..", "static"), html=True))
