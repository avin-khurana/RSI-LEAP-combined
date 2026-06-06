# LEAPS Unified Scanner

Scans the top 40 large-cap US stocks (market cap > $100B) for two distinct trade setups, then surfaces matching options contracts with real implied volatility, Greeks, and ready-to-paste trade tracker rows.

Runs automatically on weekdays at 5 PM EST via GitHub Actions and emails you the results.

---

## Two signal modes

### OVERSOLD — contrarian reversal
Fires when RSI(14) was below 30 on **any** of the last 3 trading days. No IV filter is applied — backtesting showed the IV filter removes 66–74% of winning contrarian signals.

### MOMENTUM — trend continuation
Fires when all four technical conditions align:
- Price above SMA(200)
- RSI(14) between 38–62 (neutral, not extended)
- MACD histogram crossed above zero (fresh bullish cross)
- Volume spike ≥ 115% of 50-day average

Momentum signals are further tagged **COMBO** (tech + Grade A/B fundamentals) and **STRONG** (COMBO + IV Rank < 40 + VIX < 28 + 30+ days to earnings + positive RS vs SPY).

---

## What it surfaces per signal

| Output | Description |
|--------|-------------|
| **12M LEAPS** | Nearest expiry ≥ 360 days, ITM call closest to Δ ≈ 0.70 |
| **3M ATM call** | Nearest expiry to 91 days, at-the-money call (momentum/quick-profit play) |
| **Stock target ▲** | Stock price at which the LEAPS gains +50%, back-solved via Black-Scholes |
| **Stock stop ▼** | Entry stock × 0.90 — consistent 10% stock-level stop |
| **IV Rank** | Current 30-day realized vol vs its 52-week range, with buy-guidance tier |
| **Vega-gain estimate** | Estimated % option gain if IV reverts from current to 52-week average |
| **CSV row** | Ready-to-paste row for `my_trades.csv` / trade tracker |

---

## IV Rank tiers

| Rank | Label | Guidance |
|------|-------|----------|
| < 20 | **IDEAL BUY** | IV near 52-wk low — cheapest options of the year |
| 20–40 | **GOOD** | IV below average — favorable for buying |
| 40–60 | **FAIR** | IV near average — neutral entry |
| 60–80 | **ABOVE AVG** | IV above average — consider spreads over naked longs |
| > 80 | **EXPENSIVE** | IV near 52-wk high — poor time to buy naked options |

---

## How implied volatility is resolved

Three-tier fallback per contract:

1. **Newton-Raphson back-solve** (primary) — finds the σ where `BS_call(σ) = bid_ask_mid`. Converges in ~5–10 iterations. Accurate to floating-point precision.
2. **Yahoo's `impliedVolatility` field** — fallback if the solver returns `None` (e.g. zero-width spread after hours).
3. **30-day realized volatility** — last resort when neither chain source is usable.

Yahoo's pre-computed IV for deep ITM LEAPS can be off by several points (observed: 34.7% Yahoo vs 27.0% solved for the same contract).

---

## Setup

### Local

```bash
pip install -r requirements.txt
```

Set environment variables for email (optional — results always print to stdout):

```bash
export EMAIL_FROM="you@gmail.com"
export EMAIL_APP_PASSWORD="your-16-char-app-password"   # Gmail App Password, not your login password
export EMAIL_TO="you@gmail.com"
```

For Telegram alerts (optional):

```bash
export TELEGRAM_TOKEN="bot<token>"
export TELEGRAM_CHAT_ID="<chat_id>"
```

### Run locally

```bash
python leaps_scanner.py               # scan + email + telegram
python leaps_scanner.py --no-email    # stdout only, no alerts
```

Results are also saved to `leaps_scan_results.csv` and `leaps_scan.html`.

---

## GitHub Actions — automated daily email

The workflow at `.github/workflows/scan.yml` runs the scanner every weekday at **5 PM EST (22:00 UTC)** and sends the HTML email report automatically.

### One-time setup

1. Push this repo to GitHub.
2. Go to **Settings → Secrets and variables → Actions** and add:

