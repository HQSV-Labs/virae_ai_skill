---
name: predictdog
description: "Use the Virae API to search markets, view account/portfolio/PnL, place confirmed Polymarket trades including recurring crypto TP/SL entries, inspect or cancel open orders, claim resolved payouts, and read supported product surfaces such as rewards, favorites, Predict.fun, memecoin, copy-trading, and BTC/ETH 15m auto-trade status. Use when the user wants to interact with virae.ai from an AI agent, including requests like 'find BTC markets', 'buy $10 Yes', 'buy BTC 5m up tp 65 sl 35', 'show my portfolio', 'claim winnings', or 'check auto-trade status'. Requires PREDICTDOG_API_KEY configured or provided by the user."
---

# Virae Skill

Trade prediction markets and manage Virae account state via the Virae API.

## Setup

One value is required before any API call:
- `PREDICTDOG_API_KEY` — user's API key

Check for this as an environment variable first. If not found, ask the user:
> "Please provide your Virae API key. You can generate one at virae.ai → Settings → API Keys."

**Base URL is fixed:** `https://api.predictdog.xyz` — do not ask the user for this.

## Authentication

All requests require one of these auth forms:
```
x-api-key: <PREDICTDOG_API_KEY>
```
or
```
Authorization: Bearer <PREDICTDOG_API_KEY>
```

API keys can be scoped. If an endpoint returns `403` with `API key missing required scope: ...`, tell the user which scope is missing and ask them to create or use a key with that scope:
- `account:read` - account profile and settings reads
- `portfolio:read` - portfolio, positions, open orders, claimable payouts
- `trade:polymarket` - Polymarket order, readiness, cancel, fee quote, claim
- `trade:predict` - Predict.fun trading endpoints
- `trade:memecoin` - memecoin trading endpoints

## New User / Not Set Up

If the user doesn't have an account or API key yet, or if any API call returns a wallet setup error (`setupStatus` ≠ `COMPLETED`), direct them to the website:

> "To get started, visit **virae.ai** to create an account and deposit funds. Once set up, go to Settings → API Keys to generate your API key."

If the user has an API key but `POST /api/trade/readiness` returns `ready: false`, summarize `requirements.reason` first, then the failed `checks` fields.

Common readiness codes:

| Code/kind | Meaning | Response to user |
|-----------|---------|-----------------|
| `POLYMARKET_DEPOSIT_WALLET_NOT_READY`, `POLYMARKET_SAFE_NOT_READY`, `WALLET_NOT_INITIALIZED` | Trading wallet not ready | "Your trading wallet is not ready yet. Visit virae.ai to complete wallet setup." |
| `INSUFFICIENT_BALANCE` | Not enough trade collateral or gas token | "You do not have enough balance for this trade. Deposit or top up funds at virae.ai." |
| `APPROVALS_PENDING` | Contract approvals needed | "Please complete wallet setup or approvals at virae.ai before trading." |
| `CLOUDFLARE_BLOCKED`, `CLOB_UNAVAILABLE`, `TURNKEY_UNAVAILABLE`, `TX_SUBMISSION_TIMEOUT` | Temporary upstream/backend dependency issue | "Trading is temporarily unavailable. Please retry shortly." |

## Common Workflows

### 1. Search Markets
```
GET /api/markets/search?q=<query>
```
Returns matching events with `markets[].clobTokenIds` needed for trading.
Show title, outcomes, prices, and volume.

### 2. View Portfolio
```
GET /api/trade/open-positions
GET /api/trade/claimable-positions
GET /api/auth/me
```
Positions include `cashPnl`, `percentPnl`, `currentPrice`, `avgPrice`, `size`.
`/api/auth/me` gives wallet address and USDC balance (`proxyBalanceUsd`).
Claimable payouts are shown separately from open positions and may include settled markets ready to redeem.

### 3. Check PnL
Quick summary: use `summary.totalPnl` from `GET /api/trade/open-positions`.
Historical analytics:
1. `GET /api/auth/me` → extract `user.wallet.proxyWalletAddress`
2. `GET /api/analytics/user/:address/portfolio-analytics`

### 4. Place a Trade

To find the `tokenId` for trading:
1. Search with `GET /api/markets/search?q=<query>`
2. From results: `markets[outcomeIndex].clobTokenIds[outcomeIndex]` is the `tokenId`
   - `outcomes[0]` corresponds to `clobTokenIds[0]`, etc.

Before submitting:
- Check readiness — send the same trade details you're about to place:
  ```
  POST /api/trade/readiness
  Body: {
    "venue": "POLYMARKET",
    "trade": { "tokenId": "...", "side": "BUY", "orderType": "LIMIT", "amount": 10.00, "amountKind": "NOTIONAL", "limitPrice": 0.65 }
  }
  ```
- Handle readiness errors (see New User / Not Set Up above)
- Show confirmation to user and wait for explicit approval
- **Never trade without explicit user confirmation**

```
POST /api/trade/order
Body: { tokenId, side, orderType, amount, amountKind?, limitPrice?, referencePrice?, slippageBps?, expiresAt?, strategyContext? }
```

