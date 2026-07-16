# Beads→Paperclip Dispatch Loop (2026-07-02)

## Context

The fleet has 144 open beads in the vault DB (`~/Documents/=notes/.beads`), but `bd ready` returns empty because all have blockers/dependencies. After `bd recompute-blocked`, ready beads become dispatchable. The dispatch loop triages, classifies, and dispatches them via Paperclip agent wakeups.

## Architecture

- **Beads** (Dolt remote `aegis`) = sole work-state ledger
- **Paperclip** (`127.0.0.1:3110` via SSH tunnel from Talaris) = agent wakeup trigger only (never holds task status)
- **Hermes cron** = hourly tick, `no_agent=True` (script-only, zero LLM tokens)
- **Wave limit** = 5 beads per tick (`BD_MAX_WAVE` env var, bumpable for faster clearing)

## Script

`~/.hermes/scripts/beads-paperclip-dispatch.py` — Python stdlib only (no curl — Tirith blocks it).

### Flow

1. `bd dolt pull --remote aegis && bd recompute-blocked` — sync + unblock
2. `bd ready -n 200` — get unblocked beads
3. Classify each: **A** (agent-deployable), **H** (human-gated), **D** (needs-decision)
   - H keywords: sudo, TCC, API key, credential, purchase, medical, invoice, BotFather, ratify, signature, human-only
   - D keywords: epic, decide, Evaluate paid, activate or retire, consolidation INDEX
4. Dispatch up to `MAX_WAVE` A-class beads: `bd update <id> --claim && bd comment <id> "AGENT CLAIMED"` then `POST /api/agents/<id>/wakeup?companyId=87c32b8e...`
5. D-class beads: `bd update <id> -s blocked && bd comment <id> "AGENT HUMAN HOLD: <reason>"`
6. `bd dolt push --remote aegis` — drain back to Aegis
7. Summary: dispatched count, human-gated checklist, deferred count

### Agent Assignment Heuristics

| Title keywords | Agent | Paperclip ID prefix |
|---|---|---|
| doc, review, audit, report, research, map, index | Aegis Claude Code | `f1ef5e14` |
| mr, merge, integrate, deploy, code, fix, patch | Aegis Codex | `0d32fe47` |
| skill, config, sync, scope | Fleet Routine Runner | `bb4a99d3` |
| (default) | Aegis Claude Code | `f1ef5e14` |

## Cron Job

```
cronjob(action='create', name='Beads→Paperclip hourly dispatch',
  schedule='every 1h', no_agent=True,
  script='cd ~/Documents/=notes && BEADS_DIR="$HOME/Documents/=notes/.beads" BD_MAX_WAVE=5 python3 ~/.hermes/scripts/beads-paperclip-dispatch.py 2>&1')
```

Job ID: `738f3798d377`

## Key Design Decisions

1. **Wave limit prevents flooding.** Dispatching all 121 deployable beads at once would overwhelm 13 agents. 5/hour = ~24 hours to clear, with receipts landing between waves for review.
2. **Paperclip is trigger-only.** No task status in Paperclip — beads is the sole ledger. Paperclip `wakeup` POST starts an agent run; the agent reads its assigned bead from bd.
3. **Human-gated items surface to Jack.** The script output lists H-class beads as a checklist — never dispatches them.
4. **D-class gets HUMAN HOLD comments.** Auto-commented so next human review sees the decision needed.
5. **Python stdlib only.** `urllib.request` for Paperclip API (Tirith blocks `curl`). `subprocess` for bd CLI.

## Verification

```bash
# Dry run (no mutations, no wakeups):
cd ~/Documents/=notes && BEADS_DIR="$HOME/Documents/=notes/.beads" BD_MAX_WAVE=5 python3 ~/.hermes/scripts/beads-paperclip-dispatch.py --dry-run

# Check cron job status:
cronjob(action='list') | grep "Beads"
```

## User Preference

Jack's directive: "Get the beads to ready and move them through the queue, or else remove them." This loop prevents stagnant backlog by continuously triaging and dispatching. Beads that are OBE or unresolvable should be closed, not left in the queue indefinitely.