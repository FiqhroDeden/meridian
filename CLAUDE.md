# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Autonomous DLMM liquidity provider agent for Meteora pools on Solana. ESM (`"type": "module"`), Node ≥18.

---

## Commands

```bash
npm start              # run the daemon (REPL + cron + Telegram polling) — index.js
npm run dev            # same, but DRY_RUN=true (no on-chain txs)
npm run setup          # interactive wizard: writes .env + user-config.json
npm test               # == test:syntax — node --check on every *.js (the "lint")
npm run test:syntax    # syntax-check all JS files
npm run test:screen    # node test/test-screening.js (live Meteora screening, no LLM)
npm run test:agent     # DRY_RUN agent loop smoke test (node test/test-agent.js)
npm run pm2:start      # pm2 start ecosystem.config.cjs (production process mgr)
npm run pm2:restart    # pm2 restart meridian --update-env
npm run pm2:logs       # tail pm2 logs

node cli.js <cmd> [--dry-run] [--json]   # one-shot tool invocation, JSON out (see Entry Points)
```

- **No test framework / no real linter.** "Tests" are standalone scripts; `npm test` only does a syntax check. There is no single-test runner beyond running the script file directly.
- `postinstall` runs `scripts/patch-anchor.js`, which rewrites `@coral-xyz/anchor` + `@meteora-ag/dlmm` for Node ESM bare-directory-import compatibility. If you `rm -rf node_modules` or bump those deps and hit `ERR_UNSUPPORTED_DIR_IMPORT`, re-run `node scripts/patch-anchor.js`.
- `.env` may be encrypted (`envcrypt.js`); `npm run env:encrypt` encrypts marked keys. The CLI also loads `~/.meridian/.env`.

---

## Entry Points

There are **two** ways to run the agent — both share `config.js`, `tools/`, and all state files:

1. **`index.js` — the daemon.** REPL + `node-cron` jobs (management every `managementIntervalMin`, screening every `screeningIntervalMin`) + Telegram long-polling. This is the autonomous loop. `npm start`.
2. **`cli.js` (`meridian` bin) — agent-native one-shot CLI.** Each subcommand calls a tool directly and prints JSON to stdout — no LLM, no cron. Subcommands: `balance`, `positions`, `pnl`, `candidates`, `token-info|-holders|-narrative`, `pool-detail`, `search-pools`, `active-bin`, `deploy`, `claim`, `close`, `swap`, `screen`, `manage`, `config`, `study`, `lessons`, `pool-memory`, `evolve`, `blacklist`, `performance`, `discord-signals`, `withdraw-liquidity`, `add-liquidity`, `start`. `node cli.js help` (or no args) prints a generated SKILL.md. The `.claude/commands/*.md` slash commands and `.claude/agents/{manager,screener}.md` subagents wrap these CLI verbs.

`DRY_RUN` must be set before tool imports — `cli.js` does this for `--dry-run`; for the daemon use `npm run dev`.

---

## Architecture Overview

```
index.js            Daemon entry: REPL + cron orchestration + Telegram bot polling
cli.js              `meridian` bin: one-shot tool invocation, JSON output (no LLM/cron)
agent.js            ReAct loop (OpenRouter/OpenAI-compatible): LLM → tool call → repeat
config.js           Runtime config from user-config.json + .env; exposes config object
setup.js            Interactive setup wizard (npm run setup)
envcrypt.js         Loads .env, transparently decrypts keys marked "# encrypted"
prompt.js           Builds system prompt per agent role (SCREENER / MANAGER / GENERAL)
state.js            Position registry (state.json): tracks bin ranges, OOR timestamps, notes
lessons.js          Learning engine: records closed-position perf, derives lessons, evolves thresholds
signal-weights.js   Darwinian signal weighting (signal-weights.json) — boosts signals seen in winners
signal-tracker.js   In-memory staging of screening signals (10-min TTL; not yet persisted at deploy)
decision-log.js     Rolling log of agent decisions (decision-log.json, last 100)
pool-memory.js      Per-pool deploy history + snapshots (pool-memory.json)
strategy-library.js Saved LP strategies (strategy-library.json)
briefing.js         Daily Telegram briefing (HTML)
telegram.js         Telegram bot: polling, notifications (deploy/close/swap/OOR)
hivemind.js         Agent Meridian HiveMind sync
smart-wallets.js    KOL/alpha wallet tracker (smart-wallets.json)
token-blacklist.js  Permanent token-mint blacklist (token-blacklist.json)
dev-blocklist.js    Deployer-wallet blocklist (dev-blocklist.json) — hard-filters in screening
logger.js           Daily-rotating log files + action audit trail

tools/
  definitions.js    Tool schemas in OpenAI format (what LLM sees)
  executor.js       Tool dispatch: name → fn, safety checks, pre/post hooks
  dlmm.js           Meteora DLMM SDK wrapper (deploy, close, claim, positions, PnL)
  screening.js      Pool discovery from Meteora API
  wallet.js         SOL/token balances (Helius) + Jupiter swap
  token.js          Token info/holders/narrative (Jupiter API)
  study.js          Top LPer study via LPAgent API
  okx.js            OKX DEX API: smart-money signals, holder/bundle advanced-info
  chart-indicators.js  OHLCV-derived TA signals via Agent Meridian API
  agent-meridian.js Agent Meridian API client (HiveMind / public API base + auth)

discord-listener/   Standalone sub-package (own package.json): selfbot that watches
                    Discord channels for token calls → discord-signals.json (see below)
scripts/patch-anchor.js  postinstall: patch anchor/dlmm for Node ESM
```

