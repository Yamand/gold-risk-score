# Gold Risk Score

A daily composite 0–1 risk score for gold, built from OKX's free public
candlestick data for XAUT-USDT (Tether Gold — a token backed 1:1 by physical
gold). Static site + GitHub Actions, same pattern as the BTC risk score.

**Live idea:** `0` = cheap, accumulate harder. `1` = expensive, reduce buys /
start distributing once holdings clear your $500 sell-tier threshold.

## Data source: how we got here

This went through three data sources before landing on OKX — worth knowing
in case you ever need to swap again:

1. **Stooq** (`stooq.com/q/d/l/`) — free XAUUSD CSV, no key. Chosen first as
   the closest analog to how the BTC script uses Binance. Abandoned because
   it couldn't be verified as reliable from the dev environment used to
   build this (network access issues in that specific sandbox, not
   necessarily a problem with Stooq itself).
2. **Binance XAUTUSDT** — same proven fetch pattern as the BTC script
   (`/api/v3/klines`, geo-block-resistant mirror + fallback). Reliable, but
   has less historical depth than OKX's listing of the same token.
3. **OKX XAUT-USDT** (current) — more history than Binance for this token.
   Uses OKX's `/api/v5/market/history-candles` endpoint, which is
   specifically built for paging backward through older data (OKX's
   `/market/candles` endpoint only covers a limited recent window).

**Caveat: this hasn't been run against live OKX data from the environment
that built it** — OKX wasn't reachable from that sandbox (network
allowlist), only simulated against OKX's documented response shape. Run it
locally once and sanity-check the printed date range and first few output
rows before trusting it for real use.

## How the score is built

Same four-component methodology as the BTC version, each normalized to 0–1
via **expanding historical percentile rank** (today's raw value ranked
against every prior day back to the start of data):

| Component | Weight | What it captures |
|---|---|---|
| Log-regression band position | 35% | Price vs. long-term log-log trend line, refit each run |
| 200-day MA multiple | 25% | Price stretch vs. long-term trend (price ÷ 200d MA) |
| RSI-14 (daily) | 20% | Short-term overbought/oversold |
| Volatility-adjusted momentum | 20% | 30d return ÷ 30d realized volatility |

Composite = weighted sum of the four, clipped to [0, 1].

**Caveat on the log-regression component — read this one.** The BTC
version's log-regression fits price against days-since-genesis-block, a
reasonable proxy for a BTC-style power-law adoption curve. Gold has no such
growth-curve dynamic — it's a mature, millennia-liquid asset, not an
adoption curve. This component is included for structural parity with the
BTC script (same code path, same weight), using the Nixon Shock
(1971-08-15, the end of dollar/gold convertibility) as a fixed reference
date instead of a "growth curve" anchor. Treat this component's signal more
loosely than you would for BTC — it's really a very-long-horizon log-linear
trend deviation, not a claim that gold "should" grow along a power law.

**Caveat on history depth.** XAUT-USDT (even on OKX, with more history than
Binance's listing) only has as much price history as the token itself has
existed — nowhere close to gold's real multi-decade+ price history. The
first ~200-260 days of any fetched series won't produce a score at all
(warm-up period for the 200-day MA and percentile-rank windows), and the
log-regression component specifically will be least meaningful in roughly
the first year after that, before the fit has enough data to anchor a
stable trend line.

## Repo structure

```
gold_risk_score.py                  # fetch + compute + write data/gold_risk_history.json
data/gold_risk_history.json         # generated — one row per day, public/scored output
data/gold_prices_raw.json           # generated — full raw close-price cache (internal use)
gold.html                           # static site, reads data/ directly, Chart.js
.github/workflows/daily-update-gold.yml   # cron job, runs gold_risk_score.py --update daily
```

**Why two data files?** `gold_risk_history.json` only contains rows once all
four components have enough history to compute (~200-260 day warm-up), so
the earliest fetched days never appear there. `gold_prices_raw.json` keeps
*every* fetched date/close, including those warm-up days — `--update` merges
from that file so the regression fit always sees the true full price series,
not a truncated one. (Skipping this distinction was a real bug caught while
building this: reconstructing prices from the scored file alone silently
drops the earliest data and can shift the regression fit enough to flip a
DCA zone near a boundary.)

## Setup

1. Push this repo (or these files, alongside the BTC ones) to GitHub, enable
   **GitHub Pages** (Settings → Pages → Deploy from branch → `main` / root).
2. Run the full backfill once, locally or via Actions "Run workflow" with
   the `--update` flag removed, so `data/gold_risk_history.json` and
   `data/gold_prices_raw.json` both exist before the site goes live. Check
   the printed date range on this first run — that's your real confirmation
   of how much history OKX actually has for this pair.
3. The daily workflow (`daily-update-gold.yml`) runs automatically at 00:20
   UTC, pulls the last ~400 days from OKX, recomputes over the full merged
   price series, and commits both updated JSON files. GitHub Pages
   redeploys automatically on push.

### Local run

```bash
pip install pandas numpy requests
python gold_risk_score.py            # full history backfill (first run)
python gold_risk_score.py --update   # fast daily run (last ~400 days only)
python -m http.server 8000           # then open localhost:8000/gold.html
```

## Notes

- No API key required — OKX's `/api/v5/market/history-candles` endpoint is
  public.
- The score is descriptive, not a signal to auto-trade on. Same discipline
  as the BTC version: it scales buy size, doesn't override the plan.
- XAUT is a *token* tracking gold, not gold itself — it can (rarely) trade
  at a small premium/discount to spot due to redemption mechanics, exchange
  liquidity, or counterparty considerations. Fine for a directional risk
  score; keep that distinction in mind if you're using this for anything
  more precise.
- Not financial advice.
