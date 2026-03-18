# HyperLiquid Copy Trading Bot

This folder contains one of the three live HyperLiquid copy-trading bots in this repository.

It runs independently from the other bot folders, but uses the same overall engine:

- poll a target wallet on HyperLiquid
- detect changes in the copied positions
- translate those changes into your configured sizing model
- place mirrored orders on your account through the HyperLiquid SDK

## Important Context

This README is intended to explain the structure and behavior of the bot. It is not the source of truth for the exact live wallet address, coin list, or risk numbers currently deployed.

The live source of truth is:

1. the code in this folder
2. the environment variables configured for this bot's deployment

In other words, this bot copies one specific wallet and one specific set of coins, but those exact values are expected to come from Railway variables or a local `.env`, not from hardcoded claims in this README.

## Runtime Flow

1. Poll the target wallet every few seconds through HyperLiquid's public `/info` API.
2. Filter down to the configured `COPY_COINS` universe for this bot.
3. Compare the current snapshot to the previous snapshot.
4. Convert the target movement into the bot's own desired size using the configured sizing mode.
5. Execute an IOC order on your account through the HyperLiquid SDK.

When `COPY_RECONCILE_MODE=lifecycle`, the bot does more than just mirror a final net position. It anchors a copy ratio when the target opens a trade, then mirrors adds, trims, closes, and flips throughout that position lifecycle.

## Key Files

| File | Purpose |
|---|---|
| `bot.py` | Main process, startup sync, polling loop, reconciliation, heartbeat logging |
| `config.py` | Environment-variable loading, defaults, and validation |
| `tracker.py` | Polls the copied wallet and detects position changes |
| `copier.py` | Queries account state, computes sizes, sets leverage, and places orders |
| `.env.example` | Local configuration template |
| `requirements.txt` | Python dependencies |
| `analyze_strategy.py` | Helper script for reviewing trading behavior |
| `check_wallet.py` | Helper script for inspecting wallet state |
| `recent_fills.py` | Helper script for checking recent fills |

## Configuration

Railway is the normal source of truth in production. If running locally, create a `.env` file based on `.env.example`.

The most important variables are:

| Variable | Meaning |
|---|---|
| `HL_WALLET_ADDRESS` | Signer wallet address |
| `HL_PRIVATE_KEY` | Private key for signing |
| `HL_ACCOUNT_ADDRESS` | Trading account or agent wallet |
| `COPY_TARGET_ADDRESS` | Wallet being copied |
| `COPY_COINS` | Coins this bot is allowed to copy |
| `COPY_SCALING_MODE` | How copied trades are sized |
| `COPY_RECONCILE_MODE` | `state`, `delta`, or `lifecycle` |
| `COPY_SYNC_STARTUP` | Whether to sync into existing positions on startup |
| `COPY_LEVERAGE` | Default leverage for all coins |
| `COPY_LEVERAGE_OVERRIDES` | Per-coin leverage overrides (format: `HYPE:10,ZRO:3`) |
| `COPY_DRY_RUN` | Safety switch for simulated vs live execution |

### Per-Coin Leverage

By default, `COPY_LEVERAGE` applies to every coin. To set different leverage per coin, use `COPY_LEVERAGE_OVERRIDES`:

```
COPY_LEVERAGE=5
COPY_LEVERAGE_OVERRIDES=HYPE:10,ZRO:3
```

In this example HYPE would use 10x, ZRO would use 3x, and every other coin would use the default 5x. If `COPY_LEVERAGE_OVERRIDES` is not set, all coins use `COPY_LEVERAGE`.

Treat `.env.example` as an example template, not proof of the live production values.

## Reconcile Modes

- `state`: aim for a desired net position each cycle
- `delta`: trade only the detected net change since the last snapshot
- `lifecycle`: anchor a copy ratio on open, then mirror the full trade lifecycle

## Startup Behavior

By default, the bot can lock coins that were already open on the target when the process starts. That prevents the bot from blindly jumping into the middle of an existing trade.

If `COPY_SYNC_STARTUP=true`, the bot is allowed to sync into the target's already-open positions immediately. That is mainly useful for recovery after a restart or crash.

## Risk Guards

- `COPY_MAX_TRADE_USD` caps a single order's notional
- `COPY_MAX_POSITION_USD` caps resulting exposure
- `COPY_MIN_TRADE_USD` filters out trades below the exchange minimum
- `COPY_MAX_DAILY_TRADES` acts as a kill switch if trading activity spikes unexpectedly

## Running

Local entry point:

```bash
python bot.py
```

Production entry point:

```bash
python bot.py
```
