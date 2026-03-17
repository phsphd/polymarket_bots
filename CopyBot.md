# Polymarket Copy Trader

Copy the top traders on Polymarket — automatically.

This bot scans the Polymarket leaderboard, identifies today's highest-performing
traders, and mirrors their trades on your account. No AI, no complex strategies —
just follow the people who are actually winning right now.

---

## How It Works

```
Polymarket Daily Leaderboard
        │
        ▼
  Top 10 traders by today's profit
        │
        ▼
  Monitor their trades every 60 seconds
        │
        ▼
  Copy only fresh trades (< 5 min old)
        │
        ▼
  Track P&L as markets resolve
        │
        ▼
  Dashboard at http://localhost:8080
```

### Crowd Consensus (NEW)

Instead of copying individual traders, crowd consensus looks at what **all**
top 100 leaders are doing and only follows when a majority agree:

```
Fetch top 100 daily PnL leaders (refreshed hourly)
        │
        ▼
  Collect their trades from last 2 hours
        │
        ▼
  Group by market: "Who's betting what?"
        │
        ▼
  If 55%+ agree on same direction AND 5+ leaders → FOLLOW
        │
        ▼
  Track P&L as markets resolve
```

**Example**: 15 leaders bet on "NBA: Lakers vs Celtics total over 220.5".
10 bet YES, 5 bet NO → 67% consensus → bot follows YES.

---

## Which Version Do I Need?

| | **Trader Edition** | **Viewer Edition** |
|---|---|---|
| **For** | Canadian & European users | US users |
| **Can trade?** | Yes — real orders on Polymarket | No — paper trading only |
| **Needs API key?** | Yes — your own Polymarket key | No |
| **Leaderboard** | Scans directly from Polymarket | Syncs from shared database |
| **Crowd Consensus** | Yes | Yes |
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
COPY_MAX_DAILY_USD=2000
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
- **5-minute freshness filter** — only copies trades placed in the last 5 minutes
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

## Crowd Consensus Mode

Both editions support crowd consensus. Enable in `.env`:

```ini
CROWD_CONSENSUS=true
CROWD_TOP_N=100         # Scan top 100 daily PnL leaders
CROWD_WINDOW_MIN=120    # Look at trades from last 2 hours
CROWD_MIN_BETTORS=5     # Need at least 5 leaders on a market
CROWD_THRESHOLD=55      # 55% must agree on same direction
```

**How it works:**
1. Fetches the top 100 traders by today's profit (refreshed hourly)
2. Collects all their trades from the last 2 hours
3. Groups by market — counts unique leaders on each side
4. Only copies when threshold is met (55% agreement with 5+ leaders)
5. Won't re-enter the same market twice in one day

Crowd consensus runs **alongside** individual copy trading. Both can be active
at the same time, or you can use either one alone.

---

## Choose Your Own Leaders

By default, the bot automatically follows the top 10 traders by daily profit.
If you want to pick your own leaders instead, set `USE_MY_LEADERS=true` and
list their wallet addresses:

```ini
USE_MY_LEADERS=true
MY_LEADERS=0xabc123...,0xdef456...,0x789abc...
```

### How to find wallet addresses

