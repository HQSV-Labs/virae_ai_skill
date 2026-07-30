# Virae API Reference

Base URL: `https://api.virae.ai`
Auth header: `x-api-key: $PREDICTDOG_API_KEY` or `Authorization: Bearer $PREDICTDOG_API_KEY`

---

## Auth / User

### GET /api/auth/me
Get current user info and wallet balance.

**Response:**
```json
{
  "ok": true,
  "user": {
    "id": 1,
    "wallet": {
      "proxyWalletAddress": "0x...",
      "proxyBalanceUsd": 42.50,
      "setupStatus": "COMPLETED"
    }
  }
}
```
Use `proxyWalletAddress` for analytics endpoints. Use `proxyBalanceUsd` for balance display.

If `wallet` is null or `setupStatus` ≠ `COMPLETED`, direct user to virae.ai to complete onboarding.

### API key scopes

API key requests may fail with:
```json
{ "ok": false, "error": "API key missing required scope: portfolio:read" }
```

Known scopes:
- `account:read`
- `portfolio:read`
- `trade:polymarket`
- `trade:predict`
- `trade:memecoin`
- `auto_trade:read`
- `auto_trade:manage`
- `auto_trade:activate` (live activation also requires `trade:polymarket`)

---

## Trading

### POST /api/trade/readiness
Check if user is ready to trade before placing orders. Must include the trade details.

**Request body:**
```json
{
  "venue": "POLYMARKET",
  "trade": {
    "tokenId": "string",
    "side": "BUY",
    "orderType": "LIMIT",
    "amount": 10.00,
    "amountKind": "NOTIONAL",
    "limitPrice": 0.65
  }
}
```
`limitPrice` is required only for LIMIT orders. `amountKind` is optional; use `"SHARES"` only for LIMIT BUY orders when `amount` means outcome shares instead of USDC notional.

**Success:**
```json
{
  "venue": "POLYMARKET",
  "ready": true,
  "requirements": { "ready": true, "code": "READY", "reason": null },
  "checks": {
    "gasReserve": { "checked": false, "ready": true, "code": "POLYMARKET_FUNDER_RELAY_ACTIVE", "reason": "Polygon Safe outer transactions are broadcast by the backend funder.", "assetSymbol": null, "available": null, "required": null, "amountKind": null, "decimals": null },
    "balanceSufficiency": { "checked": true, "ready": true, "code": "SUFFICIENT_BALANCE", "reason": null, "assetSymbol": "USDC.e", "available": "42.50", "required": "10.00", "amountKind": "DECIMAL", "decimals": 6 },
    "feeSufficiency": { "checked": true, "ready": true, "code": "SUFFICIENT_FEE_BALANCE", "reason": null, "assetSymbol": "USDC.e", "available": "42.50", "required": "0.10", "amountKind": "DECIMAL", "decimals": 6 }
  },
  "warnings": []
}
```

**Failure:** validation errors may use `{ "ok": false, "kind": "...", "error": "..." }`; readiness failures usually return the same response shape with `ready: false`.

| kind | Action |
|------|--------|
| `POLYMARKET_DEPOSIT_WALLET_NOT_READY`, `POLYMARKET_SAFE_NOT_READY`, `WALLET_NOT_INITIALIZED` | → virae.ai to complete setup |
| `INSUFFICIENT_BALANCE` | → virae.ai → Wallet → Deposit or top up the required asset |
| `APPROVALS_PENDING` | → virae.ai to approve contracts |

### POST /api/trade/order
Place a trade order.

**Request body:**
```json
{
  "tokenId": "string",
  "side": "BUY",
  "orderType": "LIMIT",
  "eventSlug": "btc-updown-5m-1710000000",
  "marketId": "market-id",
  "marketSlug": "market-slug",
  "marketQuestion": "BTC Up or Down...",
  "outcomeLabel": "Up",
  "outcomeIndex": 0,
  "amount": 10.00,
  "amountKind": "NOTIONAL",
  "limitPrice": 0.65,
  "expiresAt": "2026-07-06T12:00:00Z",
  "strategyContext": {
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
}
```

