# Cron Broken Delivery → No-Agent Conversion

**Date**: 2026-07-14
**Job**: `obn-morning-brief` (`da8ad7b5c684`)

## Problem

The `obn-morning-brief` cron job was LLM-driven (no `no_agent` flag). Its script `obn_telegram_brief.py` attempted delivery via `kimi mcp telegram-bridge send_summary` — a subcommand that never existed in the installed Kimi CLI (v1.47.0). Every morning the agent:

1. Ran the brief generator (worked fine)
2. Tried the Kimi delivery path (always failed)
3. Manually worked around it: SOPS-decrypted `HERMES_TELEGRAM_BOT_TOKEN` from `~/.secrets/hermes-runtime.env`, sent via curl to Telegram Bot API, archived the brief, recorded delivery in Hindsight

This burned a full LLM session every tick on a deterministic task that should cost zero tokens.

## Fix

### 1. Create a no-agent shell wrapper

```bash
#!/usr/bin/env bash
# ~/.hermes/scripts/obn-morning-brief.sh
set -euo pipefail

VAULT_PYTHON="/Users/jack.reis/Documents/=notes/.venv/bin/python3"
BRIEF_SCRIPT="/Users/jack.reis/Documents/=notes/claude/scheduled-tasks/vault-ingest/obn_morning_brief.py"
BRIEFS_DIR="/Users/jack.reis/Documents/=notes/claude/scheduled-briefs"
TODAY=$(date +%Y%m%d)

mkdir -p "$BRIEFS_DIR"
SAVE_PATH="$BRIEFS_DIR/${TODAY}-morning.md"

# Run once, capture stdout for both delivery and archival
BRIEF_OUTPUT=$("$VAULT_PYTHON" "$BRIEF_SCRIPT")
echo "$BRIEF_OUTPUT"

# Archive
echo "$BRIEF_OUTPUT" > "$SAVE_PATH"
```

Key: run the generator once, capture stdout, echo it (cron delivers it), then archive to file. Don't run the script twice (once for stdout, once for `--save`) — that duplicates the ChromaDB queries.

### 2. Update cron job to no-agent + native delivery

```
cronjob(action='update', job_id='da8ad7b5c684',
    no_agent=true,
    script='obn-morning-brief.sh',
    deliver='telegram:7618822262',
    prompt='Script-only OBn morning brief. Run the attached script and deliver its stdout verbatim to Telegram.')
```

The `deliver: telegram:` mechanism is built into Hermes cron — no external CLI or token management needed.

### 3. Patch the source script's delivery function

Replace the non-existent `kimi mcp telegram-bridge send_summary` call in `obn_telegram_brief.py` with a direct Bot API fallback using `urllib.request` + SOPS decryption. This keeps the Python script usable for manual runs outside cron.

## General Pattern

When a cron job's delivery depends on an external CLI subcommand that may not exist:
1. **Don't fix the external CLI** — it's a fragile dependency
2. **Use Hermes cron's native `deliver:` field** — it handles Telegram/Discord/origin delivery without any script-level delivery code
3. **Convert to `no_agent: true`** — if the job is deterministic (generates content from local data), it doesn't need an LLM
4. **Patch the source script** to remove the broken dependency for manual use, but the cron path should bypass it entirely

## Verification

- `cronjob(action='run', job_id='da8ad7b5c684')` → `last_status: "ok"`, `execution_success: true`
- Brief arrives in Telegram via native cron delivery
- Archived copy at `claude/scheduled-briefs/YYYYMMDD-morning.md`
- Zero LLM tokens consumed (no_agent mode)