# Fleet Awareness Poller Notes

Session-derived reference for recurring fleet monitoring.

## Problem Observed
A shell-launched background loop (`terminal(background=true)` / `while true; do ...; sleep 300; done`) was not reliable in this environment; the process exited instead of persisting.

## Working Approach
Use cron with a lockfile and log redirection:

```cron
*/5 * * * * flock -n /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.lock bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh >> /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.log 2>&1
```

## Verification
- `crontab -l` should show the entry
- `bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh` should run cleanly once
- `tail` the log file to confirm output

## Environment Quirk
The terminal backend defaulted to a missing home directory until the workdir was pinned to `/`. If a command unexpectedly errors with a `cd` failure, explicitly set `workdir: "/"` before retrying.

## Monitoring Targets
The poller script checks:
- `~/.hermes/presence/`
- `session-handoffs/`
- Coordination README changes
- Git status changes
