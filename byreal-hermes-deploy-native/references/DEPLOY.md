# Hermes on RealClaw (Native API)

Spawn a **Hermes** Telegram agent on the same host as RealClaw, sharing its brain — every skill, the full memory, the same wallet, the same safety contract. RealClaw is the parent; Hermes is the Telegram-side child. Uses RealClaw's built-in LLM API; no external API key needed.

## You only need ONE thing

**A new Telegram Bot Token** (not the one RealClaw uses):

1. Open Telegram, search for `@BotFather`
2. Send `/newbot`
3. Name it something like "MyHermes"
4. Copy the token

> Install will hard-fail if the token matches RealClaw's bot — two pollers on the same bot break both.

## Install

In your RealClaw chat, say:

> Install Hermes

The agent will:
1. Ask for your new TG Bot Token
2. Auto-read RealClaw's API config (`~/.openclaw/openclaw.json`)
3. Install and start Hermes
4. **Share RealClaw's knowledge with Hermes** (profile, safety contract, recent memory, operational skills)
5. Your RealClaw bot is unaffected

## What Hermes Gets

Hermes is RealClaw on Telegram — same host, same wallet, same brain. Default is **inherit everything**.

| From RealClaw | Shared |
|---|---|
| `AGENTS.md` | Safety contract, red lines, permission model — verbatim |
| `TOOLS.md` | Tool routing + chain reference — verbatim |
| `USER.md` | Profile, wallets, risk profile, preferences |
| `SOUL.md` | Communication style + Hermes identity block appended |
| `MEMORY.md` | Long-term curated facts |
| `memory/*.md` | Full per-day daily logs (no time window) |
| `skills/*` | All skills — onboarding, tier-switch, watchdog, all operational skills |

## What Hermes Does NOT Get

| Not shared | Why |
|---|---|
| `byreal-hermes-deploy-native` skill | Recursion guard — Hermes shouldn't deploy itself |
| `BOOT.md` | CLI deps are global npm installs; BOOT is the OpenClaw runtime hook, not the agent runtime |
| `hooks/` | OpenClaw runtime hooks (skill-updater) belong to RealClaw, not Hermes |
| `configs/<strategy>/state.json` | Strategy crons run in main session — running the same cron in Hermes would race the atomic state writes. Hermes can read state to answer questions, just not write it. |

**On-chain capability is identical.** Hermes signs through the same Privy wallet, runs the same skills, follows the same AGENTS.md. The only line it doesn't cross is registering strategy crons / writing strategy state — that lives in RealClaw main session.

## After RealClaw Restart

```bash
nohup bash ~/.openclaw/hermes/start.sh >/dev/null 2>&1 & disown
```

## Sync Knowledge

When RealClaw learns something new — or the proxy IP / API key rotates — say:

> sync hermes

This re-reads `openclaw.json`, rewrites `config.yaml`, refreshes shared files, and restarts the gateway.

## Verify

```bash
# Check Hermes is running
pgrep -f hermes_cli.main

# Check Telegram connected
tail -5 ~/.openclaw/hermes/logs/gateway.log

# Check knowledge files
ls ~/.openclaw/hermes/*.md
```

## File Layout

```
~/.openclaw/hermes/
  .env                    # TG Bot Token only (mode 0600)
  config.yaml             # RealClaw's internal API proxy (mode 0600, written from openclaw.json)
  start.sh                # Startup script
  SOUL.md                 # From RealClaw + appended Hermes identity block
  USER.md                 # From RealClaw (profile + wallets via agent-token wallet-info)
  AGENTS.md               # From RealClaw (safety contract)
  TOOLS.md                # From RealClaw (tool reference)
  gateway.pid
  logs/gateway.log
  memory/                 # Last 7 days of per-day memory (no MEMORY.md)
  skills/                 # Operational skills only (see exclusion list above)
  hermes-agent/
    .env -> ../.env
    venv/
~/.hermes -> ~/.openclaw/hermes/
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Conflict: terminated by other getUpdates` | Same bot token as RealClaw | Create a new bot via @BotFather, re-run install |
| HTTP 401 on aux services | Hermes has no RealClaw session cookie (by design) | Use public APIs (CoinGecko / Jupiter / DexScreener) instead of the proxy |
| HTTP 401 on `/v1/messages` | API key rotated | `sync hermes` |
| HTTP 404 | `base_url` ends in `/v1` | `sync hermes` (rewrites config.yaml) |
| Connection refused to proxy | Proxy IP changed across pods | `sync hermes` |
| Hermes asks "what's your wallet?" | `agent-token wallet-info` failed during install | Re-run install after fixing wallet access |