Field notes:
- `amount` keeps the standard default semantics: BUY = USDC amount, SELL = shares.
- LIMIT BUY supports `amountKind: "SHARES"` when the user specifies share quantity.
- MARKET orders may include `referencePrice` and `slippageBps` for slippage protection.
- `strategyContext` is optional and intended for recurring crypto BUY orders with strategy tracking or TP/SL.
- For ordinary Polymarket events, omit `strategyContext`.
- For user-facing Polymarket summaries, format share prices in cents (`¢`) instead of dollars.

**Success response:**
```json
{ "ok": true, "orderType": "MARKET", "result": { ... } }
```

**Error response:**
```json
{
  "ok": false,
  "kind": "INSUFFICIENT_BALANCE" | "NO_LIQUIDITY" | "ORDERBOOK_MISSING" | "MIN_ORDER_SIZE" | "MARKET_SLIPPAGE_EXCEEDED" | "CLOB_RATE_LIMITED" | "CLOB_UNAVAILABLE" | "CLOUDFLARE_BLOCKED" | "TURNKEY_UNAVAILABLE" | "TX_SUBMISSION_TIMEOUT",
  "error": "Human readable message",
  "retryAfterMs": 5000
}
```

### POST /api/trade/order/cancel
Cancel an open order.

**Request body:**
```json
{ "orderId": "string" }
```

**Success response:**
```json
{ "ok": true, "makerAddress": "0x...", "orderId": "string" }
```

### GET /api/trade/open-positions
Get current portfolio with PnL.

**Response:**
```json
{
  "ok": true,
  "makerAddress": "0x...",
  "positions": [
    {
      "venue": "polymarket",
      "title": "Will BTC exceed $100k by end of 2025?",
      "outcome": "Yes",
      "size": 50.0,
      "avgPrice": 0.55,
      "currentPrice": 0.72,
      "cashPnl": 8.50,
      "percentPnl": 30.9,
      "initialValue": 27.50,
      "currentValue": 36.00,
      "canSell": true
    }
  ],
  "summary": {
    "totalPositions": 3,
    "totalPnl": 12.40,
    "winningPositions": 2,
    "losingPositions": 1
  }
}
```

### GET /api/trade/open-orders
Get unfilled limit orders.

**Response:**
```json
{
  "ok": true,
  "orders": [
    {
      "venue": "polymarket",
      "orderId": "string",
      "side": "BUY",
      "price": 0.60,
      "originalSize": 20.0,
      "remainingSize": 20.0,
      "takeProfitPrice": 0.70,
      "stopLossPrice": 0.35,
      "createdAt": "2025-01-01T00:00:00Z",
      "expiresAt": "2025-01-02T00:00:00Z",
      "canCancel": true
    }
  ],
  "warnings": []
}
```

### GET /api/trade/claimable-positions
Get positions eligible for payout redemption (resolved markets).

### POST /api/trade/claim
Claim payout for a resolved position.

### POST /api/trade/claim-batch
Batch claim all claimable positions.

### POST /api/trade/fee/quote
Quote the Virae trading fee before a Polymarket order.

### POST /api/trade/exit-plan
Create a TP/SL exit plan for an existing Polymarket position. Confirm before calling.

### PATCH /api/trade/exit-plan
Update an existing TP/SL exit plan. Confirm before calling.

---

## Markets

### GET /api/markets/search?q=<query>
Search prediction markets by keyword. Supports any language — input is resolved to English keywords via LLM before searching.

**Query params:**
- `q` (required): free-text search query (English or other languages)

**Response:**
```json
{
  "ok": true,
  "query": "Bitcoin",
  "rawQuery": "比特币",
  "results": [
    {
      "venue": "polymarket",
      "id": "event-id",
      "slug": "will-btc-exceed-100k",
      "title": "Will BTC exceed $100k in 2025?",
      "volume": 2100000,
      "markets": [
        {
          "id": "market-id",
          "question": "Will BTC exceed $100k in 2025?",
          "outcomes": ["Yes", "No"],
          "prices": [0.72, 0.28],
          "clobTokenIds": ["token-id-yes", "token-id-no"]
        }
      ]
    },
    {
      "venue": "predict",
      "id": "predict-market-id",
      "slug": "btc-above-100k",
      "title": "BTC Above $100k",
      "question": "Will Bitcoin be above $100k?",
      "outcomes": ["Yes", "No"]
    }
  ]
}
```
**Trading from search results:**
- `outcomes[i]` maps to `clobTokenIds[i]` — use `clobTokenIds[i]` as the `tokenId` when placing an order for outcome `i`.
- Example: to buy "Yes" (index 0), use `markets[0].clobTokenIds[0]` as `tokenId` with `side: "BUY"`.