---

## Agent Roles & Tool Access

Three agent roles filter which tools the LLM can call:

| Role | Purpose | Key Tools |
|------|---------|-----------|
| `SCREENER` | Find and deploy new positions | deploy_position, get_top_candidates, get_token_holders, check_smart_wallets_on_pool |
| `MANAGER` | Manage open positions | close_position, claim_fees, swap_token, get_position_pnl, set_position_note |
| `GENERAL` | Chat / manual commands | All tools |

Sets defined in `agent.js:6-7`. If you add a tool, also add it to the relevant set(s).

---

## Adding a New Tool

1. **`tools/definitions.js`** — Add OpenAI-format schema object to the `tools` array
2. **`tools/executor.js`** — Add `tool_name: functionImpl` to `toolMap`
3. **`agent.js`** — Add tool name to `MANAGER_TOOLS` and/or `SCREENER_TOOLS` if role-restricted
4. If the tool writes on-chain state, add it to `WRITE_TOOLS` in executor.js for safety checks

---

## Config System

`config.js` loads `user-config.json` at startup. Runtime mutations go through `update_config` tool (executor.js) which:
- Updates the live `config` object immediately
- Persists to `user-config.json`
- Restarts cron jobs if intervals changed

**Valid config keys and their sections:**

| Key | Section | Default |
|-----|---------|---------|
| minFeeActiveTvlRatio | screening | 0.05 |
| minTvl / maxTvl | screening | 10k / 150k |
| minVolume | screening | 500 |
| minOrganic | screening | 60 |
| minHolders | screening | 500 |
| minMcap / maxMcap | screening | 150k / 10M |
| minBinStep / maxBinStep | screening | 80 / 125 |
| timeframe | screening | "5m" |
| category | screening | "trending" |
| minTokenFeesSol | screening | 30 |
| maxBundlersPct | screening | 30 |
| maxTop10Pct | screening | 60 |
| blockedLaunchpads | screening | [] |
| deployAmountSol | management | 0.5 |
| maxDeployAmount | risk | 50 |
| maxPositions | risk | 3 |
| gasReserve | management | 0.2 |
| positionSizePct | management | 0.35 |
| minSolToOpen | management | 0.55 |
| outOfRangeWaitMinutes | management | 30 |
| managementIntervalMin | schedule | 10 |
| screeningIntervalMin | schedule | 30 |
| managementModel / generalModel | llm | openrouter/healer-alpha |
| screeningModel | llm | openrouter/hunter-alpha |

**`computeDeployAmount(walletSol)`** — scales position size with wallet balance (compounding). Formula: `clamp(deployable × positionSizePct, floor=deployAmountSol, ceil=maxDeployAmount)`.

---

## Position Lifecycle

1. **Deploy**: `deploy_position` → executor safety checks → `trackPosition()` in state.js → Telegram notify
2. **Monitor**: management cron → `getMyPositions()` → `getPositionPnl()` → OOR detection → pool-memory snapshots
3. **Close**: `close_position` → `recordPerformance()` in lessons.js → auto-swap base token to SOL → Telegram notify
4. **Learn**: `evolveThresholds()` runs on performance data → updates config.screening → persists to user-config.json

