# Virae Skill

Trade on Virae, view portfolio/PnL, and inspect supported product surfaces from your AI agent.

This skill connects to the [Virae](https://www.virae.ai) API and works with **Claude Code** and **OpenClaw**.

## What You Can Do

- **Search markets** — "find BTC markets", "search Trump election"
- **View portfolio** — positions, PnL, open orders
- **Place trades** — "buy $20 Yes on BTC above 100k"
- **Trade recurring crypto rounds** — "buy BTC 5m up", optionally with `tp/sl`
- **Cancel orders** — list and cancel unfilled limit orders
- **Claim payouts** — redeem resolved market winnings
- **Check product status** — rewards, favorites, copy trading, Predict.fun, memecoin, and BTC/ETH 15m auto-trade status

## Prerequisites

You need a Virae API key:

1. Sign up at [virae.ai](https://www.virae.ai)
2. Complete wallet setup and deposit funds if you plan to trade
3. Go to **Settings → API Keys** and generate a key with the scopes your agent needs

Common scopes:
- `account:read`
- `portfolio:read`
- `trade:polymarket`
- `trade:predict`
- `trade:memecoin`

Set it as an environment variable:

```bash
export PREDICTDOG_API_KEY=pd_pat_your_key_here
```

Or just tell your agent the key when it asks.

## Installation

### Claude Code

```bash
# Download and install
curl -L https://github.com/HQSV-Labs/predictdog_skill/releases/latest/download/predictdog-skill.skill \
  -o predictdog-skill.skill

# Unzip into Claude skills directory
unzip predictdog-skill.skill -d ~/.claude/skills/
```

Or clone and install manually:

```bash
git clone git@github.com:HQSV-Labs/predictdog_skill.git ~/.claude/skills/predictdog-skill
```

### OpenClaw

```bash
git clone git@github.com:HQSV-Labs/predictdog_skill.git ~/.openclaw/skills/predictdog-skill
```

Or via the OpenClaw CLI (if available):

```bash
openclaw skills install github:HQSV-Labs/predictdog_skill
```

## Usage

Once installed, just talk to your agent naturally:

```
"search for NBA finals markets"
"what's my portfolio?"
"buy $10 Yes on Spurs winning the NBA Finals"
"buy BTC 5m up"
"buy BTC 5m up tp 0.65 sl 0.35"
"show my open orders"
"what's my PnL?"
"show my rewards"
"check BTC15m auto-trade status"
```

The agent will call the Virae API and handle everything allowed by the skill, including confirming trades before executing them.

For recurring crypto BUY orders on Polymarket, the agent may attach a recurring `strategyContext` with optional TP/SL risk config so the trade is tracked as a recurring crypto strategy entry.

BTC/ETH 15m auto-trade is a persistent strategy system. The skill supports read-only status checks by default. Creating, changing, enabling, pausing, resuming, or deleting strategy tasks requires explicit confirmation of the exact action.

## API

This skill uses the Virae REST API at `https://api.predictdog.xyz`. See [`references/api.md`](references/api.md) for the full endpoint reference.

Deposits and withdrawals are intentionally disabled in this skill — use [virae.ai](https://www.virae.ai) for fund management.

## License

MIT
