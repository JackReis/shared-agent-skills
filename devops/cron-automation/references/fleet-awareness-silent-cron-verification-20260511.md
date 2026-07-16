# Fleet Awareness Silent Cron Verification — 2026-05-11

## Context
A scheduled task body asked to run `bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh` every 5 minutes as silent monitoring: capture logs, no Discord notifications, return `[SILENT]` when nothing material changed.

## Useful Pattern
1. Run the requested script once directly (or through the same cron lock when needed). For this script, stdout may be empty because it writes to `~/.hermes/logs/fleet-awareness.log` and `~/.hermes/logs/.fleet-awareness-state`.
2. Inspect the script-owned log and state file, not just the cron wrapper log.
3. Check `crontab -l` only to confirm the durable schedule when the task is recurring.
4. Treat old wrapper-log errors as stale if script-owned log/state have fresh timestamps and the crontab line has already been corrected.
5. For silent monitors, verify direct content health (presence files, handoff count, README line count, git status summary) before deciding whether to report or return `[SILENT]`.

## Evidence Shape From This Run
- Crontab already contained a macOS-safe `lockf` entry for the poller.
- Direct script run exited 0 with no stdout.
- `~/.hermes/logs/fleet-awareness.log` and `.fleet-awareness-state` updated at the fresh run timestamp.
- Wrapper log still contained stale historical `flock: command not found` / TCC errors; those were not current operational failures once script-owned artifacts were fresh.
- Visible state was stable: 2 presence files, 266 handoffs, Coordination README readable directly, Hermes repo with 9 changed paths.

## Pitfall
Avoid reporting stale wrapper-log failures for a silent monitor after a successful direct run and fresh script-owned state. The reportable event is a material fleet change or a current scheduler/logging failure, not historical wrapper noise.