# Polymarket Trading Bots

Three standalone trading bots for [Polymarket](https://polymarket.com) prediction
markets. Each is a single binary — no Python, no Node, no Docker. Just download,
configure, and run.

| Bot | What it does | Needs LLM? | Binary |
|---|---|---|---|
| [Copy Trader](#copy-trader) | Follow top Polymarket traders automatically | No | 11 MB |
| [BTC Predictor](#btc-predictor) | ML-powered BTC direction prediction | No (optional) | 7.5 MB* |
| [Topic Bet](#topic-bet) | Bet on any topic using LLM analysis | Yes | 11 MB |

*BTC Predictor is Python-based with pre-trained models included.

---

## Copy Trader

Automatically copies trades from the top-performing Polymarket traders.
Scans the leaderboard daily, identifies the best traders by profit, and
mirrors their positions.

### How it works

```
Polymarket Leaderboard (top 100 by PnL)
         |
   Auto-select top 10 traders
         |
   Monitor their trades every 60s
         |
   Copy each trade at your size ($10 default)
         |
   Track P&L as markets resolve
         |
   Dashboard at http://localhost:8080
```

### Two editions

| | Trader Edition | Viewer Edition |
|---|---|---|
| For | Canadian / European users | US users |
| Trades? | Yes (with your Polymarket API key) | No (paper trading only) |
| API key? | Yes (yours) | No |
| File | `polymarket-trader.exe` | `polymarket-us.exe` |

> US users: Polymarket Global is geo-restricted. The Viewer Edition shows
> what top traders are doing with simulated P&L tracking.

### Quick start (Trader Edition)

```bash
# 1. Edit .env — add your Polymarket API key
# 2. Run
polymarket-trader.exe -mode=trader
# 3. Dashboard: http://localhost:8080
```

### Quick start (Viewer Edition)

```bash
# Just run — no API key needed
polymarket-us.exe -mode=us
# Dashboard: http://localhost:8080
```

### Commands

```
polymarket-trader.exe -mode=trader                  Start trading + dashboard
polymarket-trader.exe -mode=trader -once             One cycle then exit
polymarket-trader.exe -mode=trader -leaderboard-scan Scan leaderboard
polymarket-trader.exe -mode=trader -status           Show status

polymarket-us.exe -mode=us                          Start viewer + dashboard
polymarket-us.exe -mode=us -status                  Show status
```

### Configuration

```env
# Trader Edition only
POLY_API_KEY=your-polymarket-api-key
POLY_API_SECRET=your-api-secret
POLY_PASSPHRASE=your-passphrase

# Both editions
DRY_RUN=true                    # Paper trade first!
COPY_SIZE_USD=10                # $ per copy trade
COPY_MAX_DAILY_USD=500          # Daily cap
LEADERBOARD_TOP_N=100           # Traders to track
LEADERBOARD_AUTO_COPY_TOP_N=10  # Top N to auto-copy
DASHBOARD_PORT=8080
```

### Dashboard

Built-in at http://localhost:8080 with:
- Stats cards (trades, P&L, win rate)
- Cumulative P&L chart
- Daily P&L bar chart
- Top traders table (sortable by PnL all-time/monthly/weekly, volume)
- Recent trades with leader, side, status, P&L

Auto-refreshes every 30 seconds.

### Safety

- DRY_RUN=true by default
- Daily spending cap (COPY_MAX_DAILY_USD)
- Per-trade size limit (COPY_SIZE_USD)
- Your API key stays on your machine — never sent anywhere except Polymarket

---

## BTC Predictor

Predicts Bitcoin's short-term direction using machine learning (XGBoost +
RandomForest ensemble) trained on Binance price data. Bets on Polymarket
"BTC Up or Down" markets when the model disagrees with the market price.

**No LLM required.** All predictions are made by local ML models. Claude API
is optional, only consulted for ambiguous high-stakes decisions.

**No Binance API key required.** All market data is public.

### How it works

```
Binance US (free public data)
     |
  Fetch OHLCV candles (1m, 5m, 15m, 1h)
     |
  Compute 36 technical indicators
  (RSI, MACD, Bollinger, ATR, OBV...)
     |
  XGBoost + RandomForest ensemble
  predicts P(BTC goes UP)
     |
  If confidence > 15%: paper bet
  If edge > 5% vs Polymarket: real bet
     |
  Auto-resolve after time window
  Track P&L
     |
  Dashboard at http://localhost:8090
```

### Quick start

```bash
# 1. Install Python dependencies
pip install -r requirements.txt

# 2. Configure
cp .env.example .env

# 3. Download 90 days of BTC data (~30 seconds)
python -m btc_bot --deep-backfill 90

# 4. Train ML models (~10 seconds)
python -m btc_bot --train

# 5. Start bot + dashboard
python -m btc_bot --loop
# Dashboard: http://localhost:8090
```

Pre-trained models are included — step 3-4 are optional (but recommended for
fresh data).

### Commands

```
python -m btc_bot --deep-backfill 90   Download 90 days of Binance history
python -m btc_bot --train              Train ML models
python -m btc_bot --predict            Show current prediction
python -m btc_bot --once               One full cycle
python -m btc_bot --loop               Continuous trading + dashboard
python -m btc_bot --status             Show status
python -m btc_bot --backtest           Backtest on historical data
```

### Configuration

```env
BTC_BOT_DRY_RUN=true              # Paper trading
BINANCE_BASE_URL=https://api.binance.us
BTC_RETRAIN_HOUR=12               # Daily retrain at noon UTC
BTC_MIN_EDGE_PCT=0.05             # 5% minimum edge
BTC_MAX_POSITION_PER_MARKET_USD=100
BTC_MAX_TOTAL_EXPOSURE_USD=500
BTC_DASHBOARD_PORT=8090

# Optional: Claude API for complex decisions
# BTC_USE_CLAUDE_ADVISOR=true
# ANTHROPIC_API_KEY=sk-ant-...
```

### ML Model

- **Features:** 36 technical indicators (trend, momentum, volatility, volume, microstructure)
- **Models:** XGBoost (60% weight) + RandomForest (40% weight) ensemble
- **Calibration:** Isotonic regression for well-calibrated probabilities
- **Training data:** Binance US klines, auto-backfills up to years of history
- **Retrain:** Daily at configurable hour (default 12:00 UTC)
- **Performance:** 55-62% accuracy, AUC 0.60-0.65 (improves with more data)

### Dashboard

Built-in at http://localhost:8090 with:
- BTC price + current direction + confidence
- P(Up) probability chart over time
- BTC price chart
- ML model status (accuracy, AUC-ROC, Brier score)
- Prediction stream
- Trade history with P&L

Auto-refreshes every 15 seconds.

---

## Topic Bet

Bet on **any topic** you choose. Specify topics in a comma-separated list,
the bot searches Polymarket for matching markets and uses an LLM (Claude or
GPT) to analyze each one and decide whether to bet.

**Requires an LLM API key** (Anthropic or OpenAI).

### How it works

```
You set: TOPICS=crude oil,gold,trump,inflation
                    |
Bot searches Polymarket for matching markets
                    |
    "Will gold hit $3000?"  YES=$0.42
                    |
LLM analyzes: probability=78%, confidence=72%
                    |
    Edge = 78% - 42% = 36%  →  BET YES $10
                    |
    Dashboard at http://localhost:8091
```

### Quick start

```bash
# 1. Configure
cp .env.example .env
# Edit: add your ANTHROPIC_API_KEY and TOPICS

# 2. Run
topic-bot.exe -once      # one cycle
topic-bot.exe            # continuous loop
# Dashboard: http://localhost:8091
```

### Commands

```
topic-bot -once       One cycle (scan + analyze + trade)
topic-bot             Continuous loop (every 5 min)
topic-bot -scan       Scan markets only (no LLM, no trading)
topic-bot -status     Show status
```

### Configuration

```env
# What to bet on
TOPICS=crude oil,gold,bitcoin,trump,fed rate,inflation

# LLM (required)
LLM_PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-sonnet-4-20250514

# Or use OpenAI:
# LLM_PROVIDER=openai
# OPENAI_API_KEY=sk-...
# OPENAI_MODEL=gpt-4o-mini

# Trading
DRY_RUN=true               # Paper trading
BET_SIZE_USD=10             # $ per bet
MAX_DAILY_USD=200           # Daily cap
MIN_CONFIDENCE=60           # LLM must be 60%+ confident
LOOP_INTERVAL_SEC=300       # 5 min between cycles
DASHBOARD_PORT=8091
```

### Topic examples

```
crude oil, WTI, brent       Oil/energy markets
gold, silver                Precious metals
bitcoin, ethereum           Crypto
trump, biden, election      Politics
fed rate, inflation, CPI    Economics/macro
ukraine, china, taiwan      Geopolitics
super bowl, world cup       Sports
oscars, grammy              Entertainment
```

The bot automatically expands topics with search variations (e.g., "crude oil"
also searches "oil price", "WTI", "brent").

### How the LLM decides

For each market, the bot asks the LLM:

> Market: "Will gold hit $3000 by June 2026?"
> YES=$0.42, NO=$0.58
>
> Should I bet? Respond with recommendation, probability, confidence, reasoning.

The LLM only recommends betting when:
- Confidence >= 60%
- Edge >= 5% over market price

Most markets get **SKIP** — the LLM correctly identifies when it lacks
enough information.

### LLM cost

| Interval | Markets/cycle | Cost/cycle | Daily cost |
|---|---|---|---|
| 5 min | ~10 | ~$0.10 | ~$29 |
| 15 min | ~10 | ~$0.10 | ~$10 |
| 1 hour | ~10 | ~$0.10 | ~$2.40 |

Adjust `LOOP_INTERVAL_SEC` to control cost.

### Dashboard

Built-in at http://localhost:8091 with:
- Stats (topics, markets, analyses, trades, P&L, win rate)
- LLM analyses table (recommendation, probability, confidence, reasoning)
- Trades table (side, size, edge, status, P&L)
- Discovered markets by topic

Auto-refreshes every 15 seconds.

---

## Common Features

All three bots share:

- **DRY_RUN=true by default** — no real money until you flip the switch
- **Built-in web dashboard** — no separate frontend needed
- **SQLite local storage** — all data stays on your machine
- **No installation** — single binary (Go) or pip install (Python)
- **Auto-refresh dashboards** — 15-30 second update cycles

## System Requirements

- Windows 10/11 (64-bit) or Linux (64-bit)
- Internet connection
- ~50 MB disk space
- BTC Predictor: Python 3.10+ with scikit-learn, xgboost, pandas
- Topic Bet: LLM API key (Anthropic ~$5/mo or OpenAI)

## Disclaimer

These bots are for educational and research purposes. Prediction markets
carry financial risk — you can lose money. Start with DRY_RUN=true. You are
responsible for your own trading decisions and compliance with local regulations.