**Recurring crypto trading from search results:**
- Queries like `BTC 5m up`, `ETH 15m down`, or `SOL daily up` may resolve to recurring crypto rounds.
- If the matched event is a recurring crypto round and the user is placing a BUY order, attach a recurring `strategyContext`.
- If the user also specified TP/SL, place those values inside `strategyContext.riskConfig`.
- Generic recurring crypto definition ids are `polymarket-<asset>-<interval>-directional`.

### GET /api/events/aggregated
Browse top events across Polymarket and Kalshi.

**Response:**
```json
[
  {
    "id": "agg_...",
    "title": "2024 US Presidential Election",
    "volume": 1500000,
    "polymarket": { "price": 0.54, "url": "https://polymarket.com/event/...", "volume": 1000000 },
    "kalshi": { "price": 0.52, "url": "https://kalshi.com/markets/...", "volume": 500000 },
    "spread": 0.02,
    "roi": 0.037
  }
]
```

---

## Analytics / PnL

### GET /api/analytics/user/:address/portfolio-analytics
Historical PnL and trading statistics. Requires auth (`x-api-key` header).

**Path param:** `address` = user's proxy wallet address (get from `/api/auth/me`)

**Response:**
```json
{
  "ok": true,
  "walletAddress": "0x...",
  "analytics": {
    "currentPnl": 12.40,
    "realizedPnlAllTime": 85.20,
    "totalBoughtAllTime": 350.00,
    "cumulativeExecutedVolumeAllTimeUsd": 700.00,
    "historicalWinRate": 0.62,
    "wins": 18,
    "losses": 11,
    "closedPositionsCount": 29,
    "marketsTradedAllTime": 22,
    "computedAt": "2025-01-15T10:00:00Z"
  }
}
```

---

## Auto Trade

Use these strategy-neutral endpoints for API keys and Skills. Supported live task keys are `btc-15m-tail`, `eth-15m-tail`, and `musk-tweet-count`; `weather-temperature` is simulation-only. Tasks belong to the authenticated user.

### GET /api/auto-trade/strategies

Lists strategy capabilities, parameter groups, service-wide availability, and whether live task creation is supported.

### GET /api/auto-trade/strategies/:strategyKey/simulation

Returns the available simulation/shadow summary for the strategy.

### POST /api/auto-trade/tasks/validate

Validates and normalizes a prospective task without creating it.

```json
{
  "strategyKey": "btc-15m-tail",
  "name": "Conservative BTC tail",
  "perOrderAmountUsd": 10,
  "orderSizingMode": "FIXED_ORDER_AMOUNT",
  "entryConfig": {
    "askCap": 0.82,
    "minEntryAsk": 0.55
  },
  "riskConfig": {
    "dailyLossStopUsd": 30,
    "maxTradesPerDay": 8
  },
  "startImmediately": false
}
```

The response includes `normalized` parameters and `activation` availability. Validation requires `auto_trade:read` and does not mutate state.

Allowed task fields:

- `name`: string, 1–120 characters, or `null`
- `perOrderAmountUsd`: 1–100
- `orderSizingMode`: `FIXED_ORDER_AMOUNT` or `TASK_BANKROLL`
- `minRemainingBankrollUsd`: 0–100000 or `null`; valid only for `TASK_BANKROLL`
- `entryConfig`: strategy-specific fields below
- `riskConfig`: strategy-specific fields below
- `startImmediately`: create/validate only; defaults to `false`

BTC/ETH 15m Tail `entryConfig`:

- Price/signal: `askCap` (0.01–0.999), `minEntryAsk` (0.01–0.999), `edgeGateEnabled`, `minEdgeBps` (0–2000), `distanceGateEnabled`, `minDistancePercent` (0–5), `absoluteDistanceGateEnabled`, `minAbsoluteDistanceUsd` (0.01–10000)
- Stop behavior: `directionFlipStopEnabled`, `distanceCollapseStopEnabled`, `distanceCollapseStopPercent` (0–100), `orderbookStopEnabled`, `orderbookStopPrice` (0.01–0.99 or `null`), `orderbookStopSlippageBps` (10–2000)
- Entry timing: `entryWindowStartSeconds` and `entryWindowEndSeconds` (1–300), plus up to 12 `entryWindows` entries containing `secondsToEndMin` (1–300) and `minDistanceBps` (0–2000)
- Liquidity/orders: `maxSpread` and `maxSpreadHard` (0.001–0.05), `minLiquidityClob` (≥0), `depthMultiplier` (1–20), `entryOrderChaseEnabled`, `cancelOpenOrdersEnabled`, `cancelAfterMs` (1000–120000), `maxChaseTicks` (0–2)
- `maxNotionalUsd` (1–100) is accepted inside `entryConfig`, but must equal `perOrderAmountUsd` when both are supplied

BTC/ETH `riskConfig`: `dailyLossStopUsd` (1–10000), `dailyLossStopBehavior` (`AUTO_RESUME_NEXT_UTC_DAY` or `REQUIRE_MANUAL_RESUME`), `maxTaskNetLossUsd` (1–100000 or `null`), `consecutiveLossStop` (1–20), and `maxTradesPerDay` (1–200).

Musk Tweet Count `entryConfig`:

- Orders/allocation: `maxNotionalUsd`, `minOrderNotionalUsd` (1–100), `minExpectedProfitUsd` (0–100), `entryOrderTtlSeconds` (10–300), and allocation percentages `tailNoAllocationPct`, `lateDirectionalAllocationPct`, `lotteryAllocationPct`, `lotteryMaxSingleTradePct`, `nextMarketPrepositionPct` (0–1)
- Tail/directional signals: `lowTailBoundaryBufferTweets` (1–40), `lowTailMinAsk`/`highTailMinAsk` (0.01–0.99), `lowTailMaxAsk`/`highTailMaxAsk` (0.01–0.999), `highTailMaxRemainingHours` (1–48), `directionalMinRemainingHours` (0–24), `directionalMaxRemainingHours` (0.5–48)
- Burst/next market: `lotteryBurstRate30m` (0–100), `lotteryBurstRate60m` (0–200), `nextMarketPrepositionMaxHours` (1–96)

Musk `riskConfig`: `dailyLossStopUsd` (1–10000), `maxTaskNetLossUsd` (1–100000 or `null`), and `maxTradesPerDay` (1–200).

Unknown fields are rejected. Effective cross-field constraints are checked for both creation and patching. Runtime-only fields such as `mode`, internal hedge controls, global controls, and shadow-run settings are not writable through this API.

### POST /api/auto-trade/tasks

Creates a task. The request is the validation body above. `startImmediately` defaults to `false`; omit it or keep it false to create a paused task. Requires `auto_trade:manage`.

Starting immediately additionally requires `auto_trade:activate` plus `trade:polymarket` and geographic eligibility.

### GET /api/auto-trade/tasks

Query parameters:

- `strategyKey` — optional supported live strategy key
- `limit` — 1 to 100
- `cursor` — opaque task cursor from the prior response
- `includeArchived` — `true` to include stopped/deleted records

### GET /api/auto-trade/tasks/:taskId

Returns one owned task, including archived tasks.

### PATCH /api/auto-trade/tasks/:taskId

Updates one or more task fields. Allowed top-level fields are `name`, `perOrderAmountUsd`, `orderSizingMode`, `minRemainingBankrollUsd`, `entryConfig`, and `riskConfig`. Unknown fields and empty patches are rejected. Lifecycle changes use the pause/resume endpoints.

### POST /api/auto-trade/tasks/:taskId/pause
### POST /api/auto-trade/tasks/:taskId/resume
### DELETE /api/auto-trade/tasks/:taskId