Amount semantics:
- Default BUY `amount` is USDC notional.
- Default SELL `amount` is shares.
- LIMIT BUY may use `amountKind: "SHARES"` when the user specifies shares instead of USDC.
- MARKET orders may include `referencePrice` and `slippageBps` for price protection.

### 4a. Recurring Crypto With TP/SL

Recurring crypto single-round entries are a distinct Polymarket trading surface. Treat queries like these as recurring crypto requests rather than ordinary event trades:
- `buy BTC 5m up`
- `buy ETH 15m down`
- `buy SOL hourly up`
- `buy BNB daily down tp 0.62 sl 0.30`

For recurring crypto BUY orders, attach `strategyContext` when you can resolve the round metadata. This records the entry and arms optional TP/SL exit handling.

Use `strategyContext` only when:
- venue is `POLYMARKET`
- the matched market is a recurring crypto round
- the user is placing a BUY order

Optional TP/SL support:
- TP/SL belongs inside `strategyContext.riskConfig`
- Both values are decimal prices in `(0,1)`
- If both are present, `takeProfitPrice` must be greater than `stopLossPrice`
- TP/SL is for recurring crypto BUY orders only

Use a generic definition id matching the asset and interval: `polymarket-<asset-lower>-<interval>-directional` (for example `polymarket-btc-5m-directional`, `polymarket-eth-15m-directional`, `polymarket-sol-1h-directional`).

Example `strategyContext`:
```json
{
  "definitionId": "polymarket-btc-5m-directional",
  "executionScopeKey": "btc-updown-5m-1710000000",
  "eventSlug": "btc-updown-5m-1710000000",
  "marketId": "market-id",
  "outcomeLabel": "Up",
  "riskConfig": {
    "takeProfitPrice": 0.65,
    "stopLossPrice": 0.35
  },
  "metadata": {
    "surface": "crypto-recurring",
    "asset": "BTC",
    "interval": "5m",
    "roundStartSec": 1710000000
  }
}
```

For user-facing Polymarket summaries, present share prices in cents (`¢`) rather than `$0.xx`.

### 4b. BTC/ETH 15m Auto-Trade Status

BTC/ETH 15m Tail is a long-running auto-trade task system, not a single order. Do not create, update, pause, resume, delete, enable, or disable auto-trade tasks unless the user explicitly asks for that exact action and confirms the final request body.

Safe read-only status endpoints:
```
GET /api/auto-trade/btc-15m-tail/status
GET /api/auto-trade/btc-15m-tail/tasks
GET /api/auto-trade/btc-15m-tail/decisions?limit=50
GET /api/auto-trade/eth-15m-tail/status
GET /api/auto-trade/eth-15m-tail/tasks
GET /api/auto-trade/eth-15m-tail/decisions?limit=50
```

When the user asks to "turn on auto trading", "create a BTC15m bot", or similar, explain that this is a persistent strategy task and ask for explicit confirmation before calling any POST/PATCH task or settings endpoint.

### 5. Cancel an Order
```
GET /api/trade/open-orders   ← list first
POST /api/trade/order/cancel
Body: { orderId }
```
Confirm with user before cancelling.

### 6. View Open Orders
```
GET /api/trade/open-orders
```

### 7. Claim Resolved Positions
```
GET /api/trade/claimable-positions
POST /api/trade/claim
POST /api/trade/claim-batch
```
Confirm with the user before submitting any claim transaction.

### 8. Other Product Reads

Use these for read-only summaries when the user asks:
```
GET /api/rewards/summary
GET /api/rewards/ledger
GET /api/rewards/tasks/daily
GET /api/favorites
GET /api/favorites/markets
GET /api/copy-trading/tasks
GET /api/copy-trading/executions
GET /api/copy-trading/summary
GET /api/predict/account
GET /api/predict/readiness
GET /api/predict/positions
GET /api/predict/orders
GET /api/memecoin/tokens
GET /api/memecoin/portfolio
GET /api/memecoin/activity
GET /api/airdrop/poly-score/me
```

For create/update/execute actions under copy-trading, Predict.fun, memecoin, rewards boxes/gift-codes, favorites, or user settings: show the exact action and wait for explicit user confirmation. For fund movement, use the restrictions below.

## Restrictions

- **No transfers**: Never call deposit or withdraw endpoints
  - `/api/wallet/deposit-plans`
  - `/api/wallet/withdraw-plans`
  - `/api/turnkey/withdraw/*`
- Always confirm before placing or cancelling any order
- Always confirm before claiming payouts, executing memecoin/Predict.fun trades, creating persistent auto-trade tasks, or changing copy-trading tasks
- For deposits and withdrawals, always redirect to virae.ai
- Do not execute swap routes from this skill unless the user explicitly asks for a swap and confirms the quote.

## API Reference

See [references/api.md](references/api.md) for complete endpoint details and response types.
See [references/examples.md](references/examples.md) for sample interactions including new user onboarding.