| Secret name | Value |
|-------------|-------|
| `EMAIL_FROM` | Your Gmail address |
| `EMAIL_APP_PASSWORD` | Your Gmail [App Password](https://myaccount.google.com/apppasswords) |
| `EMAIL_TO` | Address to receive the report (can be the same) |

3. Enable Actions if prompted (**Actions** tab → "I understand my workflows, go ahead and enable them").

The workflow triggers automatically from then on. You can also run it manually from **Actions → LEAPS Scanner → Run workflow**.

> **Note:** 22:00 UTC = 5 PM EST (UTC−5). During EDT (UTC−4) the run arrives at 6 PM ET. Adjust the cron to `0 21 * * 1-5` if you want 5 PM year-round during summer.

---

## Key configuration

All tunable constants are at the top of `leaps_scanner.py`:

```python
MIN_MARKET_CAP    = 100e9    # $100B minimum to enter universe
TOP_N             = 40       # tickers to scan (sorted by market cap)

RSI_PERIOD        = 14       # RSI lookback
RSI_THRESHOLD     = 30.0     # oversold trigger
RSI_LOOKBACK      = 3        # fires if RSI < threshold on any of last N days

TARGET_DELTA      = 0.70     # ITM call delta target for LEAPS
MIN_LEAPS_DAYS    = 360      # minimum days to expiry
RISK_FREE_RATE    = 0.045    # used in Black-Scholes

PROFIT_TARGET_PCT = 0.50     # +50% option gain → stock target
STOP_STOCK_PCT    = 0.10     # −10% stock drop → stop level

VIX_MAX           = 28       # momentum IV/VIX gate
IV_RANK_MAX       = 40       # momentum IV Rank gate
RSI_MOM_LO        = 38       # RSI neutral band (low)
RSI_MOM_HI        = 62       # RSI neutral band (high)

ACCOUNT_SIZE      = 100_000  # used for position sizing
MAX_POS_PCT       = 0.05     # max 5% of account per position
```

---

## RSI implementation

Uses Wilder's smoothing (`EMA with com = period − 1`) to match TradingView's RSI exactly — not the simple rolling-average approximation used by most libraries.

---

## Output example

```
════════════════════════════════════════════════════════════════════════
  OVERSOLD SIGNALS (1) — RSI < 30  [no IV filter]
════════════════════════════════════════════════════════════════════════

  ────────────────────────────────────────────────────────────────────────
  WMT     RSI(last 3d)=[29.1 / 33.0 / 32.0]  Grade:B  IV Rank=22/100 [GOOD]
  Price=$115.86  IV guidance: IV below average — favorable for buying
  12M LEAPS: $105 Call (2027-06-17)  IV=27%  Prem=$21.35  BE=+9.1%  2x=$4,270
  Target: stock ≥ $128.40 (+10.8%)  → option +50% ($32.03)
  Stop  : stock ≤ $104.27 (−10.0%)  → stock-level stop
  ┌─ Paste into my_trades.csv ──────────────────────────────────
  │ WMT,2026-06-06,115.86,105,C,2027-06-17,21.35,128.40,104.27,32.03,27,OPEN,,,,,
  └─────────────────────────────────────────────────────────────
  3M ATM: $115 Call (2026-09-19)  Prem=$6.40  2x  BE=+5.5%
```

The HTML email adds per-ticker Greeks (Δ, Θ/day, Vega/1% IV), the IV 52-week range (low → avg → high), vega-gain estimate, Target ▲ / Stop ▼ levels, and a near-misses watch table (RSI 30–35 and 6+/9 momentum conditions).

---

## Fundamentals scoring

Each ticker is graded A / B / C from up to 5 factors (20 pts each, max 100):

| Factor | Points |
|--------|--------|
| Forward P/E < 20 | 20 pts (10 pts if < 35) |
| Revenue growth > 20% | 20 pts (10 pts if > 8%) |
| Profit margin > 20% | 20 pts (10 pts if > 8%) |
| Debt/Equity < 50 | 20 pts (10 pts if < 150) |
| Earnings growth > 15% | 20 pts (10 pts if > 5%) |

Grade A ≥ 70, Grade B ≥ 40, Grade C < 40. Oversold signals with Grade A and RSI < 35 are starred [Fund A ★].

---

## Disclaimer

Not financial advice. Data sourced from Yahoo Finance via `yfinance`. Options data may be delayed or incomplete outside market hours.
