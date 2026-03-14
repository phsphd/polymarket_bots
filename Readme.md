# Polymarket Copy Trader

Copy the top traders on Polymarket — automatically.

This bot scans the Polymarket leaderboard, identifies the highest-performing
traders, and mirrors their trades on your account. No AI, no complex strategies —
just follow the people who are actually winning.

---

## How It Works

```
Polymarket Leaderboard
        │
        ▼
  Top 100 traders ranked by profit
        │
        ▼
  Auto-select top 10 by all-time P&L
        │
        ▼
  Monitor their trades every 60 seconds
        │
        ▼
  Copy each trade at your configured size ($10 default)
        │
        ▼
  Track P&L as markets resolve
        │
        ▼
  Dashboard at http://localhost:8080
```

---

## Which Version Do I Need?

| | **Trader Edition** | **Viewer Edition** |
|---|---|---|
| **For** | Canadian & European users | US users |
| **Can trade?** | Yes — real orders on Polymarket | No — paper trading only |
| **Needs API key?** | Yes — your own Polymarket key | No |
| **Leaderboard** | Scans directly from Polymarket | Syncs from shared database |
| **Dashboard** | Yes | Yes |
| **File** | `polymarket-trader.exe` | `polymarket-us.exe` |

> **US users**: Polymarket Global is geo-restricted in the US. The Viewer
> Edition lets you see what top traders are doing and track simulated P&L,
> but cannot place real orders.

---

## Trader Edition (Canada / Europe)

Full copy trading bot. Scans the leaderboard, follows top traders, and places
real orders on your Polymarket account.

### Setup

1. Download `polymarket-trader.exe` and `.env`
2. Get your API key from [polymarket.com](https://polymarket.com) → Settings → API Keys
3. Edit `.env`:

```ini
POLY_API_KEY=your-api-key
POLY_API_SECRET=your-api-secret
POLY_PASSPHRASE=your-passphrase

DRY_RUN=true          # Paper trade first!
COPY_SIZE_USD=10      # $ per trade
COPY_MAX_DAILY_USD=500
```

4. Run:

```
polymarket-trader.exe -mode=trader
```

5. Open http://localhost:8080 for the dashboard
6. Watch paper results for a few days, then set `DRY_RUN=false` to go live

### Commands

```
polymarket-trader.exe -mode=trader                    Start bot + dashboard
polymarket-trader.exe -mode=trader -once              Run one cycle
polymarket-trader.exe -mode=trader -leaderboard-scan  Scan leaderboard only
polymarket-trader.exe -mode=trader -status            Show status
```

### Safety Features

- **DRY_RUN=true** by default — no real money until you flip the switch
- **Daily cap** — `COPY_MAX_DAILY_USD` stops the bot after spending $X/day
- **Per-trade cap** — `COPY_SIZE_USD` controls individual trade size
- **Your key stays local** — credentials never leave your machine
- **Full audit trail** — every trade logged in local database

---

## Viewer Edition (US)

Research tool + paper trading simulator. See what top Polymarket traders are
doing and track what your P&L would be if you followed them.

### Setup

1. Download `polymarket-us.exe` and `.env`
2. Run:

```
polymarket-us.exe -mode=us
```

3. Open http://localhost:8080 for the dashboard

That's it. No API key, no account needed.

### Commands

```
polymarket-us.exe -mode=us                    Start bot + dashboard
polymarket-us.exe -mode=us -once              Run one cycle
polymarket-us.exe -mode=us -leaderboard-scan  Pull latest leaderboard
polymarket-us.exe -mode=us -status            Show status
```

---

## Dashboard

Both editions include a built-in web dashboard at http://localhost:8080.

**What you see:**

- **Stats cards** — total trades, open positions, P&L, win rate
- **Cumulative P&L chart** — your equity curve over time
- **Daily P&L bars** — green and red bars showing daily performance
- **Top traders table** — leaderboard with PnL, volume, sortable by time period
- **Recent trades** — every copy trade with market, side, price, status, P&L

Auto-refreshes every 30 seconds. Dark theme.

---

## Configuration Reference

### All Editions

| Variable | Default | Description |
|---|---|---|
| `DRY_RUN` | `true` | Paper trading mode. Set `false` for real trades (Trader only) |
| `DB_PATH` | `copytrader.db` | Local SQLite database file |
| `COPY_SIZE_USD` | `10.0` | USD amount per copy trade |
| `COPY_MAX_DAILY_USD` | `500.0` | Maximum daily copy trading spend |
| `LOOP_INTERVAL_SEC` | `60` | Seconds between trade scan cycles |
| `RESOLVE_INTERVAL` | `10` | Check market resolutions every N cycles |
| `DASHBOARD_PORT` | `8080` | HTTP port for the dashboard |
| `LEADERBOARD_TOP_N` | `100` | Number of traders to track per ranking |
| `LEADERBOARD_AUTO_COPY_TOP_N` | `10` | Top N traders to auto-copy |
| `LOG_LEVEL` | `info` | Log verbosity: debug, info, warn, error |

### Trader Edition Only

| Variable | Default | Description |
|---|---|---|
| `POLY_API_KEY` | *(empty)* | Your Polymarket API key |
| `POLY_API_SECRET` | *(empty)* | Your Polymarket API secret |
| `POLY_PASSPHRASE` | *(empty)* | Your Polymarket API passphrase |

---

## FAQ

**Q: Is this safe to use?**
A: The bot starts in DRY_RUN mode — no real money is used until you explicitly
change `DRY_RUN=false`. Even then, daily spending is capped.

**Q: Where are my credentials stored?**
A: In the `.env` file on your local machine. They are never sent anywhere
except directly to Polymarket's API when placing orders.

**Q: What happens if my internet drops?**
A: The bot retries on the next cycle. No trades are placed during downtime.
Your local SQLite database preserves all state.

**Q: Can I change which traders I follow?**
A: The bot automatically selects the top 10 by all-time profit. Change
`LEADERBOARD_AUTO_COPY_TOP_N` to follow more or fewer traders.

**Q: How is P&L calculated?**
A: When a market resolves, the bot calculates profit/loss based on entry
price vs outcome. Daily and cumulative P&L are tracked automatically.

**Q: Why can't US users trade?**
A: Polymarket Global is not available to US residents due to CFTC regulations.
The Viewer Edition gives you the research and paper trading experience.

**Q: Do I need to keep the bot running 24/7?**
A: For best results, yes. The bot picks up leader trades as they happen.
If you stop and restart, it will catch up on any trades it missed.

---

## System Requirements

- Windows 10/11 (64-bit) or Linux (64-bit)
- Internet connection
- ~50MB disk space
- No other software required — single self-contained binary

---

## Files

Each edition is a single `.exe` (or Linux binary) plus a `.env` config file.
No installation needed — just download and run.

| File | Size | What it is |
|---|---|---|
| `polymarket-trader.exe` | 11 MB | Trader Edition (Windows) |
| `polymarket-trader-linux` | 11 MB | Trader Edition (Linux) |
| `polymarket-us.exe` | 11 MB | Viewer Edition (Windows) |
| `.env` | 1 KB | Configuration (edit this) |
