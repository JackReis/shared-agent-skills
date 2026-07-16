# Case Study: Stale Overnight Dev Config Revival — 83% Path Drift

**Date:** 2026-05-27
**Session:** openclaw-revive-overnight-schedule
**Skill:** `cron-automation`
**Agent:** Wings (Hermes)

## What happened

The user asked to schedule overnight dev workflows for "sugar cleanup + semantic search." The agent discovered `.overnight-dev.json` (Jan 2026, 5 months old) with 6 referenced scripts. Only 1 of 6 paths still existed:

| Referenced Path | Status |
|-----------------|--------|
| `claude/agents/transition/markdown-formatter.py` | MISSING — directory empty |
| `cleanup_vault_duplicates.py` | MISSING |
| `000-INTEGRATION-HUB.md` | MISSING |
| `test_deduplication.py` | EXISTS |
| `semantic_search/` | MISSING |
| `vault_organizer/` | MISSING |

The old config was effectively a no-op — it would fail silently every night.

## Root cause

Overnight dev configs are write-once, run-forever. Nobody validates that referenced paths still exist. After 5 months of file moves, renames, and directory restructuring, the config became 83% stale.

## Corrective action

1. **Abandon the stale config.** Do not try to repair it — the world has moved on.
2. **Write a fresh config** with ONLY paths that exist today.
3. **Add a path-validation runner** that checks every referenced path before executing, reports valid/missing counts, and returns non-zero if any are missing.
4. **Schedule the runner**, not the raw scripts.

## The runner pattern

```python
# overnight-dev-runner.py
# 1. Load config JSON
# 2. For each script: check os.path.exists()
# 3. Report: N valid, M missing
# 4. If M > 0: exit 2 (cron sees failure, logs it)
# 5. If N > 0: execute each valid script in sequence
# 6. Write markdown report to memory/ with timestamp
```

This prevents silent no-ops. A cron job that runs every night and does nothing because its targets are gone is worse than no cron job at all — it creates log noise and false confidence.

## Lesson

**Scheduled automation decays.** Every cron job, overnight runner, or scheduled workflow that references file paths is a liability. Before scheduling (or reviving an old schedule):
1. Validate every referenced path
2. Count valid vs missing
3. If missing > 0, fix the config before scheduling
4. Build validation into the runner itself so future drift is caught

## Verdict

Agent correctly identified stale config, wrote fresh config with validation, and scheduled a runner script instead of raw commands. Pattern captured in `cron-automation` SKILL.md.

## Skill update

Patched into `cron-automation` SKILL.md under a new pitfall: "Overnight dev / scheduled automation path drift — stale configs with missing paths running silently every night."