---

## Screener Safety Checks (executor.js)

Before `deploy_position` executes:
- `bin_step` must be within `[minBinStep, maxBinStep]`
- `volatility` must be a positive finite number when provided; fresh pool detail with volatility 0/null is rejected
- Total range must be at least `max(35, minBinsBelow)` bins; 1-bin/tiny deploys are refused
- Position count must be below `maxPositions` (force-fresh scan, no cache)
- No duplicate pool allowed (same pool_address)
- No duplicate base token allowed (same base_mint in another pool)
- `amount_x > 0` is rejected. Deploys are single-side SOL only (`amount_y` / `amount_sol`)
- SOL balance must cover `amount_y + gasReserve`
- `blockedLaunchpads` enforced in `getTopCandidates()` before LLM sees candidates

---

## bins_below Calculation (SCREENER)

Linear formula based on positive pool volatility (set in screener prompt, `index.js`):

```
bins_below = round(minBinsBelow + (volatility / 5) * (maxBinsBelow - minBinsBelow)), clamped to [minBinsBelow, maxBinsBelow]
```

- Default clamp is `[35, 69]`
- `volatility <= 0`, null, or non-finite → skip/refuse deploy
- High volatility (5+) → maxBinsBelow
- Any value in between is valid (continuous, not tiered)

---

## Telegram Commands

Handled directly in `index.js` (bypass LLM):

| Command | Action |
|---------|--------|
| `/positions` | List open positions with progress bar |
| `/close <n>` | Close position by list index |
| `/set <n> <note>` | Set note on position by list index |

Progress bar format: `[████████░░░░░░░░░░░░] 40%` (no bin numbers, no arrows)

---

## Race Condition: Double Deploy

`_screeningLastTriggered` in index.js prevents concurrent screener invocations. Management cycle sets this before triggering screener. Also, `deploy_position` safety check uses `force: true` on `getMyPositions()` for a fresh count.

---

## Bundler Detection (token.js)

Two signals used in `getTokenHolders()`:
- `common_funder` — multiple wallets funded by same source
- `funded_same_window` — multiple wallets funded in same time window

**Thresholds in config**: `maxBundlersPct` (default 30%), `maxTop10Pct` (default 60%)
Jupiter audit API: `botHoldersPercentage` (5–25% is normal for legitimate tokens)

---

## Base Fee Calculation (dlmm.js)

Read from pool object at deploy time:
```js
const baseFactor = pool.lbPair.parameters?.baseFactor ?? 0;
const actualBaseFee = baseFactor > 0
  ? parseFloat((baseFactor * actualBinStep / 1e6 * 100).toFixed(4))
  : null;
```

---

## Model Configuration

- Default model: `process.env.LLM_MODEL` or `openrouter/healer-alpha`
- Fallback on 502/503/529: `stepfun/step-3.5-flash:free` (2nd attempt), then retry
- Per-role models: `managementModel`, `screeningModel`, `generalModel` in user-config.json
- LM Studio: set `LLM_BASE_URL=http://localhost:1234/v1` and `LLM_API_KEY=lm-studio`
- `maxOutputTokens` minimum: 2048 (free models may have lower limits causing empty responses)

---

## Lessons System

`lessons.js` records closed position performance and auto-derives lessons. Key points:
- `getLessonsForPrompt({ agentType })` — injects relevant lessons into system prompt
- `evolveThresholds()` — adjusts screening thresholds based on winners vs losers
- Performance recorded via `recordPerformance()` called from executor.js after `close_position`
- **Known issue**: `evolveThresholds()` references `maxVolatility` and `minFeeTvlRatio` but config.js uses `minFeeActiveTvlRatio` and has no `maxVolatility` key — the evolution of these keys is a no-op

---

## Signal Weighting & Decision Log

