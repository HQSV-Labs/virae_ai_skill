# Predictdog Skill — Example Interactions

---

## New User Onboarding

**User:** "I want to trade on Predictdog" (no API key configured)

```
To get started with Predictdog:

1. Visit predictdog.xyz to create an account
2. Deposit USDC into your wallet (Wallet → Deposit)
3. Go to Settings → API Keys to generate your API key
4. Set PREDICTDOG_API_KEY in your environment

Once set up, you can search markets, view your portfolio, and trade — all from here.
```

---

**User:** has API key, but wallet not set up or balance is $0

When `POST /api/trade/readiness` returns `ready: false`:
```
Your wallet isn't ready for trading yet.

Reason: <requirements.reason or failed check reason>

Please visit predictdog.xyz to complete wallet setup, approvals, or deposit the required funds.

Once this is ready, come back and I'll execute the trade after you confirm it.
```

---

## Searching Markets

**User:** "search for bitcoin markets"

1. `GET /api/markets/search?q=bitcoin`
2. Display results:
   ```
   Found 5 markets for "bitcoin":

   1. Will BTC exceed $100k in 2025?
      Yes: 72¢ | No: 28¢ | Volume: $2.1M

   2. BTC above $80k by June?
      Yes: 45¢ | No: 55¢ | Volume: $850K
   ```
3. Ask: "Would you like to trade any of these?"

---

## Viewing Portfolio

**User:** "show my portfolio" / "what positions do I have?"

1. `GET /api/trade/open-positions`
2. `GET /api/trade/claimable-positions`
3. `GET /api/auth/me` (for balance)
4. Display:
   ```
   Portfolio Summary
   Balance: $42.50 USDC
   Total PnL: +$12.40
   Redeemable: $30.00

   Positions (3):
   BTC > $100k (Yes) - 50 shares @ 72¢ | +$8.50 (+30.9%)
   Trump wins 2024 (Yes) - 30 shares @ 85¢ | +$4.20 (+16.5%)
   ETH > $5k (Yes) - 20 shares @ 30¢ | -$0.30 (-5.0%)
   ```

---

## Checking PnL

**User:** "what's my PnL?" / "how am I doing?"

1. `GET /api/trade/open-positions` → `summary.totalPnl`
2. `GET /api/auth/me` → `user.wallet.proxyWalletAddress`
3. `GET /api/analytics/user/:address/portfolio-analytics`
4. Display:
   ```
   PnL Report
   Open positions: +$12.40
   All-time realized: +$85.20
   Win rate: 62% (18W / 11L)
   Total volume traded: $700
   Markets traded: 22
   ```

---

## Placing a Trade

**User:** "buy $20 Yes on BTC above 100k"

1. Search: `GET /api/markets/search?q=BTC above 100k`
2. From results: `markets[0].clobTokenIds[0]` = tokenId for "Yes", `clobTokenIds[1]` = tokenId for "No"
3. Check readiness: `POST /api/trade/readiness`
4. Check balance: `GET /api/auth/me` → `proxyBalanceUsd`
5. **Show confirmation before submitting:**
   ```
   Trade Confirmation
   Market: Will BTC exceed $100k in 2025?
   Side: BUY Yes
   Amount: $20.00 USDC
   Current price: 72¢
   Est. shares: ~27.8
   Balance after: $22.50

   Confirm? (yes/no)
   ```
6. On confirmation: `POST /api/trade/order`
   ```json
   { "tokenId": "...", "side": "BUY", "orderType": "MARKET", "amount": 20.00 }
   ```

---

## Placing a LIMIT BUY in Shares

**User:** "buy 20 shares Yes at 60 cents"

1. Search and resolve `tokenId`.
2. Check readiness:
   ```json
   {
      "venue": "POLYMARKET",
      "trade": {
         "tokenId": "token-id-yes",
         "side": "BUY",
         "orderType": "LIMIT",
         "amount": 20,
         "amountKind": "SHARES",
         "limitPrice": 0.60
      }
   }
   ```
3. Show confirmation:
   ```
   Trade Confirmation
   Market: Will BTC exceed $100k in 2025?
   Side: BUY Yes
   Order: LIMIT 20 shares @ 60¢
   Est. notional: $12.00 before fees

   Confirm? (yes/no)
   ```
4. On confirmation: `POST /api/trade/order` with the same trade fields.

---

## Placing a Recurring Crypto Trade With TP/SL

**User:** "buy BTC 5m up tp 0.65 sl 0.35"

