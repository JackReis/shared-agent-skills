# Fleet Awareness Cron Body: Manual Poll Verification (2026-05-11)

## Context
A scheduled Hermes cron invocation asked to run:

```bash
bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
```

The task body explicitly said this was silent monitoring only, with no Discord notifications. The script itself writes meaningful output to `~/.hermes/logs/fleet-awareness.log` and state to `~/.hermes/logs/.fleet-awareness-state`; stdout can be empty on success.

## Reusable Pattern
1. Treat the cron task body as a single poll execution, not a request to reinstall or rewrite the schedule unless scheduling is absent/broken.
2. Run the requested script directly and check exit status.
3. Inspect the script-owned log/state files for fresh timestamps; do not rely only on the wrapper redirection log under `~/.hermes/cubby/runs/`.
4. Optionally inspect `crontab -l` to confirm the recurring `*/5` entry and lock wrapper are present.
5. If the manual poll succeeded and no material fleet changes require user attention, return exactly `[SILENT]`.

## Evidence Shape From This Run
- Direct script execution exited 0 with no stdout beyond the surrounding probe.
- Script-owned log/state advanced to the current timestamp and showed normal poll completion.
- Wrapper log was stale and contained old errors, so it was not the authoritative signal for this run.

## Pitfall
A fresh script-owned state timestamp plus a stale wrapper log is not necessarily a failed poll. The wrapper log only captures stdout/stderr from the scheduler wrapper; a quiet successful script can update its own log/state without appending to the wrapper log.