- **`signal-weights.js`** — Darwinian weighting of the 10 screening signals (`organic_score`, `fee_tvl_ratio`, `volume`, `mcap`, `holder_count`, `smart_wallets_present`, `narrative_quality`, `study_win_rate`, `hive_consensus`, `volatility`). Signals that appear in profitable closes get boosted; those in losers decay. Weights persist in `signal-weights.json` and are injected into the screener prompt.
- **`signal-tracker.js`** — stages a candidate's signals in-memory (10-min TTL) between screening and the LLM decision. Deploy-time persistence is **not yet wired**, so this is short-lived context, not durable attribution data.
- **`decision-log.js`** — `appendDecision()` writes a sanitized, capped (100-entry) rolling log of agent decisions to `decision-log.json` for later review.

---

## Discord Signal Pipeline

`discord-listener/` is a **separate package** (install with `cd discord-listener && npm install`). It uses a Discord *selfbot* (`discord.js-selfbot-v13`, a personal account token, not a bot token) to watch configured channels for Solana addresses. Each address runs `pre-checks.js`: dedup (10-min) → token-blacklist → resolve to Meteora DLMM pool → deployer check vs `deployer-blacklist.json` → min-fees (`DISCORD_MIN_FEES_SOL`). Passing signals are written to `discord-signals.json` with status `pending`; the screener (`/screen`, `cli.js screen`, and the screening cron) consumes them as priority candidates before the normal cycle.

---

## Three Separate Blocklists

Don't confuse them:

| File | Keyed by | Loaded by | Enforced where |
|------|----------|-----------|----------------|
| `token-blacklist.json` | token **mint** | `token-blacklist.js` | screening + discord pre-checks |
| `dev-blocklist.json` | **deployer wallet** | `dev-blocklist.js` (`isDevBlocked`) | screening hard-filter before LLM; editable via Telegram |
| `deployer-blacklist.json` | deployer wallet (`addresses[]`) | `discord-listener/pre-checks.js` | Discord pipeline only |

---

## HiveMind

Agent Meridian HiveMind sync is handled by `hivemind.js`; the shared API client lives in `tools/agent-meridian.js` (base `https://api.agentmeridian.xyz/api`, auth via `PUBLIC_API_KEY`). It uses built-in Agent Meridian defaults unless overridden by config or env.

---

## Environment Variables

| Var | Required | Purpose |
|-----|----------|---------|
| `WALLET_PRIVATE_KEY` | Yes | Base58 or JSON array private key |
| `RPC_URL` | Yes | Solana RPC endpoint |
| `OPENROUTER_API_KEY` | Yes | LLM API key |
| `TELEGRAM_BOT_TOKEN` | No | Telegram notifications |
| `TELEGRAM_CHAT_ID` | No | Telegram chat target |
| `LLM_BASE_URL` | No | Override for local LLM (e.g. LM Studio) |
| `LLM_MODEL` | No | Override default model |
| `DRY_RUN` | No | Skip all on-chain transactions |
| `HIVE_MIND_URL` | No | Collective intelligence server |
| `HIVE_MIND_API_KEY` | No | Hive mind auth token |
| `HELIUS_API_KEY` | No | Enhanced wallet balance data |
| `PUBLIC_API_KEY` | No | Agent Meridian public API auth (`x-api-key`) |
| `AGENT_MERIDIAN_API_URL` | No | Override Agent Meridian API base |
| `OKX_API_KEY` / `OKX_SECRET_KEY` / `OKX_PASSPHRASE` / `OKX_PROJECT_ID` | No | OKX DEX authed endpoints (public smart-money works without) |
| `JUPITER_API_KEY` / `JUPITER_REFERRAL_ACCOUNT` / `JUPITER_REFERRAL_FEE_BPS` | No | Jupiter swap referral config |
| `DISCORD_USER_TOKEN` / `DISCORD_GUILD_ID` / `DISCORD_CHANNEL_IDS` / `DISCORD_MIN_FEES_SOL` | No | Discord listener (selfbot) |
| `ENVRYPT_KEY` / `ENVCRYPT_KEY` | No | Key to decrypt `# encrypted` .env values (else `.envrypt` file) |

---

## Known Issues / Tech Debt

- `lessons.js evolveThresholds()` evolves `maxVolatility` + `minFeeTvlRatio` (wrong key names — should be `minFeeActiveTvlRatio`; `maxVolatility` doesn't exist in config at all). The evolution is a no-op for those keys.
- `get_wallet_positions` tool (dlmm.js) is in definitions.js but not in MANAGER_TOOLS or SCREENER_TOOLS — only available in GENERAL role.
