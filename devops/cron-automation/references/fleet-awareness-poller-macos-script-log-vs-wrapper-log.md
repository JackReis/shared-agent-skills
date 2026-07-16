# Fleet awareness poller: macOS script log vs wrapper log (2026-05-10)

Scenario: scheduled read-only fleet awareness poller on macOS, invoked by cron every 5 minutes:

```cron
*/5 * * * * /usr/bin/lockf -t 0 /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.lock /bin/bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh >> /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.log 2>&1
```

Key observations:

- The wrapper redirect log under `~/.hermes/cubby/runs/fleet-awareness-poller.log` can be stale and contain old failures (`flock: command not found`, old `md5sum` errors, macOS TCC `Operation not permitted`).
- The meaningful current poll output is written by the script itself to `~/.hermes/logs/fleet-awareness.log`, with state in `~/.hermes/logs/.fleet-awareness-state`.
- A manual script run exited 0 and produced fresh entries in the script log/state even while the wrapper log's mtime remained old.
- The script prunes the main log to the last 1000 lines before appending, so `wc -l ~/.hermes/logs/fleet-awareness.log` can stay constant across successful runs. Verify by tailing for a fresh timestamp and checking `.fleet-awareness-state`, not by requiring line-count growth.
- In the interactive agent runtime, `session-handoffs` and `Coordination/README.md` can be readable (`256` handoffs, README `231` lines), while cron sometimes logs them as not readable. Treat that as cron/TCC health evidence, not as a content delta.

Verification recipe:

1. Probe runtime first: `pwd`, `$HOME`, `id -un`, and whether the script exists.
2. Run `bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh` once manually.
3. Check fresh timestamps/mtimes for both:
   - `~/.hermes/logs/fleet-awareness.log`
   - `~/.hermes/logs/.fleet-awareness-state`
   Do not rely on `wc -l` growth because the poller trims the log to a fixed tail before appending.
4. Tail the script log for a current timestamp and compare the state hashes/counts to the current filesystem when deciding whether anything changed.
5. Only use `~/.hermes/cubby/runs/fleet-awareness-poller.log` to diagnose cron wrapper failures; do not treat old wrapper errors as proof the current script is failing.
6. If final delivery is configured by the cron wrapper, do not call messaging tools; report findings in the final response or `[SILENT]` when no actionable change exists.
