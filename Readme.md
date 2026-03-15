# Polymarket Trading Bot (Open-Claw Agent)

A multi-strategy trading bot for [Polymarket](https://polymarket.com) prediction markets. Trades across **crypto, politics, economics, sports, geopolitics, and news-driven markets** using 10 pluggable strategies, Kelly Criterion sizing, and optional LLM-powered analysis.

Runs in **dry-run mode by default** — no real money is risked until you explicitly configure credentials and set `DRY_RUN=false`.

---

## Table of Contents

- [Features](#features)
- [Setup](#setup)
- [Authentication](#authentication)
- [Configuration](#configuration)
- [Usage](#usage)
- [Strategies](#strategies)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Trade Log & Risk State](#trade-log--risk-state)
- [Adding a New Strategy](#adding-a-new-strategy)
- [Geographic Restrictions](#geographic-restrictions)
- [Disclaimer](#disclaimer)

---

## Features

- **10 trading strategies** covering crypto, politics, economics, and more
- **Kelly Criterion sizing** — position sizes optimized for bankroll growth
- **Market auto-classification** into 8 categories (crypto, politics, economics, sports, geopolitics, entertainment, science/tech, other)
- **LLM-powered analysis** (optional) — uses OpenAI or Anthropic for probability estimation and news impact assessment
- **Risk controls** — per-market caps, total exposure limits, daily loss circuit breaker, slippage protection
- **Persistent state** — risk exposure and daily PnL survive restarts
- **Trade logging** — every order is logged to `trades.jsonl` with full details
- **Dry-run by default** — safe to test without credentials
- **Multi-source pricing** — Binance US with CoinGecko fallback

---

## Setup

### Prerequisites

- Python 3.10 or higher
- pip (Python package manager)

### 1. Clone / download the project

```bash
cd /path/to/polymarket_bot
```

### 2. Create a virtual environment (recommended)

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# macOS/Linux
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

**Core dependencies:**

| Package | Purpose |
|---------|---------|
| `py-clob-client==0.34.6` | Official Polymarket CLOB client for order placement |
| `requests` | REST API calls (market discovery, pricing) |
| `python-dotenv` | Load `.env` file for configuration |
| `openai` | (Optional) LLM strategies via OpenAI API |
| `anthropic` | (Optional) LLM strategies via Anthropic API |

### 4. Create your `.env` file

Copy the example below and save as `.env` in the project root:

```env
# ==============================
# PLATFORM (choose one)
# ==============================
POLY_PLATFORM=us                      # "us" (CFTC-regulated) or "global" (non-US)

# ==============================
# US PLATFORM CREDENTIALS (if POLY_PLATFORM=us)
# ==============================
# Get these from polymarket.us/developer after KYC on the iOS app
# POLYMARKET_KEY_ID=your-key-id-uuid
# POLYMARKET_SECRET_KEY=your-base64-ed25519-secret-key

# ==============================
# GLOBAL PLATFORM CREDENTIALS (if POLY_PLATFORM=global)
# ==============================
# POLY_PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE
# Run "python -c 'from src.auth import main; main()'" to auto-derive:
# POLY_API_KEY=                         # Auto-filled
# POLY_API_SECRET=                      # Auto-filled
# POLY_API_PASSPHRASE=                  # Auto-filled

# ==============================
# RECOMMENDED
# ==============================
DRY_RUN=true                          # Start with dry-run! Set to false for live trading
POLY_ENV=mainnet                      # mainnet or testnet
TICKERS=BTC,ETH,SOL                   # Crypto tickers to track for spot price comparison
LOG_LEVEL=INFO                        # DEBUG, INFO, WARNING, ERROR

# ==============================
# RISK CONTROLS
# ==============================
MAX_POSITION_PER_MARKET_USD=200       # Max USD per single market
MAX_TOTAL_EXPOSURE_USD=1000           # Max total USD deployed across all markets
MIN_EDGE_PCT=0.015                    # Minimum edge (1.5%) for intra-market arb
DAILY_LOSS_LIMIT_USD=200              # Stop trading if daily losses exceed this
MAX_SLIPPAGE_BPS=50                   # Max slippage in basis points (0.50%)

# ==============================
# LOOP SETTINGS
# ==============================
LOOP_INTERVAL_SEC=10                  # Seconds between scan cycles

# ==============================
# PRICING
# ==============================
USE_BINANCE=true                      # Enable Binance spot price fetching
# BINANCE_BASE_URL=https://api.binance.us   # Default; change if geo-blocked

# ==============================
# OPTIONAL: AI-POWERED STRATEGIES
# ==============================
# Enables: LLM probability estimation, news sentiment, political/economic analysis
# AI_DECIDER=openai                   # Options: openai, anthropic, or leave empty
# OPENAI_API_KEY=sk-...               # Required if AI_DECIDER=openai
# OPENAI_MODEL=gpt-4o-mini            # Default model
# ANTHROPIC_API_KEY=sk-ant-...        # Required if AI_DECIDER=anthropic
# ANTHROPIC_MODEL=claude-3-5-sonnet-20241022

# ==============================
# OPTIONAL: EXTERNAL DATA SOURCES
# ==============================
# FRED_API_KEY=your_fred_key          # Free from https://fred.stlouisfed.org/docs/api/
                                      # Enables: economic indicator strategy (Fed, CPI, GDP)
# NEWSAPI_KEY=your_newsapi_key        # Free from https://newsapi.org
                                      # Enables: news sentiment strategy (100 req/day free)

# ==============================
# OPTIONAL: ADVANCED
# ==============================
# POLY_ORDERBOOK_URL=https://clob.polymarket.com/orderbook?market_id={id}
# POLY_MAX_ORDERBOOKS=50
# POLY_FUNDER_ADDRESS=                # Proxy/funder wallet address (advanced)
# POLY_SIGNATURE_TYPE=0               # 0=EOA, 1=POLY_PROXY, 2=POLY_GNOSIS_SAFE
# RISK_STATE_FILE=risk_state.json     # Persistent risk state file
# TRADE_LOG_FILE=trades.jsonl         # Trade log file
```

### 5. Verify installation

```bash
python -m src.main --once
```

You should see output like:
```
INFO  === Cycle 1 start ===
INFO  Discovered 1000 active markets
INFO  Markets by category: {'POLITICS': 312, 'SPORTS': 245, 'CRYPTO': 42, ...}
INFO  Built orderbooks for 1000 markets
INFO  Spot prices: {'BTC': '$67,200', 'ETH': '$1,950', 'SOL': '$83'}
INFO  === Cycle 1 done (3.5s) | signals: cross=1 longshot=175 | ...
```

---

## Authentication

The bot supports **two platforms** with different authentication systems:

### Polymarket US (for US users) — Ed25519 Auth

The US platform uses **Ed25519 signature-based authentication** via the official `polymarket-us` SDK.

**Setup Steps:**

1. **Download the Polymarket US iOS app** and complete KYC (ID + proof of address)
2. **Get API keys** at [polymarket.us/developer](https://polymarket.us/developer)
   - You'll receive a `key_id` (UUID) and `secret_key` (Base64-encoded Ed25519 private key)
   - The secret key is shown **only once** — save it securely
3. **Add to `.env`:**
   ```env
   POLY_PLATFORM=us
   POLYMARKET_KEY_ID=your-key-id-uuid
   POLYMARKET_SECRET_KEY=your-base64-ed25519-secret-key
   DRY_RUN=false
   ```
4. **Run the bot:**
   ```bash
   python -m src.main --once
   ```

> **Note:** Polymarket US is currently invite-only (as of March 2026). You need an invitation to complete KYC and generate API keys.

The SDK handles all request signing automatically — no manual crypto operations needed.

---

### Global Polymarket (for non-US users) — EIP-712 Auth

The global platform uses a **two-level authentication** system:

| Level | What | Purpose |
|-------|------|---------|
| **L1** | Private key (EIP-712 signing) | Proves wallet ownership on Polygon |
| **L2** | API credentials (key + secret + passphrase) | HMAC-signed requests for trading |

Both levels are required for placing live orders. The bot handles this automatically.

**Setup Steps:**

**Step 1: Get your private key**

**For non-US users (Global Polymarket):**

There are three wallet types:

| Signature Type | Value | Who | How to Get Private Key |
|----------------|-------|-----|----------------------|
| **EOA** | `0` | MetaMask / hardware wallet users | Export from MetaMask |
| **POLY_PROXY** | `1` | Magic Link / Google login users | Export from Polymarket |
| **GNOSIS_SAFE** | `2` | Gnosis Safe multisig users | Use the Safe owner key |

**Option A — MetaMask / EOA (Type 0):**
1. Open MetaMask > click the three dots on your account > Account Details
2. Click "Show Private Key", enter your password
3. Copy the key (starts with `0x`)
4. Your wallet pays gas in POL tokens directly

**Option B — Polymarket Email / Google login (Type 1):**
1. Go to [reveal.magic.link/polymarket](https://reveal.magic.link/polymarket)
2. Enter the same email or click the Google login button you used on Polymarket
3. Check your email for the Magic Link, click "Log in"
4. Your private key will be displayed — **copy and save it securely**
5. Your proxy wallet address (shown on polymarket.com when you click your profile) is your `POLY_FUNDER_ADDRESS`
6. Set `POLY_SIGNATURE_TYPE=1` in `.env`

**Option C — Gnosis Safe (Type 2):**
1. Use one of the Safe owner wallet keys
2. Set `POLY_FUNDER_ADDRESS` to your Safe address
3. Set `POLY_SIGNATURE_TYPE=2` in `.env`

Add your key to `.env`:
```env
POLY_PRIVATE_KEY=0xYOUR_PRIVATE_KEY_HERE

# Only needed for Magic Link (Type 1) or Gnosis Safe (Type 2):
# POLY_SIGNATURE_TYPE=1
# POLY_FUNDER_ADDRESS=0xYOUR_POLYMARKET_PROXY_ADDRESS
```

> **Note:** If you've never logged into polymarket.com, your proxy wallet hasn't been deployed yet. Log in at least once first to create it.

**Step 2: Derive API credentials**

Run the auth setup tool (one-time):
```bash
python -c "from src.auth import main; main()"
```

This will:
1. Connect to `https://clob.polymarket.com` (Polygon, chain ID 137)
2. Sign an EIP-712 message with your private key (L1 auth)
3. Call `create_or_derive_api_creds()` to get L2 credentials
4. Save `POLY_API_KEY`, `POLY_API_SECRET`, `POLY_API_PASSPHRASE` to your `.env`
5. Create a backup in `poly_api_creds.json`

Expected output:
```
Connecting to https://clob.polymarket.com (chain_id=137)...
Deriving API credentials (signing EIP-712 message)...
API Key:        xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
API Secret:     abcdefgh...wxyz
API Passphrase: abcd...wxyz

Credentials saved to .env
Backup saved to poly_api_creds.json (add to .gitignore!)

Verifying credentials...
Server time: 1741500000
API keys on account: 1
Authentication verified successfully!
```

**Step 3: Verify credentials**

```bash
python -c "from src.auth import main; main()" -- --check
```

Or just start the bot — it will verify on startup:
```bash
python -m src.main --once
```

### Auto-Derivation

If you set `POLY_PRIVATE_KEY` but skip Step 2, the bot will **automatically derive** API credentials on first run and save them to `.env`. You only need to run the auth setup manually if auto-derivation fails.

### Credential Reference

| Env Variable | Source | Description |
|--------------|--------|-------------|
| `POLY_PRIVATE_KEY` | Your wallet | Hex-encoded private key (with or without `0x` prefix) |
| `POLY_API_KEY` | Auto-derived | UUID — identifies your API session |
| `POLY_API_SECRET` | Auto-derived | Base64 string — used for HMAC-SHA256 request signing |
| `POLY_API_PASSPHRASE` | Auto-derived | Random string — additional auth factor |
| `POLY_FUNDER_ADDRESS` | Optional | Proxy/funder wallet for advanced setups |
| `POLY_SIGNATURE_TYPE` | Optional | `0` = EOA (default), `1` = POLY_PROXY, `2` = POLY_GNOSIS_SAFE |

### Security Notes

- Never commit `.env` or `poly_api_creds.json` to git (both are in `.gitignore`)
- API credentials can be re-derived anytime from your private key
- Each private key maps to one set of API credentials (idempotent)
- The bot only places orders you configure — it cannot withdraw funds

---

## Configuration

### Minimum Setup (Dry-Run)

No configuration needed. The bot runs in dry-run mode with sensible defaults:

```bash
python -m src.main --once
```

### Enabling Live Trading

**US Platform:**
1. Complete KYC on the Polymarket US iOS app
2. Generate API keys at polymarket.us/developer
3. Set `POLYMARKET_KEY_ID` and `POLYMARKET_SECRET_KEY` in `.env`
4. Set `POLY_PLATFORM=us` and `DRY_RUN=false`
5. Start with small limits (`MAX_POSITION_PER_MARKET_USD=50`, `MAX_TOTAL_EXPOSURE_USD=200`)

**Global Platform:**
1. Set `POLY_PLATFORM=global` and `POLY_PRIVATE_KEY` in `.env`
2. Run `python -c "from src.auth import main; main()"` to derive API credentials
3. Set `DRY_RUN=false` in `.env`
4. Ensure your wallet has USDC on Polygon for trading
5. Start with small limits (`MAX_POSITION_PER_MARKET_USD=50`, `MAX_TOTAL_EXPOSURE_USD=200`)

### Enabling AI Strategies (Anthropic API Key)

The bot uses an LLM to estimate probabilities, assess news impact, and rank trading signals. This enables 4 additional strategies: LLM probability estimation, news sentiment analysis, enhanced political analysis, and enhanced economic analysis.

**Setup (Anthropic / Claude):**

1. Go to [console.anthropic.com](https://console.anthropic.com/) and create an account
2. Add a payment method (Settings > Billing > Add Payment Method)
   - Minimum deposit is $5, which will last a long time at bot usage levels
3. Go to **Settings > API Keys** (or visit [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys))
4. Click **Create Key**, give it a name (e.g. "polymarket-bot")
5. Copy the key (starts with `sk-ant-api03-...`) — it's shown only once
6. Add to `.env`:
   ```env
   AI_DECIDER=anthropic
   ANTHROPIC_API_KEY=sk-ant-api03-YOUR_KEY_HERE
   ANTHROPIC_MODEL=claude-sonnet-4-20250514
   ```

**Estimated cost:** ~$0.01-0.05 per bot cycle (depending on number of markets scanned). Running 24/7 at 10-second intervals costs roughly $5-15/month.

**Alternative — OpenAI / GPT:**

1. Go to [platform.openai.com](https://platform.openai.com/) and create an account
2. Go to **API Keys** ([platform.openai.com/api-keys](https://platform.openai.com/api-keys))
3. Click **Create new secret key**, copy it
4. Add to `.env`:
   ```env
   AI_DECIDER=openai
   OPENAI_API_KEY=sk-YOUR_KEY_HERE
   OPENAI_MODEL=gpt-4o-mini
   ```

> **Tip:** `gpt-4o-mini` is cheapest (~$0.15/1M tokens). `claude-sonnet-4-20250514` is the best balance of cost and quality for probability estimation.

---

### Enabling News Sentiment (NewsAPI Key)

The news sentiment strategy fetches breaking headlines and uses the LLM to assess their impact on market probabilities. This catches price movements before the market reprices.

**Setup (free tier: 100 requests/day):**

1. Go to [newsapi.org](https://newsapi.org/) and click **Get API Key**
2. Create a free account (email + password, no credit card needed)
3. Your API key is shown on the dashboard after signup
4. Add to `.env`:
   ```env
   NEWSAPI_KEY=your_32_character_api_key_here
   ```

**Free tier limits:**
- 100 requests per day (the bot uses ~10 per cycle, so ~10 cycles/day with news)
- Headlines up to 24 hours old
- English language sources only

**What it does:**
- Extracts keywords from market questions (e.g. "Fed rate cut" → searches for "Fed", "rate", "cut")
- Fetches matching headlines from the last 12 hours
- LLM assesses whether news confirms/denies the market question
- Generates HIGH urgency signals when news shifts probability >5% from market price

> **Note:** NewsAPI requires `AI_DECIDER` to also be set (Anthropic or OpenAI) for the LLM impact assessment. Without LLM, it can still fetch headlines but can't generate trading signals.

---

### Enabling Economic Data (FRED API Key)

The economic strategy trades markets about Fed rate decisions, inflation (CPI), unemployment, GDP, and other macro indicators by comparing market prices to real government data.

**Setup (completely free, no limits):**

1. Go to [fred.stlouisfed.org](https://fred.stlouisfed.org/)
2. Click **My Account** (top right) and create a free account
3. After logging in, go to [fred.stlouisfed.org/docs/api/api_key.html](https://fred.stlouisfed.org/docs/api/api_key.html)
4. Click **Request API Key**
5. Fill in the short form (name, email, description — can be anything like "personal project")
6. Your API key is emailed to you immediately (32-character hex string)
7. Add to `.env`:
   ```env
   FRED_API_KEY=your_32_character_hex_key_here
   ```

**What data it provides:**
| Series | Indicator | Used For |
|--------|-----------|----------|
| `FEDFUNDS` | Federal Funds Rate | "Will the Fed cut rates?" markets |
| `CPIAUCSL` | Consumer Price Index | "Will inflation be above X%?" markets |
| `UNRATE` | Unemployment Rate | "Will unemployment fall below X%?" markets |
| `GDP` | Gross Domestic Product | "Will GDP grow?" / recession markets |
| `SP500` | S&P 500 Index | Stock market prediction markets |
| `DGS10` | 10-Year Treasury Yield | Bond/yield curve markets |

**How it works:**
- Identifies economic markets by keyword matching (e.g. "inflation", "Fed", "unemployment")
- Fetches the latest real data point from FRED
- Compares current value vs the market's threshold (e.g. CPI is 3.2%, market asks "above 3%?")
- Estimates probability based on distance from threshold + momentum
- Falls back to LLM estimation when FRED data isn't available

> **Tip:** FRED data updates on a schedule (monthly for CPI/unemployment, quarterly for GDP). The strategy is most valuable right after a data release, when Polymarket prices may lag behind the actual number.

---

### All Optional Keys — Quick Reference

| Key | Where to Get | Cost | What It Enables |
|-----|-------------|------|-----------------|
| `ANTHROPIC_API_KEY` | [console.anthropic.com](https://console.anthropic.com/settings/keys) | ~$5-15/mo | LLM strategies (probability, news, political, economic) |
| `NEWSAPI_KEY` | [newsapi.org](https://newsapi.org/) | Free (100 req/day) | Breaking news sentiment signals |
| `FRED_API_KEY` | [fred.stlouisfed.org](https://fred.stlouisfed.org/docs/api/api_key.html) | Free (unlimited) | Real economic data for macro markets |

---

## Usage

### One-Shot Scan

Run a single cycle (scan, analyze, execute) and exit:

```bash
python -m src.main --once
```

### Continuous Loop

Run continuously, scanning every `LOOP_INTERVAL_SEC` seconds:

```bash
python -m src.main --loop
```

Or simply (loop is the default):

```bash
python -m src.main
```

### Running in Background

```bash
# Linux/macOS
nohup python -m src.main --loop > bot.log 2>&1 &

# Windows (PowerShell)
Start-Process python -ArgumentList "-m src.main --loop" -RedirectStandardOutput bot.log -RedirectStandardError bot_err.log -NoNewWindow
```

### Checking Results

**Trade log** — every order is appended to `trades.jsonl`:
```bash
# View recent trades
python -c "
import json
with open('trades.jsonl') as f:
    for line in f:
        t = json.loads(line)
        print(f'{t[\"timestamp\"][:19]} {t[\"strategy\"]:18s} {t[\"action\"]:8s} \${t[\"size_usd\"]:>7.2f} edge={t[\"edge_pct\"]:.3f} {t[\"market_slug\"][:50]}')
"
```

**Risk state** — current exposure tracked in `risk_state.json`:
```bash
python -c "import json; print(json.dumps(json.load(open('risk_state.json')), indent=2))"
```

---

## Strategies

The bot runs **10 strategies** per cycle. Each strategy scans markets, generates signals with edge/confidence/urgency scores, and the agent ranks, deduplicates, and executes the best signals.

### 1. Intra-Market Arbitrage
**Category:** Crypto | **Risk:** Near-zero | **Urgency:** HIGH

Buys both YES and NO when their combined ask price is below $1.00 after fees. Guaranteed profit on resolution.

```
edge = 1.00 - (yes_ask + no_ask) - taker_fee_estimate
```

- Accounts for Polymarket's 2% taker fee on winnings
- Sizes limited by available liquidity on both sides
- Requires real orderbook depth (not just synthetic prices)

### 2. Endgame (Near-Expiry)
**Category:** Crypto | **Confidence:** High | **Urgency:** HIGH

Trades markets within 48 hours of resolution where one outcome is near-certain (bid > 90%) but the ask still offers edge.

- Time-weighted confidence (closer to resolution = more confident)
- Conservative sizing (30-50% of max position)
- Automatic fee deduction from edge calculation

### 3. Cross-Platform (Spot vs Prediction)
**Category:** Crypto | **Data:** Binance/CoinGecko | **Urgency:** MEDIUM-HIGH

Compares real-time spot prices from Binance/CoinGecko against Polymarket strike prices.

```
Example: BTC spot = $67,000, Market asks "Will BTC hit $65K?" at YES=$0.55
→ BUY YES (spot already above strike, market underpricing)
```

- Parses "hit $X", "above $X", "reach $X", ">$X" patterns (with K/M/B suffixes)
- Generates both BUY_YES and BUY_NO signals
- Recognizes ticker symbols AND full names (BTC/Bitcoin, ETH/Ethereum, etc.)

### 4. Longshot Bias Exploitation
**Category:** All markets | **Risk:** Statistical | **Urgency:** LOW

Exploits the empirically validated tendency for low-probability contracts to be overpriced and high-probability contracts to be underpriced.

```
Contracts at $0.05-$0.20 → overpriced (sell/buy NO)
Contracts at $0.80-$0.95 → underpriced (buy YES)
```

- Applies calibration adjustments derived from academic research
- Kelly-sized positions for optimal bankroll growth
- Diversifies across many markets (typically finds 100+ signals per cycle)

### 5. Flash Crash Mean-Reversion
**Category:** All markets | **Risk:** Moderate | **Urgency:** HIGH

Detects sudden probability drops (>15% within 5 minutes) and buys the crashed side, expecting mean reversion.

- Maintains rolling price history across cycles
- Take-profit: +$0.10 | Stop-loss: -$0.05
- Conservative sizing for larger crashes

### 6. Correlated Market Arbitrage
**Category:** All markets | **Risk:** Low | **Urgency:** MEDIUM

Finds logical pricing violations between related markets.

```
Example: "Trump wins 2028" = 35%  →  "Republican wins 2028" must be ≥ 35%
If Republican market is at 30%, buy YES (5% edge, near-riskless)
```

- Detects price-target implications (BTC $200K → BTC $150K)
- Detects political implications (candidate → party)
- Uses text similarity matching to find related market pairs

### 7. LLM Probability Estimation
**Category:** Crypto | **Requires:** `AI_DECIDER` configured | **Urgency:** MEDIUM

Uses LLM to batch-estimate true probabilities for crypto markets, then trades when the model disagrees with market prices by >5%.

- Fractional Kelly sizing (0.25x) based on LLM probability vs market price
- Expected value calculation per dollar invested
- Processes up to 15 markets per LLM batch call

### 8. Political Polling
**Category:** Politics | **Data:** 538 + LLM | **Urgency:** LOW

Compares polling data and model forecasts to Polymarket prices for election markets.

- Parses election type (presidential, senate, governor, primary)
- Extracts candidate names, parties, states
- Attempts FiveThirtyEight forecast data
- Falls back to LLM estimation with election context
- Conservative Kelly sizing (0.20 fraction)

### 9. Economic Indicators
**Category:** Economics | **Data:** FRED + LLM | **Urgency:** MEDIUM

Trades markets around Fed decisions, inflation, GDP, unemployment using real economic data.

- Parses: "Fed cut rates", "CPI above 3%", "unemployment below 4%", "basis points"
- Fetches real-time data from FRED API (Federal Reserve Economic Data)
- Estimates probabilities from current values vs thresholds
- Falls back to LLM analysis when FRED data is unavailable

### 10. News Sentiment
**Category:** All (0.20-0.80 priced) | **Requires:** `NEWSAPI_KEY` or `AI_DECIDER` | **Urgency:** HIGH

Detects breaking news that affects market probabilities before the market has repriced.

- Fetches recent headlines from NewsAPI.org (12-hour window)
- Uses LLM to assess news impact: positive, negative, or neutral
- Only trades on non-neutral impact with >5% edge
- Rate-limited to 10 markets per cycle (API budget management)

---

## Architecture

```
                          ┌─────────────────┐
                          │    main.py       │
                          │  CLI entry point │
                          └────────┬────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │     open_claw_agent.py       │
                    │  Orchestrator: scan → rank   │
                    │  → execute → log             │
                    └──┬───┬───┬───┬───┬───┬──────┘
                       │   │   │   │   │   │
        ┌──────────────┘   │   │   │   │   └──────────────┐
        ▼                  ▼   ▼   ▼   ▼                  ▼
┌──────────────┐    ┌─────────────────────┐    ┌─────────────────┐
│  strategies/ │    │  polymarket_client   │    │   risk.py       │
│  10 modules  │    │  (Global: CLOB v2)   │    │   Exposure caps │
│  (see below) │    │  polymarket_us_client│    │   Daily PnL     │
└──────────────┘    │  (US: Ed25519 SDK)   │    │   Persistence   │
                    └─────────────────────┘    └─────────────────┘
        │
        ├── market_classifier.py     auto-categorize markets
        ├── intra_market.py          crypto arb
        ├── endgame.py               near-expiry
        ├── cross_platform.py        spot vs prediction
        ├── longshot_bias.py         favorite/longshot bias
        ├── mean_reversion.py        flash crash
        ├── correlated_arb.py        logical pricing
        ├── llm_estimate.py          LLM probability
        ├── political.py             polling + elections
        ├── economic.py              FRED + indicators
        └── news_sentiment.py        breaking news

Supporting modules:
        ├── sizing/kelly.py          Kelly Criterion math
        ├── pricing/binance.py       Spot prices (Binance + CoinGecko)
        └── ai/decision.py           LLM ranking + probability estimation
```

### Cycle Flow

1. **Discover** — Fetch active markets from Polymarket Gamma API
2. **Classify** — Auto-categorize into 8 categories
3. **Build books** — Construct orderbooks (real or synthetic from prices)
4. **Scan** — Run all 10 strategies, each generates typed `Signal` objects
5. **Aggregate** — Combine signals, deduplicate by (market_id, action)
6. **Rank** — AI-assisted ranking + size multiplier (or rule-based fallback)
7. **Execute** — Top 5 signals per cycle, risk-checked, with slippage protection
8. **Log** — Append trades to `trades.jsonl`, update `risk_state.json`
9. **Sleep** — Wait `LOOP_INTERVAL_SEC` seconds, repeat

---

## Project Structure

```
polymarket_bot/
├── README.md                          # This file
├── PRD.md                             # Product requirements document
├── STRATEGY_RESEARCH.md               # Research notes
├── requirements.txt                   # Python dependencies
├── .env                               # Your configuration (not in git)
├── .gitignore                         # Excludes secrets + runtime files
│
├── src/                               # Python bot (10 strategies + copy trading)
│   ├── main.py                        # CLI: --once, --loop, --sim-*, --leaderboard-*
│   ├── config.py                      # Environment-based configuration
│   ├── open_claw_agent.py             # Agent orchestrator (10 strategies)
│   ├── polymarket_client.py           # Global platform API client
│   ├── polymarket_us_client.py        # US platform API client
│   ├── risk.py                        # Risk manager with persistence
│   ├── leaderboard.py                 # Leaderboard scanner + PostgreSQL
│   ├── copy_db.py                     # Copy trading PostgreSQL persistence
│   ├── auth.py                        # CLOB authentication setup
│   ├── ai/decision.py                 # LLM ranking + probability estimation
│   ├── sizing/kelly.py                # Kelly Criterion position sizing
│   ├── pricing/binance.py             # Spot prices (Binance + CoinGecko)
│   ├── sim/                           # Simulation engine (PostgreSQL-backed)
│   │   ├── db.py, exchange.py, portfolio.py, resolver.py, analytics.py, cli.py
│   └── strategies/                    # 10+1 trading strategies
│       ├── market_classifier.py       # Auto-categorize markets (8 categories)
│       ├── copy_trading.py            # Follow leader wallet trades
│       ├── intra_market.py            # Crypto YES+NO arbitrage
│       ├── endgame.py, cross_platform.py, longshot_bias.py, mean_reversion.py
│       ├── correlated_arb.py, llm_estimate.py, political.py, economic.py
│       └── news_sentiment.py          # Breaking news impact
│
├── dashboard/                         # Next.js 15 web dashboard (deployed on Vercel)
│   ├── src/app/page.tsx               # Portfolio page
│   ├── src/app/leaders/page.tsx       # Top traders leaderboard
│   ├── src/app/copies/page.tsx        # Copy trade history
│   ├── src/lib/queries.ts             # All PostgreSQL queries
│   └── src/components/                # Charts, Nav, LeaderTabs
│
├── scripts/                           # Operations & monitoring
│   ├── bot_monitor.py                 # Health monitor — emails on crash (every 5 min)
│   ├── run_monitor.bat                # Task Scheduler wrapper for monitor
│   ├── pg_aggregate_cleanup.py        # Daily aggregation + cleanup (3 AM)
│   └── run_cleanup.bat                # Task Scheduler wrapper for cleanup
│
└── polymarket_GO_bot/                 # Go copy trader (for distribution)
    ├── main.go                        # 3-mode CLI (ca/trader/us)
    ├── copier/copier.go               # Trade copier + consensus + price tracker
    ├── api/polymarket.go              # CLOB + Data API clients
    ├── db/sqlite.go                   # Local SQLite (unrealized P&L, resolution)
    ├── db/pgstore.go                  # Aiven PostgreSQL sync
    ├── dashboard/server.go            # Local dashboard on :8080
    ├── ARCHITECTURE.md                # Design decisions
    ├── ROADMAP.md                     # Feature roadmap
    └── dist/                          # Distribution packages (3 editions)
        ├── polymarket_us.zip          # US viewer edition
        ├── polymarket_trader.zip      # CA/EU trader edition
        └── polymarket_ca.zip          # CA scanner edition
```

---

## Trade Log & Risk State

### Trade Log (`trades.jsonl`)

Every order placed (including dry-run) is appended as a JSON line:

```json
{
  "timestamp": "2026-03-09T01:00:41.081836+00:00",
  "strategy": "cross_platform",
  "market_id": "540844",
  "market_slug": "will-bitcoin-hit-1m-before-gta-vi",
  "action": "BUY_NO",
  "size_usd": 140.0,
  "edge_pct": 0.10,
  "confidence": 0.85,
  "orders": [{"market_id": "540844", "token_id": "918631...", "side": "BUY", "price": 0.5131, "size": 274.24, "outcome": "NO"}],
  "responses": [{"status": "dry_run", "order": {...}}],
  "dry_run": true
}
```

### Risk State (`risk_state.json`)

Persists across restarts. Resets daily PnL automatically at midnight:

```json
{
  "total_exposure_usd": 340.0,
  "per_market_exposure_usd": {"540844": 140.0, "553858": 50.0, ...},
  "realized_pnl_usd": 0.0,
  "last_reset_date": "2026-03-09"
}
```

To reset risk state manually:
```bash
rm risk_state.json
```

---

## Adding a New Strategy

1. Create `src/strategies/your_strategy.py`:

```python
from dataclasses import dataclass
from typing import Any, Dict, List

@dataclass
class Signal:
    strategy: str
    market_id: str
    market_slug: str
    side: str          # "BUY_YES" or "BUY_NO"
    size_usd: float
    expected_edge_pct: float
    confidence: float  # 0.0 - 1.0
    urgency: str       # "LOW", "MEDIUM", "HIGH"
    metrics: dict | None = None

def scan_your_signals(
    markets: List[Dict[str, Any]],
    books_by_id: Dict[str, Dict[str, Any]],
    max_position_usd: float,
) -> List[Signal]:
    signals = []
    # Your logic here...
    return signals
```

2. Import and call it in `src/open_claw_agent.py`:

```python
from .strategies.your_strategy import scan_your_signals

# Inside cycle_once(), add:
your_sigs = scan_your_signals(markets, books, self.cfg.risk.max_position_per_market_usd)
all_signals.append(("your_strategy", your_sigs))
```

That's it. The agent handles deduplication, ranking, risk checks, execution, and logging.

---

## Geographic Restrictions

Polymarket has **two separate platforms** with different access rules:

### Global Polymarket (polymarket.com)

The global platform **blocks orders from the US** and 32 other countries. The CLOB API at `clob.polymarket.com` enforces this via IP geolocation. Blocked countries include: US, France, Germany, Italy, Australia, Singapore, Russia, and others.

This bot is built for the **global CLOB API** (`clob.polymarket.com`, Polygon chain ID 137).

### Polymarket US (CFTC-Regulated)

As of December 2025, Polymarket launched a **separate CFTC-regulated US platform**:

- Requires KYC via the official iOS app (ID verification + proof of address)
- Trading goes through registered brokerages / FCMs (not direct crypto wallets)
- Has its own API with 23 REST endpoints and 2 WebSocket endpoints
- Currently **invite-only** (as of March 2026)
- Uses Ed25519 authentication keys (different from the global platform's EIP-712 signing)

**If you are a US user**, this bot does not currently support the Polymarket US API. You would need to:
1. Get access to Polymarket US (invite-only)
2. Complete KYC via the iOS app
3. Generate Ed25519 API keys through their developer portal

US API support may be added in a future update.

### Check Your Eligibility

```bash
curl -s https://polymarket.com/api/geoblock | python -m json.tool
```

Sources:
- [Polymarket Geographic Restrictions](https://docs.polymarket.com/api-reference/geoblock)
- [Polymarket CFTC Approval (CoinDesk)](https://www.coindesk.com/business/2025/11/25/polymarket-secures-cftc-approval-for-regulated-u-s-return/)
- [Polymarket US API Guide](https://tradingvps.io/polymarket-us-guide/)

---

## Simulation & Copy Trading

### Simulation Mode

Paper-trade with real market data, no real money. Uses PostgreSQL to track orders, positions, and P&L.

```bash
# Enable in .env
SIM_MODE=true
SIM_STARTING_BALANCE=10000
DATABASE_URL=postgres://user:pass@host:port/db?sslmode=require

# Run
python -m src.main

# Check status
python -m src.main --sim-status
python -m src.main --sim-report
python -m src.main --sim-reset
python -m src.main --sim-resolve
```

### Leaderboard Scanner

Scans Polymarket's top 100 traders daily across 5 rankings (PnL all-time/month/week, Volume all-time/month). Stores in PostgreSQL and optionally feeds the copy trading strategy.

```bash
# Manual scan
python -m src.main --leaderboard-scan

# Check current top traders
python -m src.main --leaderboard-status
```

API: `GET https://data-api.polymarket.com/v1/leaderboard`
- `orderBy`: PNL or VOL
- `timePeriod`: DAY, WEEK, MONTH, ALL
- `limit`: 1-50, `offset`: 0-1000

**Note:** This API is geo-blocked from US IPs via Cloudflare, but works intermittently on IPv4. The scanner retries automatically and typically captures 100+ traders per scan even from the US.

Configuration:
```env
LEADERBOARD_ENABLED=true
LEADERBOARD_TOP_N=100
LEADERBOARD_AUTO_COPY_TOP_N=10
LEADERBOARD_AUTO_COPY=true       # Auto-add top traders to copy trading
```

### Copy Trading (Follow Top Traders)

Automatically follows trades from top leaderboard traders. No LLM required.

```env
STRATEGY_COPY_TRADING=true
COPY_SIZE_USD=10                 # USD per copy trade
COPY_MAX_DAILY_USD=500           # Daily spending cap
```

Flow:
1. Leaderboard scanner identifies top 10 traders by PnL
2. Every cycle, fetches their recent trades from `data-api.polymarket.com/trades`
3. Deduplicates against stored history
4. Records copy trades (DRY_RUN or live)
5. Periodically checks market resolutions and calculates P&L

### Dashboard

Next.js web dashboard showing portfolio, leaderboard, and copy trade data. Deployed on Vercel, reads from the same PostgreSQL.

```bash
cd dashboard && npm install && npm run dev
# http://localhost:5000
```

**Pages:**
- `/` — Portfolio: equity curve, daily P&L, strategy performance, positions, trades
- `/leaders` — Top Traders: leaderboard stats, PnL charts, tabbed trader table
- `/copies` — Copy Trades: stats, trade history with leader, side, status, P&L

---

## Operator Guide (Running the Data Pipeline)

This section is for running the Python bot as a data pipeline that serves the Go copy trader users and the Vercel dashboard.

### What You Run

```
Your machine (US)
  ├── Python bot (this repo)
  │     ├── Leaderboard scanner → writes to PostgreSQL
  │     ├── Copy trading → writes to PostgreSQL
  │     └── Sim engine → writes to PostgreSQL
  │
  └── PostgreSQL (Aiven cloud)
        ├── leaderboard_top_traders → Dashboard /leaders page
        ├── copy_trades → Dashboard /copies page
        ├── go_top_traders → Go bot US edition syncs from here
        └── sim_* tables → Dashboard / page
```

### Quick Start

```bash
# 1. Activate venv
venv\Scripts\activate        # Windows
source venv/bin/activate     # Linux/macOS

# 2. Verify .env has these set:
#    DATABASE_URL=postgres://...
#    SIM_MODE=true
#    LEADERBOARD_ENABLED=true
#    LEADERBOARD_AUTO_COPY=true
#    STRATEGY_COPY_TRADING=true
#    DRY_RUN=true
#    AI_DECIDER=               (empty — copy trading doesn't need LLM)
#    LOOP_INTERVAL_SEC=60

# 3. Initial leaderboard scan
python -m src.main --leaderboard-scan

# 4. Start continuous loop
python -m src.main
```

### What Happens Each Cycle

1. **Discover markets** from Gamma API (intermittent from US, bot handles failures)
2. **Daily leaderboard scan** if not yet scanned today (writes `leaderboard_top_traders`)
3. **Copy trading scan** — checks top 10 traders for new trades
4. **Record copies** as DRY_RUN in `copy_trades` table
5. **Resolve markets** periodically — calculates P&L
6. **Snapshot portfolio** — equity curve data point
7. **Auto-sync to `go_top_traders`** if `LEADERBOARD_AUTO_COPY=true`

### Keeping Go Bot Users Fed

The Go bot's US edition pulls from `go_top_traders` in PostgreSQL. To keep both tables in sync, the Python leaderboard scanner writes to `leaderboard_top_traders`, which is also the source for the Vercel dashboard.

If you also need `go_top_traders` populated (for Go bot users), run this one-time sync:

```bash
python -c "
from dotenv import load_dotenv; load_dotenv()
import psycopg2, os
conn = psycopg2.connect(os.getenv('DATABASE_URL'))
cur = conn.cursor()
cur.execute('''INSERT INTO go_top_traders (proxy_wallet, user_name, x_username, verified_badge, profile_image,
    rank_pnl_all, rank_pnl_month, rank_pnl_week, rank_vol_all, rank_vol_month,
    pnl_all, pnl_month, pnl_week, vol_all, vol_month, is_auto_copy, first_seen, last_updated)
SELECT proxy_wallet, user_name, x_username, verified_badge, profile_image,
    rank_pnl_all, rank_pnl_month, rank_pnl_week, rank_vol_all, rank_vol_month,
    pnl_all, pnl_month, pnl_week, vol_all, vol_month, is_auto_copy, first_seen, last_updated
FROM leaderboard_top_traders
ON CONFLICT (proxy_wallet) DO UPDATE SET
    user_name=EXCLUDED.user_name, pnl_all=EXCLUDED.pnl_all, vol_all=EXCLUDED.vol_all,
    rank_pnl_all=EXCLUDED.rank_pnl_all, is_auto_copy=EXCLUDED.is_auto_copy,
    last_updated=EXCLUDED.last_updated''')
conn.commit()
cur.execute('SELECT COUNT(*) FROM go_top_traders')
print(f'Synced {cur.fetchone()[0]} traders to go_top_traders')
conn.close()
"
```

Or add this to a daily cron job / scheduled task.

### Running as a Background Service (Windows)

```powershell
# Start in background
Start-Process python -ArgumentList "-m src.main" -WorkingDirectory "C:\Paul\IT\IT\polymarket_bot" -RedirectStandardOutput bot.log -RedirectStandardError bot_err.log -NoNewWindow

# Or use Task Scheduler for auto-start on boot
```

### Running as a Background Service (Linux)

Create `/etc/systemd/system/polymarket-bot.service`:
```ini
[Unit]
Description=Polymarket Copy Trading Bot
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/polymarket_bot
ExecStart=/opt/polymarket_bot/venv/bin/python -m src.main
Restart=always
RestartSec=30
EnvironmentFile=/opt/polymarket_bot/.env

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable polymarket-bot
sudo systemctl start polymarket-bot
sudo journalctl -u polymarket-bot -f
```

---

## Go Copy Trader (Distribution)

A standalone Go binary for distributing copy trading to end users. See `polymarket_GO_bot/` for full documentation.

| Edition | Users | What It Does |
|---|---|---|
| **Trader** | CA/EU users | Full copy trading with their own Polymarket API key |
| **Viewer** | US users | Dashboard + paper P&L tracking (no trading) |
| **CA Scanner** | VPS operator | Scans leaderboard from non-US IP, pushes to PostgreSQL |

Both editions pull leaderboard data that your Python bot pushes to PostgreSQL.

### Key Features (v1.1 — March 2026)

- **Live Unrealized P&L** — fetches current token prices from CLOB API every cycle, computes per-trade P&L in real time. Dashboard shows Entry/Now price columns and color-coded gains/losses.
- **Market Horizon Filter** (`MAX_MARKET_DAYS=7`) — skips markets resolving more than 7 days out, keeping capital in short-term markets with faster resolution.
- **Consensus Mode** (`CONSENSUS_PERCENT=30`, opt-in) — only copies trades when 30%+ of followed leaders agree on the same market direction. Two-pass scan: collect all signals, then filter by agreement.
- **Resolution Tracking** — detects closed markets via CLOB API, calculates realized P&L per trade, updates daily P&L and cumulative stats.

### Distribution Packages

Three zip files in `polymarket_GO_bot/dist/`:

| Zip | Size | Contents |
|-----|------|----------|
| `polymarket_us.zip` | ~9MB | exe, .env, README, cleanup scripts |
| `polymarket_trader.zip` | ~9MB | exe, .env (with API key fields), README, cleanup scripts |
| `polymarket_ca.zip` | ~9MB | exe, .env (scanner config), README, cleanup scripts |

See `polymarket_GO_bot/ARCHITECTURE.md` and `polymarket_GO_bot/ROADMAP.md` for details.

---

## BTC Bot

ML-powered Bitcoin prediction bot that trades Polymarket's BTC Up/Down markets.

- **Model:** Trains on Binance BTCUSDT candle data (15m and 1h timeframes)
- **Edge detection:** Compares model probability vs Polymarket market price, trades when edge > 5%
- **Kelly sizing:** Quarter-Kelly position sizing for optimal bankroll growth
- **Auto-retrain:** Daily model retraining at 12:00 UTC with 90-day backfill

```bash
cd C:\paul\bots\polymarket_bot && python -m btc_bot --loop
```

**Note:** Requires active BTC Up/Down markets on Polymarket. When no matching markets exist, the bot continues making predictions but cannot place trades.

---

## Stock Bot

Trades stock/index-related prediction markets using real-time price data.

```bash
cd C:\paul\bots\polymarket_bot && python -m stock_bot --loop
```

---

## Bot Health Monitor

Automated monitoring script that checks if all bots are running and sends email alerts on crash.

**Script:** `scripts/bot_monitor.py`
**Schedule:** Windows Task Scheduler "BotMonitor" — every 5 minutes
**Alerts:** Email to `phs.phd@gmail.com` via Gmail SMTP

### Monitored Bots

| Bot | Process Signature |
|-----|-------------------|
| BTC Bot | `btc_bot --loop` |
| Stock Bot | `stock_bot --loop` |
| Copy Bot (Go) | `polymarket_GO_bot.exe` |

### Features

- 30-minute cooldown between alerts per bot (no spam)
- Logs to `scripts/monitor.log`
- Includes restart commands in alert email
- `--status` flag for manual health check

```bash
# Manual status check
python scripts/bot_monitor.py --status

# Run with alerting
python scripts/bot_monitor.py
```

---

## Database Maintenance

Daily cleanup script aggregates stats and removes old resolved data.

**Script:** `scripts/pg_aggregate_cleanup.py`
**Schedule:** Windows Task Scheduler "PolymarketCleanup" — daily at 3 AM

Creates 6 aggregation tables (`agg_*`) in PostgreSQL, then deletes resolved data older than 7 days from both PG and local SQLite databases.

```bash
python scripts/pg_aggregate_cleanup.py --dry-run   # Preview changes
python scripts/pg_aggregate_cleanup.py              # Run for real
```

---

## Scheduled Tasks Summary

| Task | Schedule | Script |
|------|----------|--------|
| BotMonitor | Every 5 minutes | `scripts/run_monitor.bat` |
| PolymarketCleanup | Daily 3 AM | `scripts/run_cleanup.bat` |

---

## Disclaimer

This software is provided as-is with no warranty. Prediction markets carry financial risk and you can lose money. You are solely responsible for your API keys, private keys, trading decisions, and compliance with local regulations. This is not financial advice.
