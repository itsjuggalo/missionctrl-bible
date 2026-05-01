# MissionCtrl Bible v3

Living documentation of the entire Mission Control trading operation. Updated continuously as the system evolves. This file is the source of truth — when a script's purpose is unclear, when a process gets killed by mistake, when the agents need to know what reads what — start here.

| Field | Value |
|---|---|
| Owner | Mike (itsjuggalo, Discord 733531188752285716) |
| Server | Oracle Cloud Ubuntu 22.04.5 ARM, 5.8 GB RAM |
| Public IP | 132.145.205.15 |
| Public DNS | missionctrl.serveftp.com |
| Hostname | openclaw |
| Files Documented | 763 (from Apr 28 full recon) |
| PM2 Processes | 26 (24 baseline + boba-qa + trail-daemon) |
| System Cron Entries | 77 |
| OpenClaw Cron Jobs | 19 |
| State Files | 521 |
| Secret Files | 176 |
| v2 Generated | April 28, 2026 |
| v3 Reconciled | April 30, 2026 |

---

## Table of Contents

1. System Overview
2. PM2 Process Registry (26 processes)
3. System Crontab (77 entries — see Bible v2 PDF for verbatim list)
4. OpenClaw Cron Jobs (19 jobs — see Bible v2 PDF for verbatim list)
5. ~/scripts/ (72 files — see Bible v2 PDF for full inventory)
6. ~/.openclaw/ (160 files)
7. ~/mission-control-restored/ (219 files)
8. ~/mission-control/ (155 files)
9. ~/go-trader/ (96 files)
10. ~/Kronos/ (36 files)
11. ~/coupon-claw/ (25 files)
12. State Files (521 JSON)
13. Secrets (176 files)
14. Discord Channel Map (169 channels)
15. Telegram Channels (8 sources)
16. Alpaca and Trading Architecture
17. Boba Decision Cycle (v3 — Protocol A/B + T0–T4 ladder)
18. Build and Workflow Rules
19. Common Debugging Recipes
20. Appendix (IDs, ports, recovery)
21. **NEW — April 28–30 Updates** (post-v2 changes)
22. **NEW — Active Backlog** (open work)
23. **CHANGELOG** (append-only log)

> For the full file-by-file inventory of chapters 3–15, refer to MissionCtrl_Bible_v2.pdf in this same repo. This BIBLE.md captures the high-level architecture + post-v2 changes. The PDF is regenerated on demand via `bible-pdf`.

---

## Chapter 1 — System Overview

Mission Control is a multi-agent algorithmic trading + signal-aggregation system. Six AI agents reason over options flow, crypto signals, and market data from 20+ sources, then execute trades on Alpaca paper accounts. Dashboard at missionctrl.serveftp.com on port 3033 (proxied via nginx). All processes managed by PM2.

### 1.1 Top-Level Directories

- `~/scripts/` — Active backbone — 72 root + subdir scripts
- `~/.openclaw/` — OpenClaw runtime — agents, secrets (176), skills, workspace state
- `~/mission-control-restored/` — Next.js dashboard (port 3033) — 219 files
- `~/mission-control/` — Service backbone — 155 files across 18 subdirs
- `~/go-trader/` — Go execution engine — 8 platforms, 96 files
- `~/Kronos/` — BTC + multi-asset forecast model — 36 files
- `~/coupon-claw/` — Standalone Discord coupon validation bot — 25 files
- `~/.openclaw/data/` — Live data files including BIBLE.md (this file) + agent-tasks.json

### 1.2 The Six AI Agents

**Boba** (Claude Sonnet 4.6 via Anthropic API) — Orchestrator and primary decision-maker. Reads whale flow, Kronos forecasts, Firebase signals; outputs trade picks; executes on Alpaca R2 paper. Cycle runs at 11:00, 13:30, 15:00, 20:00 ET on weekdays plus every 4 hours for crypto-only. Auth-locked to OWNER_ID 733531188752285716. Critical bug fixed Apr 29: EXIT path must cancel related_orders before close. Decision cycle ID: cfcf6645-d175-478c-bd9f-e97c00a1e690.

**Orion** (Gemini 2.5 Flash) — Scanner / technical analyst. Generates morning flow brief at 8:45 ET premarket, runs intraday TA, posts to Telegram team channel. Free tier 20 req/day quota — pace requests.

**JazzyHazzy** (Gemini / GPT-4o-mini) — Research / news / earnings. Catalyst calendar, earnings preview, news sentiment, sector rotation. 5 weekday brief crons via brief_saver.py.

**Grok** (xAI grok-3-mini via Telegram bridge) — X/social search. Runs x_search on tickers in active flow alerts, posts sentiment summaries to #agent-debate, responds to !askgrok commands.

**DeepSheet** (DeepSeek API on Telegram) — Long-form research, fundamental + macro analysis pulls, responds to Telegram queries on demand.

**Kronos** (consultant-only 6th agent) — Generates price forecasts on demand only (~3 min generation time). Provides forecast to Boba when called — NOT a hard gate. Helps Boba on stocks/crypto/options info when needed; does not influence Boba's pick ranking automatically.

Plus two non-trading agents: main (admin) and claude-code (dev).

### 1.3 Cross-LLM Bridges

In addition to the primary agents, two Telegram-bridged LLMs are accessible for ad-hoc queries:

