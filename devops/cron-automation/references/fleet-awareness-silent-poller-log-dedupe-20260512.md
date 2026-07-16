# Fleet Awareness Silent Poller: Log Dedupe and Manual-Run Artifacts (2026-05-12)

## Context
A scheduled Hermes cron task body asked to run:

```bash
bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
```

Delivery instructions said this was silent monitoring only: capture logs, do not notify Discord, and return `[SILENT]` when there is no genuinely new material finding.

## Evidence Pattern
- The requested script existed and direct execution exited 0.
- The recurring crontab entry was present with `/usr/bin/lockf -t 0 ... poll-fleet-awareness.sh >> ... 2>&1`.
- Script-owned artifacts were authoritative:
  - `~/.hermes/logs/fleet-awareness.log` had a fresh poll timestamp.
  - `~/.hermes/logs/.fleet-awareness-state` had a matching `last_check`.
- The latest direct/manual poll logged `Presence changed since last check` and `Git status changed since last check`, but direct inspection showed the same two active agents, the same handoff count, stable Coordination README metadata, and the same 9 changed Hermes paths.
- Earlier log entries had reported `New handoffs detected`, but those were historical entries already captured by prior polls, not new findings from the current cron body.

## Reusable Rule
For silent fleet-awareness cron bodies, distinguish **current material changes** from **log/state artifacts**:

1. Run the requested poller once and verify exit 0.
2. Check script-owned log/state freshness, not only wrapper logs.
3. If the current run emits change markers, directly inspect the underlying watched sources before reporting.
4. Treat permission-context flips, hash-baseline changes, or manual-run state updates as monitor artifacts when direct source inspection shows no real fleet change.
5. Do not report historical `New handoffs detected` lines merely because they are still in the retained log tail. Report only changes that are new for this run or still actionable.
6. If everything verifies and no material change remains, return exactly `[SILENT]`.
