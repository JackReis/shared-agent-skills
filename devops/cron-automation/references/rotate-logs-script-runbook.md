# Rotate Logs Script Runbook

## Scope
Use this for cron jobs that rotate/trim local log files, especially fleet maintenance tasks.

## Canonical command
- `bash ~/.hermes/cubby/scripts/rotate-logs.sh`

## Required behavior
1. Trim logs to a maximum of 5000 lines each (keep newest 5000, archive older lines).
2. Archive overflow into `~/.hermes/logs/archive/YYYY-MM/`.
3. Remove files in `~/.hermes/logs` older than 30 days.
4. Re-run idempotently to verify no file remains above threshold and no stale files remain beyond retention.

## Observability checks
- Verify archive directory exists: `~/.hermes/logs/archive/<current-YYYY-MM>/`.
- Verify no file remains above 5000 lines after rotation.
- Verify no stale `.log*` in `~/.hermes/logs` exceeds 30 days.

## Cron configuration pitfall
A common breakage pattern is configuring a shell-only maintenance task as an LLM-invoked job (`model`/`provider` path) instead of a local shell route.

Prefer:
- `no_agent: true` + explicit command, or
- `script`-based scheduling,
so the command runs locally without token consumption.

## Recovery note
If rotation fails in cron output:
1. confirm script exists and is executable,
2. confirm path is absolute and home-expanded (works in cron context),
3. rerun manually once from shell,
4. inspect cron job definition for agentic dispatch fields (`model`, `provider`, `no_agent`, `script`).