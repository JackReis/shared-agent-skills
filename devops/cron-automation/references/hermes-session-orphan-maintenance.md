# Hermes Session-Orphan Maintenance as a No-Agent Job

Use this pattern when Hermes state growth is driven by old `source=tool` sessions whose `ended_at` remains null, causing normal retention to skip them.

## Diagnosis

1. Measure state DB size and `hermes sessions stats`.
2. Query sessions by source, age, and `ended_at` state.
3. Compare SQL candidate counts with `hermes sessions prune --dry-run` where supported.
4. A large discrepancy can mean rows are still marked active; normal prune intentionally refuses them.
5. Cross-check candidate IDs against the gateway session registry. Zero overlap is required before finalization.
6. Confirm the leak is historical or identify the still-producing execution path; do not merely delete evidence of an active bug.

## One-time repair

- Finalize only old, open, registry-absent `source=tool` rows.
- Use Hermes's canonical `SessionDB.end_session(id, "orphaned_tool")` method rather than raw SQL updates.
- Close the DB, then use `hermes sessions prune --source tool --older-than <days> --yes`.
- Prune cron history separately with a more conservative window.
- Preserve Telegram, CLI, subagent, and live registry sessions.
- Run SQLite integrity/foreign-key checks; perform optimize/VACUUM only in a controlled low-write or stopped-gateway window.

## Permanent no-agent guard

The deterministic script should:

- fail closed if `sessions.json` is missing, unreadable, or malformed;
- recursively collect registered session IDs and exclude all of them;
- select only open tool sessions older than a conservative cutoff;
- finalize through `SessionDB.end_session()`;
- delegate deletion to the supported Hermes CLI;
- emit empty stdout on a healthy no-op;
- emit structured JSON only when it repaired state;
- exit non-zero on any uncertainty so cron alerts instead of silently acting.

Version and test the implementation outside `~/.hermes/scripts`, install exact bytes as a regular contained script, verify SHA equality across hosts, run direct no-op canaries, then create one `no_agent=true` job per independent Hermes state store.

## Verification

- Unit-test registry fail-closed behavior and candidate exclusion.
- Test the exact CLI argument shape against old/new Hermes versions; bare integer day values are broadly compatible.
- Direct canary: exit 0 and zero stdout bytes when no work remains.
- Scheduler canary: verify `last_run_at`, `last_status`, and delivery separately.