1. Go to [polymarket.com/leaderboard](https://polymarket.com/leaderboard)
2. Click on a trader you want to follow
3. Copy their wallet address from the profile URL or page
4. Add it to `MY_LEADERS` (comma-separated, no spaces around commas)

---

## Dashboard

Both editions include a built-in web dashboard at http://localhost:8080.

**What you see:**

- **Stats cards** — total trades, open positions, P&L, win rate
- **Cumulative P&L chart** — your equity curve over time
- **Daily P&L bars** — green and red bars showing daily performance
- **Active leaders table** — today's top traders by weekly PnL, with auto-copy status
- **Recent trades** — every copy trade with market, side, price, status, P&L

All times shown in Eastern Time (EST). Auto-refreshes every 30 seconds. Dark theme.

---

## Configuration Reference

### All Editions

| Variable | Default | Description |
|---|---|---|
| `DRY_RUN` | `true` | Paper trading mode. Set `false` for real trades (Trader only) |
| `DB_PATH` | `copytrader.db` | Local SQLite database file |
| `COPY_SIZE_USD` | `10.0` | USD amount per copy trade |
| `COPY_MAX_DAILY_USD` | `2000.0` | Maximum daily copy trading spend |
| `LOOP_INTERVAL_SEC` | `60` | Seconds between trade scan cycles |
| `RESOLVE_INTERVAL` | `10` | Check market resolutions every N cycles |
| `DASHBOARD_PORT` | `8080` | HTTP port for the dashboard |
| `MAX_MARKET_DAYS` | `7` | Skip markets resolving beyond N days (0 = no limit) |
| `LEADERBOARD_TOP_N` | `100` | Number of traders to track per ranking |
| `LEADERBOARD_AUTO_COPY_TOP_N` | `10` | Top N daily traders to auto-copy |
| `LOG_LEVEL` | `info` | Log verbosity: debug, info, warn, error |
| `USE_MY_LEADERS` | `false` | Use custom leader list instead of auto-discovery |
| `MY_LEADERS` | *(empty)* | Comma-separated wallet addresses |

### Crowd Consensus

| Variable | Default | Description |
|---|---|---|
| `CROWD_CONSENSUS` | `false` | Enable crowd consensus mode |
| `CROWD_TOP_N` | `100` | How many daily PnL leaders to scan |
| `CROWD_WINDOW_MIN` | `120` | Lookback window in minutes |
| `CROWD_MIN_BETTORS` | `5` | Minimum leaders on a market before consensus applies |
| `CROWD_THRESHOLD` | `55` | Agreement % required to follow |

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

**Q: Can I use both individual copy and crowd consensus?**
A: Yes! They run independently. Individual copy follows the 10 auto-copy
leaders. Crowd consensus scans all 100 and votes. Both can place trades.

**Q: How often does the crowd leader list update?**
A: Every hour. The bot fetches the current top 100 by daily PnL from
Polymarket's leaderboard API, so it always tracks today's best performers.

**Q: Why 55% threshold?**
A: Higher than coin-flip (50%) ensures meaningful agreement. You can adjust
`CROWD_THRESHOLD` — set to 70 for stronger consensus, 51 for simple majority.

**Q: How is P&L calculated?**
A: When a market resolves, the bot calculates profit/loss based on entry
price vs outcome. Daily and cumulative P&L are tracked automatically.

**Q: Why can't US users trade?**
A: Polymarket Global is not available to US residents due to CFTC regulations.
The Viewer Edition gives you the research and paper trading experience.

---

## Database Maintenance

The bot stores trades in a local SQLite database (`copytrader.db`). Over time
this grows. A cleanup script is included to aggregate statistics and remove
old resolved data (>7 days).

### Setup (Windows Task Scheduler)

```
schtasks /Create /TN "PolymarketCleanup" /TR "C:\path\to\scripts\run_cleanup.bat" /SC DAILY /ST 03:00 /RL HIGHEST
```

### Manual Run

```bash
# Dry run (aggregate stats only, no delete):
python scripts/pg_aggregate_cleanup.py --dry-run

# Full run (aggregate + delete old data):
python scripts/pg_aggregate_cleanup.py
```

Requires Python 3 and `psycopg2` (`pip install psycopg2-binary`).

---

## System Requirements

- Windows 10/11 (64-bit)
- Internet connection
- ~50MB disk space
- Python 3.10+ (for cleanup script only)
- No other software required — single self-contained binary

---

## Files

| File | Size | What it is |
|---|---|---|
| `polymarket-trader.exe` | 16 MB | Trader Edition (Windows) |
| `polymarket-us.exe` | 16 MB | Viewer Edition (Windows) |
| `.env` | 1 KB | Configuration (edit this) |
| `scripts/pg_aggregate_cleanup.py` | 15 KB | Database maintenance script |
| `scripts/run_cleanup.bat` | 0.1 KB | Windows scheduled task wrapper |
