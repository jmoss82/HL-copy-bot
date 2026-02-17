# HyperLiquid Copy Trading Bot

Monitors a target trader's perp positions on HyperLiquid in real-time and mirrors their trades onto your account.

## How It Works

1. **Poll** the target wallet every 3 seconds via the public `/info` API (no auth needed)
2. **Diff** their positions against the last snapshot to detect opens, closes, increases, decreases, and flips
3. **Scale** the detected change according to your configured ratio
4. **Execute** an IOC limit order through the spread on your account

## Files

| File | Purpose |
|---|---|
| `bot.py` | Main entry point, async loop, startup sync, heartbeat logging |
| `config.py` | All settings loaded from environment variables with defaults |
| `tracker.py` | Polls target wallet, diffs positions, classifies changes |
| `copier.py` | Executes mirrored trades on your account via the HL SDK |

## Setup

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure environment variables

Copy the template and fill in your credentials:

```bash
cp .env.example .env
```

**Required variables:**

| Variable | Description |
|---|---|
| `HL_WALLET_ADDRESS` | Your signer wallet address |
| `HL_PRIVATE_KEY` | Your private key |
| `HL_ACCOUNT_ADDRESS` | Your trading account (if using agent wallet) |
| `COPY_TARGET_ADDRESS` | Wallet address of the trader to copy |
| `COPY_FIXED_RATIO` | Trade size multiplier (0.1 = 10% of their size) |

**Optional overrides (sensible defaults built in):**

| Variable | Default | Description |
|---|---|---|
| `COPY_SCALING_MODE` | `fixed_ratio` | `fixed_ratio`, `proportional`, or `fixed_size` |
| `COPY_LEVERAGE` | `40` | Your leverage |
| `COPY_IS_CROSS` | `false` | `true` for cross margin, `false` for isolated |
| `COPY_POLL_INTERVAL` | `3.0` | Seconds between target polls |
| `COPY_SLIPPAGE_BPS` | `10.0` | Max slippage for IOC orders (basis points) |
| `COPY_MAX_POSITION_USD` | `5000` | Hard cap on notional exposure |
| `COPY_COINS` | `BTC` | Comma-separated coins to copy |
| `COPY_SYNC_STARTUP` | `true` | Match target position on startup |
| `COPY_MAX_DAILY_TRADES` | `200` | Kill switch if something goes wrong |
| `COPY_DRY_RUN` | `true` | No real orders until set to `false` |
| `COPY_LOG_LEVEL` | `INFO` | `DEBUG` for verbose output |

### 3. Run

```bash
python bot.py
```

Start with `COPY_DRY_RUN=true` to watch it detect trades without placing orders. Check the logs, then flip to `false` when ready.

## Deployment (Railway)

The bot is designed to run as a standalone Railway service. Set environment variables in the service's Variables tab — no `.env` file needed on the server. Entry point is `python bot.py`.

## Scaling Modes

- **`fixed_ratio`** — Multiply target's trade size by `COPY_FIXED_RATIO`. A ratio of `0.1` means you trade 10% of their size.
- **`proportional`** — Automatically scales based on `(your equity / their equity)`. Adjusts as account values change.
- **`fixed_size`** — Always trade `COPY_FIXED_SIZE` per signal regardless of target's size. Direction is matched.