1. Search: `GET /api/markets/search?q=BTC 5m up`
2. Resolve the recurring round event and map outcome `Up` to `markets[0].clobTokenIds[i]`
3. Check readiness: `POST /api/trade/readiness`
4. **Show confirmation before submitting:**
    ```
    Trade Confirmation
    Market: Bitcoin Up or Down - April 18, 2:50AM-2:55AM ET
    Side: BUY Up
    Amount: $1.00 USDC
    Current price: 49¢
    Est. shares: ~2.04
    TP: 65¢
    SL: 35¢

    Confirm? (yes/no)
    ```
5. On confirmation: `POST /api/trade/order`
    ```json
    {
       "tokenId": "token-id-up",
       "side": "BUY",
       "orderType": "MARKET",
       "amount": 1.0,
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

Notes:
- TP/SL is for recurring crypto BUY orders only.
- TP/SL values are decimal prices in `(0,1)` even if the user speaks in cents.
- In user-facing Polymarket summaries, present share prices in cents (`¢`).

---

## Managing Open Orders

**User:** "show my open orders" / "cancel my limit orders"

1. `GET /api/trade/open-orders`
2. Display:
   ```
   Open Orders (2):
   1. BUY Yes @ 60¢ - 20 shares remaining (BTC > $100k)
   2. SELL No @ 40¢ - 15 shares remaining (Trump wins)
   ```
3. If cancel requested, confirm: "Cancel order #1 for BTC > $100k? (yes/no)"
4. `POST /api/trade/order/cancel` with `{ "orderId": "..." }`

---

## Claiming Resolved Markets

**User:** "any markets I can claim?" / "redeem my winnings"

1. `GET /api/trade/claimable-positions`
2. If found:
   ```
   Claimable Positions:
   - Trump wins 2024 (Yes): 30 shares → ~$30.00

   Claim all? (yes/no)
   ```
3. `POST /api/trade/claim-batch`

---

## Checking BTC/ETH 15m Auto-Trade Status

**User:** "show my BTC15m auto-trade status"

1. `GET /api/auto-trade/btc-15m-tail/status`
2. `GET /api/auto-trade/btc-15m-tail/tasks`
3. `GET /api/auto-trade/btc-15m-tail/decisions?limit=20`
4. Display:
   ```
   BTC 15m Tail
   Status: Enabled
   Tasks: 1 active
   Last decision: skipped - no qualifying entry
   ```

If the user asks to enable, pause, resume, delete, or create a task, show the exact endpoint and body and wait for explicit confirmation. This is a persistent strategy, not a one-off market order.

---

## Reading Rewards and Copy Trading

**User:** "show my rewards and copy-trading status"

1. `GET /api/rewards/summary`
2. `GET /api/rewards/tasks/daily`
3. `GET /api/copy-trading/summary`
4. `GET /api/copy-trading/tasks`
5. Summarize current tier, rebates, daily tasks, copy-trading task state, and recent executions if requested.

Do not create or change copy-trading tasks without explicit confirmation.

---

## Predict.fun and Memecoin Reads

**User:** "show my Predict.fun account"

Use:
- `GET /api/predict/account`
- `GET /api/predict/readiness`
- `GET /api/predict/positions`
- `GET /api/predict/orders`

**User:** "show my memecoin portfolio"

Use:
- `GET /api/memecoin/tokens`
- `GET /api/memecoin/portfolio`
- `GET /api/memecoin/activity`

For Predict.fun order creation/removal or memecoin execute calls, show the exact action and wait for explicit confirmation.

---

## Error Handling

| Error kind | Message to user |
|-----------|----------------|
| `INSUFFICIENT_BALANCE` | "Not enough USDC. Your balance is $X." |
| `NO_LIQUIDITY` | "No liquidity at that price. Try a market order instead." |
| `ORDERBOOK_MISSING` | "Market not found or unavailable." |
| `MIN_ORDER_SIZE` | "The order is below the minimum size. Increase the amount and try again." |
| `MARKET_SLIPPAGE_EXCEEDED` | "The market moved beyond your slippage limit. Refresh the quote or raise the limit." |
| `CLOB_RATE_LIMITED` | "Too many requests. Please wait a moment." |
| `CLOB_UNAVAILABLE` | "Polymarket is temporarily unavailable. Please retry shortly." |
| `CLOUDFLARE_BLOCKED` | "The backend request was blocked upstream. This is not an API key problem." |
| `TURNKEY_UNAVAILABLE` | "Turnkey signing is temporarily unavailable. Please retry shortly." |
| `TX_SUBMISSION_TIMEOUT` | "Transaction submission timed out at the RPC provider. Please retry shortly." |
| API key missing required scope | "This API key is missing `<scope>`. Create or use a key with that scope at predictdog.xyz → Settings → API Keys." |
| invalid TP/SL | "TP/SL is only supported for recurring crypto BUY orders, and TP must be greater than SL." |