Pause and delete require `auto_trade:manage`. Resume also requires `auto_trade:activate`, `trade:polymarket`, geographic eligibility, and service-wide live-trading availability.

### GET /api/auto-trade/tasks/:taskId/decisions

Supports `limit`, `cursor`, and optional `reasonCode`.

### GET /api/auto-trade/tasks/:taskId/audit-log

Supports `limit` and `cursor`. API-originated changes record the actor type, API key id, request id, and before/after values.

### GET /api/auto-trade/tasks/:taskId/pnl

Returns task-scoped PnL history.

### GET /api/auto-trade/tasks/:taskId/orders

Returns task-scoped order history. Supports `limit` and `cursor`.

### POST /api/auto-trade/tasks/:taskId/share
### GET /api/auto-trade/shares/:code
### POST /api/auto-trade/shares/:code/import

Creates, reads, or imports a task configuration share. Share import supports BTC/ETH 15m Tail and always creates a paused task. Share creation/import requires `auto_trade:manage`; authenticated share reads require `auto_trade:read`.

### GET /api/auto-trade/preferences
### PATCH /api/auto-trade/preferences

Reads or updates:

```json
{
  "emailAutoTradeSettlementEnabled": true,
  "telegramAutoTradeSettlementEnabled": true
}
```

All state-changing endpoints require an `Idempotency-Key` header of at most 200 characters. Reusing a key with the same operation and body returns the original response with `Idempotency-Replayed: true`; reusing it with a different body returns `409`.

Service-wide enablement, shadow-run configuration, and operations controls are intentionally not exposed to API keys.

---

## Rewards, Favorites, Copy Trading

Read-only endpoints commonly useful to agents:

- `GET /api/rewards/summary`
- `GET /api/rewards/ledger`
- `GET /api/rewards/tasks/daily`
- `GET /api/rewards/polymarket/user-markets`
- `GET /api/favorites`
- `GET /api/favorites/markets`
- `GET /api/copy-trading/tasks`
- `GET /api/copy-trading/executions`
- `GET /api/copy-trading/summary`

Mutation endpoints for rewards claims, boxes, gift codes, favorites, and copy-trading task changes require explicit user confirmation.

---

## Predict.fun

Predict.fun is separate from Polymarket and uses `PREDICT_FUN` readiness/trading semantics.

Read-only:
- `GET /api/predict/account`
- `GET /api/predict/readiness`
- `GET /api/predict/positions`
- `GET /api/predict/orders`
- `GET /api/predict/activity`
- `GET /api/predict/markets`
- `GET /api/predict/markets/:marketId`
- `GET /api/predict/markets/:marketId/orderbook`

Mutations:
- `POST /api/predict/activation/ensure`
- `POST /api/predict/approvals/ensure`
- `POST /api/predict/orders`
- `POST /api/predict/orders/remove`

Confirm before any mutation. Predict.fun readiness can also be checked through:
```json
{
  "venue": "PREDICT_FUN",
  "trade": { "...": "Predict.fun order body" }
}
```

---

## Memecoin

Read-only:
- `GET /api/memecoin/tokens`
- `GET /api/memecoin/portfolio`
- `GET /api/memecoin/activity`
- `GET /api/memecoin/:swapId`

Quote/execute:
- `POST /api/memecoin/quote`
- `POST /api/memecoin/execute`

Use `POST /api/trade/readiness` with `{ "venue": "MEMECOIN", "trade": { "quoteKey": "..." } }` after quote and before execute. Confirm the exact quote and execution before calling execute.

---

## Wallet / Balance

Balance is available via `/api/auth/me` → `user.wallet.proxyBalanceUsd` (USDC.e on Polygon).

### GET /api/funder/pol-balance
Get POL (gas token) balance.

---

## Restricted Endpoints (do not call)

The following are off-limits for this skill:
- `POST /api/wallet/deposit-plans` — deposits
- `GET /api/wallet/deposit-plans/*` — deposit status
- `POST /api/wallet/withdraw-plans` — withdrawals
- `POST /api/turnkey/withdraw/*` — Turnkey withdrawals
- `POST /api/swap/execute` — swaps unless the user explicitly requested and confirmed a swap quote