- **grok_telegram.py** (PM2: grok-bot) — Sends Telegram chat messages to Grok via xai_api_key. Bot responds in same chat.
- **deepseek_telegram.py** (PM2: deepseek-bot) — Same pattern with DeepSeek via deepseek_api_key.

These are NOT called from inside Boba's decision cycle — they're user-facing chat bridges that let Mike message Grok/DeepSeek directly through Telegram.

---

## Chapter 2 — PM2 Process Registry (26 processes)

> **CRITICAL RULE — read before any pm2 stop/delete:** Before killing or deleting any PM2 process, check this section to see what reads its outputs and what it writes. The Apr 30 incident where telegram-listener was killed almost broke the entire crypto signal pipeline because the assumption that it duplicated tgbridge was wrong — they use entirely different auth (Telethon user-mode vs bot HTTP) and serve different purposes.

### 2.1 Critical processes (DO NOT KILL without checking downstream)

**telegram-listener** — `~/mission-control/telegram-listener/listener.py` — Telethon user-mode MTProto session (Mike's @itsjuggalo). Reads 8 Telegram channels + Firebase Vivid2.json + WebSocket aicryptosignals.pro. Writes signals.json (rotates at 500 entries) + channel_stats.json. Downstream consumers: `codex-expert/codex_expert.py`, `tweak/tweak_signals.py`. **Different auth from tgbridge — they NEVER conflict.**

**tgbridge** — `~/scripts/tg-bridge/tgbridge.py` — Telegram bot HTTP API (token at ~/.openclaw/secrets/telegram-bot-token.txt). Polls Telegram getUpdates for command-center group, forwards to Discord webhook. **NOT same as telegram-listener.**

**firebase-signal-relay** — `~/mission-control-restored/bots/firebase_signal_relay.py` — Polls Firebase Name/Name2/Vivid feeds at stock-signal-72772-default-rtdb.firebaseio.com. Writes ~/.openclaw/workspace/directives/firebase_trade_signals.json. Consumed by Boba via load_firebase_signals().

**signal-receiver** — `~/mission-control/signal-receiver/` (port 8420) — TradingView webhook receiver, secret at ~/.openclaw/secrets/tradingview_webhook_secret. Proxied via nginx /webhook/, /signals/, /health.

**mission-control** — `~/mission-control-restored/` — Next.js dashboard prod on port 3033 (`next start`). Build cmd: `npm run build && pm2 restart mission-control`.

**boba-mega** — `~/scripts/boba-bot/` — 4-block Boba bot: scan, Q&A, backtest, trade arm/disarm.

**boba-autothread** — `~/scripts/boba-bot/` — Auto-creates Discord threads on big flow ($5M+).

**boba-qa** *(NEW Apr 28)* — `~/scripts/boba_qa_bot.py` — Bot identity BobaUpgraded#0602, PM2 id 31, listens on #boba-cmd. Natural language Q&A on flow data with T0–T4 ladder + repeater/DTE scoring. Reuses discord_boba_token.

**trail-daemon** *(NEW Apr 28)* — `~/scripts/trail_daemon.py` — Ratcheting stops for swing positions: +50% / +100% / +200% tier cuts. Tier ladder shipped in commit efde570. Manages R2 swing positions only; ignores positions tagged as flow protocol.

**position_sell_daemon** — Manages R1 ($150K, PA3Y7UTNIQFZ, key PKYB76T2) — L2 stop_limit-only. Strategy: PROTECT_ENTRY at +30%, RIDE at +50%+. Holds (Apr 28): CAR/CMCSA/IWM/SFM.

### 2.2 Full PM2 list (26 processes)

| Name | CWD | Purpose |
|---|---|---|
| askgrok-bot | /home/ubuntu | !askgrok Discord commands |
| boba-autothread | ~/scripts/boba-bot | Auto Discord threads on big flow |
| boba-mega | ~/scripts/boba-bot | 4-block Boba bot |
| **boba-qa** *(NEW)* | ~/scripts | BobaUpgraded#0602 Q&A bot |
| brief-forwarder | ~/mission-control-restored | Forward briefs to channels |
| captcha-bot | ~/scripts/captcha-bot | Captcha handling |
| coupon-claw | ~/coupon-claw | Coupon scraping |
| coupon-monitor | ~/coupon-claw | Coupon alerts |
| deepseek-bot | ~/mission-control-restored | DeepSeek model on Telegram |
| firebase-signal-relay | ~/mission-control-restored | Firebase poller |
| flow-monitor | ~/mission-control/signal-receiver | Live options flow ingestion |
| grok-bot | ~/mission-control-restored | Grok x_search |
| liq-collector | ~/scripts/liquidation-tracker | Liquidation data |
| mission-control | ~/mission-control-restored | Next.js dashboard port 3033 |
| option-signals | ~/mission-control-restored/Option-Signals-Scraper | Option Signals scraper |
| option-signals-relay | ~/mission-control-restored/Option-Signals-Scraper | Relays scraped signals |
| reminder-bot | ~/scripts | Reminder system |
| signal-receiver | ~/mission-control/signal-receiver | TradingView webhook port 8420 |
| skill-scheduler | ~/.openclaw/workspace | Cron-batch skill execution |
| spacer-bot | ~/scripts/spacer-bot | Discord channel spacer |
| status-bot | ~/scripts | System status reports |
| synth-control-bot | ~/mission-control-restored/Option-Signals-Scraper | Synth/control orchestration |
| telegram-listener | ~/mission-control/telegram-listener | Telethon scraper (8 channels) |
| tgbridge | ~/scripts/tg-bridge | TG bot → Discord webhook |
| todo-bot | /home/ubuntu | Todo management |
| **trail-daemon** *(NEW)* | ~/scripts | Ratcheting stops for swing positions |
| watchlist-editor-bot | ~/.openclaw/secrets | Watchlist editor |

---

## Chapter 16 — Alpaca & Trading Architecture

### 16.1 Two Paper Accounts

| Field | R1 (on hold) | R2 (Boba's active) |
|---|---|---|
| Account ID | PA3Y7UTNIQFZ | PA3R6MOPBWF7 |
| Status | +35% from $100K, Options L2 | $503K, Level 3, Margin 4x |
| OCO Support | stop_limit-only on options | Native order_class:oco |
| Key Files | alpaca-r1-key-id, alpaca-r1-secret | alpaca-key-id, alpaca-secret |
| Key Prefix | PKYB76T2 | PKCY4TP4 |
| Used By | auto_trader.py + position_sell_daemon | boba_decision_cycle.py + trail_daemon |
| Holdings (Apr 28) | CAR, CMCSA, IWM, SFM | Boba's daily picks |

### 16.2 Layer 2 OCO (shipped Apr 25)

execute_pick_on_alpaca uses native Alpaca order_class:oco with take_profit + stop_loss dicts. Single API call, broker manages mutual exclusion. Stocks + options only (NOT crypto). Test: TSLA GTC $376.74, TP $394.79, SL $357.19, commit 6af6485. Bracket-cleaner daemon retired.

### 16.3 Position Intent Bug Fix (Apr 29)

**All 6 sell paths must include `"position_intent": "sell_to_close"`** on options sell orders:

1. EXIT path (full close)
2. TRIM_25 (partial sell 25%)
3. TRIM_50 (partial sell 50%)
4. TIGHTEN_STOP OCO take-profit leg
5. Initial protective stop-limit
6. OCO take-profit replacement

Without it, Alpaca defaults to sell_to_open which triggers cash-secured-put BP requirements and fails with insufficient buying power. Patched on all 6 paths Apr 29.

### 16.4 EXIT Path Cancel Fix (Apr 29)

EXIT path must `cancel related_orders` before close — otherwise Alpaca returns 403 "insufficient qty" because the protective stop-limit and the close order are both trying to sell the same contracts.

### 16.5 Tradier Sandbox

Account VA65395018, option_level 6, margin. Used by position_sell_daemon (stop pricing), squeeze_scanner (IV), earnings_bot (ATM straddle), pattern-bot/ohlc_provider (active OHLC). Cache: ~/.openclaw/workspace/state/tradier_cache/ 60s TTL.

### 16.6 Other Wallets (WalletsPage.tsx, auth gate disabled Apr 27)

Alpaca R1+R2 merged with R1/R2 badge per row; Coinbase Live ($2,925); Robinhood Stocks ($6,087, 27 positions); Robinhood Crypto ($3,822); Hyperliquid go-trader ($15,526); Hyperliquid Personal ($99).

---

## Chapter 17 — Boba Decision Cycle (v3)

File: `~/scripts/boba_decision_cycle.py` (~1397 lines after Apr 28 patches). Boba is the only agent that places trades directly.

### 17.1 Schedule

- 11:00 ET — Pre-market open
- 13:30 ET — Mid-morning
- 15:00 ET — Pre-close
- 20:00 ET — After-hours
- Every 4 hours — `--crypto-only` mode

### 17.2 Workflow (9 steps)

1. Read scored signals from `~/mission-control/signal-receiver/data/scored_signals_recent.json`
2. Filter to new since last run (dedupe via boba_seen_signals.json)
3. Build top-5 shortlist by score with ticker variety
4. Trigger kronos_forecast_v2.py per ticker (fire-and-forget, SAMPLE_COUNT=1)
5. Build prompt: AGENT_IDENTITIES.md + Alpaca R2 state + signals + Kronos + Firebase signals + best-options + grok_brief — all loaded with `or ""` guards to prevent TypeError when source missing
6. Call Claude Sonnet (anthropic_api_key)
7. Parse JSON response (picks + reasoning + sizing)
8. For each pick: execute on R2 via order_class:oco → post Telegram team → post BobaTrades webhook → log to ops
9. Update boba_seen_signals.json

### 17.3 Prompt v3 — Protocol A / Protocol B (shipped Apr 28)

**Two trading protocols** rendered into every cycle prompt:

**Protocol A — Flow** (short-term, signal-driven): Reads T1+T2 unusual flow, requires SWEEP + (A/AA) + Vol>OI + same-day NY 4AM–8PM ET. Aggressive premium ladder (T0–T4). Targets 1–7 DTE moves on conviction.

**Protocol B — Swing** (multi-week, fundamentals-driven): Long DTE only (≥45 DTE), ITM/ATM strikes (delta 0.50–0.70), sector strength gate, earnings posture, Kronos/Orion regime alignment, hard -25% loser cutoff, trail winners via trail_daemon.

### 17.4 T0–T4 Premium Ladder (shipped Apr 28)

Tier-based premium thresholds for flow ranking:

| Tier | Threshold | Treatment |
|---|---|---|
| T0 | $10M+ | Mega-conviction, top priority |
| T1 | $5M+ | Huge — must enumerate first |
| T2 | $1M+ | Large — must enumerate before T3 |
| T3 | $500K+ | Unusual huge — Vol>OI + SWEEP + (A/AA) |
| T4 | $100K+ | Standard unusual flow — Vol>OI |

`$100–250K is NOT white.` Commit 256bcfc.

### 17.5 Schema v2 — Journal Fields (shipped Apr 28)

Every Boba pick now journaled with:

- `protocol`: "flow" | "swing" | "manual" | "auto-btc"
- `entry_criteria`: list of strings (e.g. `["T2_1M+", "sweep", "AA", "repeater_3x", "kronos_agrees"]` for flow; `["dte_77", "delta_0.48", "iv_pct_42", "oi_7859", "earnings_45d", "kronos_agrees"]` for swing)
- `brief_context`: list of source citations *(Apr 28 Step 11)* — at least one of regime/Grok/Orion/Kronos. Format: `["regime=VOLATILE_CHOP_neutral", "grok=tech_drag_continues", "orion=NVDA_oversold_RSI32", "kronos=agrees_high"]`. If no brief context used, must explicitly state `"none used because <specific reason>"` — never blank.

Reasoning checklist now has 8 items (was 7). Item 8 = BRIEF CONTEXT CITED.

For non-HOLD position actions, the `reason` field MUST cite specific brief context (e.g. "Kronos now CONFLICTS — flipped from bearish to bullish", "Orion BTC RSI 78 overbought + funding rate extreme = take profit"). Commit `boba prompt v3`.

### 17.6 Defensive Loaders (Apr 29 fix)

`load_market_briefing()`, `load_grok_brief()`, `load_orion_skills()`, `load_firebase_signals()`, `load_best_options()` — all wrapped in `or ""` to prevent TypeError when source missing. Any of these returning None caused the entire decision cycle to crash before this fix.

### 17.7 Safety Caps

- **Killswitch**: `~/.openclaw/workspace/state/boba_killswitch` — if exists, abort. Set by cost_watchdog when burn rate > $1/hr. Cleared by `~/scripts/unpause_boba.sh`.
- **Max picks/cycle**: 3
- **Max risk/cycle**: $3000
- **Cost watchdog**: Hourly Anthropic burn check, disables openclaw cron cfcf6645-d175-478c-bd9f-e97c00a1e690

---

## Chapter 18 — Build & Workflow Rules

### 18.1 Git Discipline

- Pre-edit: `git commit -am 'pre-edit: <file>'`
- Post-success: `git commit -am '✓ <what changed>' && git push`
- Rollback: `git checkout <file>` (one step back)

### 18.2 Build Verification

After every `npm run build && pm2 restart`, ALWAYS:

```
grep -l '<distinctive_string>' .next/static/chunks/*.js
```

Build exit 0 + HTTP 200 do **NOT** prove code shipped. Only positive grep does. If grep misses, diagnose: wrong file? Turbopack cache stale? PM2 stale? Multiple render paths? Backup files in `src/` being compiled?

### 18.3 React JSX Patch Verification

Build passing does NOT prove a JSX-referenced variable was declared. After any patch adding new state/useMemo/useEffect for a variable used in JSX, ALWAYS grep the declaration and confirm its line falls INSIDE the component function before pm2 restart. Verify all 4 pieces present for new derived values: interface, useState, useEffect, useMemo.

### 18.4 React useEffect Deps

useEffect deps must reference state/refs declared above the effect, NEVER derived consts declared further down — causes TDZ ReferenceError in production minified builds. Depend on underlying state and dereference inside the effect.

### 18.5 Communication Rules

- **PuTTY rule**: Only PuTTY commands in code blocks. No prose in blocks. No placeholders inside blocks.
- **Prompts rule**: Whole response in ONE copy-pasteable code block when Mike asks for a prompt.
- **Numbering rule**: Single continuous flat numbered list. No subsections, no restart.
- **Workaholic rule**: No "sleep well" / "call it a night" / "great stopping point". Default: "what's next?"
- **No credential rotation reminders**: Mike has explicit standing instruction.
- **Time format**: Always EST/ET in display. Server is UTC; convert before showing.

### 18.6 No Restore — Fix In Place

NEVER restore as first option. Diagnose specific error, make targeted fix.

### 18.7 No Mock Data, Preserve Features

- No mock data in MC dashboard.
- Swap data sources only — don't rebuild layouts unless asked.
- Preserve features by default. Ask before stripping any.
- **NEVER** remove processes/scripts/files, kill PM2 processes, archive logfiles, or delete code without explicit Mike approval.
- Layout requests stay layout-only.
- Fix-in-place supersedes cleanup instinct even when something looks orphan/duplicate/dead.

### 18.8 Unusual Flow Priority Rule

Boba/Grok/!flowpicks apply T0–T4 ladder BEFORE ranking top 3 each day. Same-day NY 4AM–8PM ET only. Must enumerate T0+T1+T2 first.

---

## Chapter 19 — Common Debugging Recipes

### 19.1 Dashboard 503 / connection refused

1. `pm2 list` — confirm mission-control online
2. `pm2 logs mission-control --lines 200`
3. `curl -I http://localhost:3033`
4. `sudo nginx -t && sudo systemctl status nginx`
5. If crashed: `cd ~/mission-control-restored && npm run build` first, then pm2 restart
6. gateway-watchdog.sh runs hourly to auto-restart on HTTP non-200

### 19.2 Boba not trading

1. `cat ~/.openclaw/workspace/state/boba_killswitch`
2. `cat ~/.openclaw/workspace/state/cost_watchdog.json`
3. `~/scripts/unpause_boba.sh`
4. `tail -f /tmp/boba_cycle.log`
5. `cat ~/.openclaw/workspace/skill_outputs/boba_decisions_validated.json | jq '.[-5:]'`

### 19.3 Telegram listener memory bloat

1. Symptom: PM2 telegram-listener > 400MB
2. Fix: `pm2 restart telegram-listener`
3. Watch: `pm2 monit`
4. Cause: Telethon SQLite + signals.json accumulation (capped 500 by SIGNALS_MAX)

### 19.4 Discord webhook 401/404

1. `ls -la ~/.openclaw/secrets/discord*<name>*`
2. `curl -X POST $(cat ~/.openclaw/secrets/discord_X_webhook) -H 'Content-Type: application/json' -d '{"content":"test"}'`
3. 404 = webhook deleted; 401/403 = bot token invalid

### 19.5 Build success but code didn't ship

1. `DISTINCTIVE='myUniquePatchString123'`
2. `grep -l "$DISTINCTIVE" .next/static/chunks/*.js`
3. If no match: `rm -rf .next; npm run build; pm2 restart mission-control`; re-grep
4. Check for multiple render paths (OptionsPage.tsx vs OptionsPage.tsx.bak)

### 19.6 Firebase pipeline stale

1. `pm2 logs firebase-signal-relay --lines 50`
2. `cat ~/.openclaw/workspace/directives/firebase_trade_signals.json | jq 'last'`
3. `curl https://stock-signal-72772-default-rtdb.firebaseio.com/Vivid2.json | head -c 200`
4. Apr 25 fix: build_embed was a typo for format_signal

### 19.7 Disk full

1. Currently 81% used, 8.7 GB free
2. `du -sh ~/full-backup-* ~/ubuntu-backup-* ~/UpgradedMC*.zip`
3. `du -sh ~/.openclaw/workspace/state/liquidations/binance_liqs.jsonl`
4. `journalctl --vacuum-time=7d`
5. `pm2 flush`

---

## Chapter 20 — Appendix

### 20.1 Owner / Server IDs

- Mike's Discord User ID: 733531188752285716
- Discord Guild ID: 1486025777970548908
- Verified Role ID: 1486033851594838118
- Telegram Command Center: -5066886378
- Telegram TG_CHAT_ID briefs: 7888676328
- Telethon API_ID: 31236866
- GitHub username: itsjuggalo
- Hostname: openclaw
- Public IP: 132.145.205.15
- Public DNS: missionctrl.serveftp.com
- Massage site IP: 150.136.39.179
- Massage site DNS: massagebymike.serveftp.com
- OS: Ubuntu 22.04.5 LTS aarch64
- Shape: VM.Standard.A2.Flex
- RAM / Disk: 5.8 GB / 45 GB (81% used)
- Boba decision cycle ID: cfcf6645-d175-478c-bd9f-e97c00a1e690

### 20.2 Ports In Use

| Port | Service | Description |
|---|---|---|
| 80 | nginx HTTP | /webhook/, /signals/, /health → :8420; rest → 301 HTTPS |
| 443 | nginx HTTPS | Dashboard via SSL (Let's Encrypt) |
| 3000 | (reserved) | Legacy port — mission-control config still references |
| 3033 | Next.js dashboard | ACTIVE port, mission-control PM2 process |
| 8420 | signal-receiver | TradingView webhook receiver |

### 20.3 Recovery Cheat Sheet

```bash
# Restart everything
pm2 restart all && pm2 save

# Check what's down
pm2 list

# Re-enable Boba after killswitch
~/scripts/unpause_boba.sh

# Force rebuild dashboard (clear Turbopack cache + stale chunks)
cd ~/mission-control-restored && rm -rf .next && npm run build && pm2 restart mission-control

# Rotate logs
pm2 flush

# Reload nginx
sudo nginx -t && sudo systemctl reload nginx

# Check disk
df -h /

# Trigger Boba manually (bypass cron)
python3 ~/scripts/boba_decision_cycle.py

# Trigger Boba crypto-only
python3 ~/scripts/boba_decision_cycle.py --crypto-only

# Trigger backup to GitHub
~/scripts/run_backup.sh
```

### 20.4 Backup Repos

- `itsjuggalo/4.23.2026MissionCtrlV2` — primary backup
  - main (e49d47fd7) — ~/mission-control-restored full snapshot
  - master (716a5e0) — ~/scripts including secrets/env
  - secrets-branch (7b50ff3a) — ~/.openclaw/secrets/ full backup
- `itsjuggalo/missionctrl-bible-recon` — recon dumps for Bible regeneration (private)
- `itsjuggalo/missionctrl-bible` — this BIBLE.md as README + PDF (public)

---

## Chapter 21 — April 28–30 Updates (post-v2)

Everything shipped between Bible v2 generation (Apr 28) and Bible v3 reconciliation (Apr 30).

### 21.1 Boba Upgrade — 6 Steps Shipped Apr 28

1. **Step 1 — Schema v2 journal** (commits be9911a05 + fa9d622): protocol + entry_criteria + brief_context fields on every pick. Backfilled across cycles.
2. **Step 2 — Boba prompt Protocol A/B + T0–T4 ladder** (commit 256bcfc): 14,948-char prompt rendered every cycle. File grew 1340 → 1397 lines.
3. **Step 3 — boba_qa_bot** (commit cb9cc23): PM2 id 31, BobaUpgraded#0602 identity, listens #boba-cmd. T0–T4 ladder + repeater/DTE scoring.
4. **Step 4 — trail_daemon** (commit efde570): PM2 id 32, ratchets +50/+100/+200% stops on R2 swing positions.
5. **Step 5 — PerformancePage protocol filter** (commit 637778da5): All/Flow/Swing/Manual/Auto-BTC dropdown with localStorage + per-protocol counts. Build verified via "Filter by Protocol" string in 0o7dy.o9jx2iy.js bundle.
6. **Step 6 — Dead-code quarantine**: `boba_decision_cycle.mc-restored-scripts.py` (Apr 27 stale copy at ~/mission-control-restored/scripts/) — moved to quarantine, NOT deleted. Zero cron/PM2/import refs confirmed before move. Plus 4 large backup tarballs (Apr 24-25, ~2GB) moved to quarantine.

### 21.2 Step 11 — Brief Context Citation (Apr 28)

Boba prompt v3 hardening. Required `brief_context` field per pick + auditable non-HOLD reasons + macro/regime/grok/orion/kronos source naming. Reasoning checklist item 8 added: BRIEF CONTEXT CITED. See Chapter 17.5.

### 21.3 best-options-logger Pipeline (Apr 29)

- **Path**: `~/scripts/best-options-logger/best_options_logger.py`
- **Cron**: `*/15 8-23 * * 1-5` (every 15 min, 4AM–8PM ET, weekdays)
- **Output**: `~/.openclaw/data/best-options/YYYY-MM-DD.json` — daily archive
- **Consumed by**: Boba prompt at line 620 — top 15 T1+T2 contracts injected via `load_best_options()`

### 21.4 FlowHistoryPage at /flow-history (Apr 29)

New page in TRADING section right after OptionsWatcher. Component: `FlowHistoryPage.tsx`. Redirect stub at `/app/flow-history/page.tsx`. Routes through Sidebar `case 'Flow History'`. Shows historical/archived flow data with timestamp + staleness coloring.

### 21.5 /flowtop Slash Command + !boba Auto-Context (Apr 29)

- `/flowtop` — Discord slash command on boba_mega bot, returns top T1+T2 flow with whale data
- `!boba` — Auto-injects whale flow + best_options + Firebase context into Boba responses

### 21.6 1970 Epoch Bug Fix (Apr 29)

Firebase `Expiry` field is returned as **seconds**, not milliseconds. JS `new Date(N)` treats N as ms, so without `*1000` multiplier, all expiries displayed as Jan 21 1970. Fixed via `*1000` multiplier in OptionsPage + Sidebar `fmtExpiry`. Commit 537b5bd96.

### 21.7 BEST OPTIONS Sidebar Year Display (Apr 29)

Sidebar widget now shows year (12/18 → 12/18/26) so contracts spanning year-end aren't ambiguous.

### 21.8 Apr 30 Session — UI Unification + Tasks Wipe

- **Flow row unification** (commit d2789b4e3): All 3 flow blocks (FLOW ALERTS / UNUSUAL FLOW / HUGE FLOW) use identical 7-col grid `34px 42px 50px 82px 44px 1fr 90px` with shared legend strip (▲bullish ▼bearish ⚡sweep ▦block C call P put). Same `cat()`/`catColor()` helpers (huge=#e040fb, unusual=#ffd600, repeat=#4fc3f7, weekly=#ff9800, etf=#66bb6a).
- **ActivityPage rebuild** (commits 33aadf5dd → 520bbc89c → e26d90f17): Row grouping with ×N count badge + severity colors (errors red `#ef5350`, warns orange `#ff9800`, errors get tinted bg `#1a0a0d`). Then split into per-agent block grid (auto-fill min 420px). Each agent gets bordered card with header (icon, name, type, total events, error count) + scrollable internal body.
- **/api/activity 409 filter** (commit 4dc538c4b): Skips `Telegram API error: 409|Conflict: terminated by other getUpdates` lines from PM2 logs without touching any process or logfile. Verified: 100 events / 0 with 409 remaining.
- **Tasks page rebuild + security wipe** (commits 086625825 + 1c0dcac47): Wiped 14 hardcoded personal tasks (including "Rotate Exposed Keys — Hyperliquid private key shared in chat") that were exposed in frontend bundle. Replaced with agent operations dashboard reading `/api/agent-tasks` → `~/.openclaw/data/agent-tasks.json`. 6 agent columns (Boba/JazzyHazzy/Orion/Grok/DeepSheet/Kronos) showing CORE duties + NEW discovered tasks.
- **Agent task helper**: `~/scripts/lib/agent_tasks.py` — atomic flock+os.replace, dedupe, max 50 discovered/agent. Signature: `add_discovered_task(agent, task, source, ticker)`.
- **Bible system**: This file + bible-log/show/pdf helpers + GitHub repo (Apr 30).

### 21.9 Personal Mission Board (Wiped from public TasksPage, kept here for record)

- **BACKLOG**: Pine Scripts (Holy Grail / DayTrader v1 / Grid Bot v1) [HIGH, Boba]; Boba Brief Prompt Upgrade [HIGH, Boba]; 30m SuperTrend Layer [MEDIUM, Boba]; Massage Therapy Website [MEDIUM]; WebSocket Real-Time Updates [MEDIUM]; Wire Remaining Pages (Scanner/Risk/Office Terran theme) [MEDIUM]
- **IN PROGRESS**: Rotate Exposed Keys — Hyperliquid [CRITICAL]; Signals + Telegram Pages [HIGH]; Hardcoded API Keys (4 source files) [HIGH]
- **DONE**: Wallet Page Complete; SQLite Database (8 tables, 282 trades); SuperTrend Auto-Trader [Boba]; Hybrid Brief System [Boba]; Security Hardening (secrets consolidated chmod 600)

---

## Chapter 22 — Active Backlog (Open Work)

### 22.1 Self-Grading Weekly Loop (NOT YET BUILT)

From Apr 29 YouTube review insight. New Friday 4PM ET cron: Boba reads the week's journal + R1/R2 P&L vs SPY/QQQ + Kronos consults + open trades, outputs `~/.openclaw/workspace/proposals/YYYY-MM-DD-weekly-grade.md` with letter grade A–F and 3–5 proposed edits to SOUL/IDENTITY/skills/rules. Mike reviews Saturday morning, accepts/rejects, commits.

### 22.2 Opus 4.7 Upgrade for Boba (PENDING DECISION)

Apr 29 — Opus 4.7 benchmark "Agentic Financial Analysis" jumped to 64.4% (+4 over 4.6). If Boba's orchestrator is on `claude-opus-4-6`, swap to `claude-opus-4-7` is likely free alpha on synthesis layer. Caveat: agentic search regressed in 4.7 — non-issue here since x_search is on Grok. Verify via `grep -r "claude-opus" ~/scripts/ ~/mission-control-restored/`.

### 22.3 Repos to Evaluate (in priority order)

1. **EVO** — auto-research orchestrator. Loop: hypothesis → experiment → measure → keep wins → discard losses. Highest-leverage for Pine Script parameter sweeps against existing gate criteria (BTCUSD/SPY/QQQ/ETH 6/12/24mo, beat HODL, DD<25%, WR>40%).
2. **AutoForge / GSD / Superpowers** — PRD-driven engineering with sub-agents. Pick ONE when tackling MC pages overhaul backlog (8 pages pending).
3. **Firecrawl** — sandboxed browser scraper, free tier. Pairs with MarkItDown for SEC Form 4/13F scanner build.
4. **MarkItDown** (Microsoft) — PDF/Word/Excel → clean markdown for LLM consumption. Reduces token cost.
5. **Kronos** — 10-min repo check for updates since wired in.

### 22.4 MC Pages Overhaul Backlog

Projects, usage, tasks (DONE Apr 30), risk, journal, scanner, regime — apply Terran theme. Office page also pending.

### 22.5 Mobile Layouts Pass

Audit which pages currently break on 380px viewport. Build `useIsMobile()` hook + `<MobileTable>` card pattern. Apply page-by-page: Performance, Signals, Telegram, Activity, Options, Positions, Trades, Scanner, Risk, Regime, Powertrader, Wallets. Single commit per page.

### 22.6 Pine Script Strategies

- Whale Catcher v1.2 — DONE (+11.34% PF 1.227)
- SuperTrend 1H v2.2 — needs work
- DayTrader v3 5m — queued
- 30m ST second layer on 1H — queued
- Holy Grail 4-layer — queued
- Grid Bot — queued

Gate criteria: BTCUSD/SPY/QQQ/ETH 6/12/24mo, beat HODL, DD<25%, WR>40%.

### 22.7 SPY 18-Put Protective Stop Blocker (FIXED Apr 29)

Boba EXIT path needs cancel related_orders before close — fixed in code, see Chapter 16.4.

### 22.8 Branch Reconciliation

main vs master diverged on 4.23.2026MissionCtrlV2 backup repo. Need eventual reconciliation pass.

### 22.9 StarCraft Marine vs Protoss Landing Page Animation

Pixel-art RTS battle animation built in 2 iterations using original sprites (IP-safe) with gold-trimmed marine matching MISSION CTRL palette. Pending integration.

---

## Chapter 23 — CHANGELOG

Append-only log of bible updates. Format: `### YYYY-MM-DD HH:MM ET — Title`

<!-- bible-log entries appended below -->

### 2026-04-30 — Bible v3 reconciliation pass

- **Source**: Read Bible v2 PDF (71 pages, Apr 28) + 28 conversation transcripts from Apr 27–30 archive
- **Added Chapter 21**: April 28–30 Updates — captures 6-step Boba upgrade, Step 11 brief_context, best-options-logger, FlowHistoryPage, /flowtop + !boba, 1970 epoch fix, BEST OPTIONS year display, Apr 30 UI unification + Tasks wipe
- **Added Chapter 22**: Active Backlog — self-grading loop, Opus 4.7 upgrade, repo evals (EVO/AutoForge/Firecrawl/MarkItDown), mobile layouts pass, Pine Scripts queue, branch reconciliation, StarCraft animation
- **Updated Chapter 2**: PM2 Process Registry — added boba-qa (id 31, BobaUpgraded#0602) + trail-daemon (id 32). Now 26 processes. Added critical-process safety section flagging telegram-listener vs tgbridge distinction (Apr 30 incident).
- **Updated Chapter 16**: Alpaca Architecture — added Section 16.3 (position_intent on all 6 sell paths) + 16.4 (EXIT path cancel related_orders) Apr 29 fixes
- **Updated Chapter 17**: Boba Decision Cycle — replaced with v3: Protocol A/B prompt, T0–T4 ladder, schema v2 journal fields (protocol + entry_criteria + brief_context), defensive `or ""` loaders, item 8 BRIEF CONTEXT CITED in reasoning checklist
- **Updated Chapter 18**: Build & Workflow Rules — added React JSX/useEffect verification (18.3, 18.4), no-restore + no-mock-data + never-delete rules consolidated (18.6, 18.7)
- **Removed**: Nothing — Bible v2 verbatim file inventories (chapters 3–15) preserved in PDF, summarized here in TOC. Bible v3 is the merge, not a replacement.
- **Bible system bootstrapped**: BIBLE.md at ~/.openclaw/data/, helpers bible-log/show/pdf at ~/scripts/bible/, GitHub repo itsjuggalo/missionctrl-bible (public, README = BIBLE.md)

### 2026-04-30 16:34 ET - Bible system online
- Change: Auto-update loop tested
- Details: Helpers + GitHub repo wired Apr 30

### 2026-04-30 17:22 ET - Layer 2/3 enforcement added
- Change: Memory entries 11+12 updated for bible system + auto-link gotcha; user preferences template provided
- Details: Session bootstrap command: bible-show | head -200 ; bible-show April 28 ; bible-show Active Backlog

### 2026-04-30 21:18 ET - Snapshot Poster + Grading Loop Shipped
- Change: Built and shipped 3 systems: bible v3 with auto-push, weekly grader posting Grade D to BobaTrades with 5 proposed edits, EVO conservative replay validated +$3K/week edge, account snapshot poster posting all 3 Alpaca accounts (R1/R2/JazzyHazzy R1) to #alpaca-account every 30 min with OCC decoded symbols + 7d/30d perf vs SPY benchmark.
- Details: Snapshot at /home/ubuntu/scripts/snapshots/account_snapshot_post_py — Cron */30 * * * *. Grader at /home/ubuntu/scripts/weekly_grade/weekly_grade_py — Cron 0 20 * * 5 (Fri 4pm ET). JazzyHazzy R1 account live at PA38IUKNR237 ($10K, L3, 4x margin) but jazzy_decision_cycle.py forked-but-not-wired pending future decision. SPY 7d=+2.07%, 30d=+13.34% benchmarks. Webhook UA fix needed for Cloudflare 1010 block on raw urllib calls.

### 2026-04-30 21:40 ET - JazzyHazzy Live on PA38IUKNR237
- Change: Forked Boba decision cycle, swapped Anthropic Claude Sonnet for OpenAI GPT-4o-mini, locked to PA38IUKNR237 $10K paper R1 conservative book, injected HARD GATE prohibiting <14 DTE without 3-condition override (flow score ≥90 + Kronos 2x breakeven + named catalyst), separate journal at jazzy_decisions_validated, killswitch at jazzy_killswitch, identity prompt rewrites JazzyHazzy as peer agent to Boba.
- Details: Cycle script jazzy_decision_cycle in /home/ubuntu/scripts/, cron matches Boba exactly: 11/13:30/14/15/16/17:30/19/20 ET weekdays + every 4hr crypto. Boba R2 untouched (md5 fe4881e71393bfca963310e0d1361a04 verified throughout). First live cycle fires tomorrow 11:00 AM ET. JazzyHazzy account: PA38IUKNR237 $10K equity, 4x margin = $40K BP, L3 options approved. Goes live immediately with no killswitch.

### 2026-04-30 21:50 ET - Grader Extended for Head-to-Head
- Change: Updated weekly grader to include JazzyHazzy alongside Boba R2 and R1 hold baseline. Loads jazzy_decisions journal, fetches PA38IUKNR237 P&L, computes jazzy_minus_r2 and jazzy_minus_spy deltas, rewrote grading prompt to compare two trading agents head-to-head with model/risk-profile context (Claude Sonnet aggressive vs GPT-4o-mini conservative), updated Discord digest format. JazzyHazzy reports no data this run since account just went live ~30min ago — by next Friday she'll have full week of decisions for first real comparison.
- Details: Grader at /home/ubuntu/scripts/weekly_grade/weekly_grade_py — 9 surgical edits via Python string replace with assertions, all verified before write. Backup at /tmp/weekly_grade_py_backup_*. Live test posted Grade for 2026-05-01 to BobaTrades. Boba R2 file md5 fe4881e71393bfca963310e0d1361a04 still verified unchanged. Webhook 403 warning at top of script is pre-existing non-blocking quirk — actual digest still posts. Next Friday cron 0 20 * * 5 will fire automatically with full JazzyHazzy data.
