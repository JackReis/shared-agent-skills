# Fleet Awareness Poller: Manual Verification Overlap (2026-05-11)

## Context
A scheduled 5-minute fleet awareness cron job and a manual verification run both executed around the `01:30` boundary. The direct poll exited 0 and updated `~/.hermes/logs/fleet-awareness.log` plus `~/.hermes/logs/.fleet-awareness-state`, but the log showed duplicate same-timestamp poll headers and a wrapper-log artifact:

```text
mv: /Users/jack.reis/.hermes/logs/fleet-awareness.log.tmp: No such file or directory
```

## Lesson
When manually verifying a cron-managed monitor, do not run the underlying script naked if the scheduled tick may be active. Either:

1. Run the manual probe through the same lock wrapper used by cron, or
2. Wait until safely between schedule boundaries, or
3. Treat duplicate same-timestamp log lines/temp-file errors as overlap evidence, not as content-change evidence.

## Recommended Manual Probe Shape

First copy the active lock path from the scheduler entry rather than choosing a new one:

```bash
crontab -l | grep 'poll-fleet-awareness'
```

Then run through that exact lock path, e.g.:

```bash
/usr/bin/lockf -t 0 /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.lock \
  /bin/bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh
```

Using a different manual lock such as `~/.hermes/logs/fleet-awareness.lock` is equivalent to no overlap protection against the cron job, even though the manual command appears to be locked.

Then verify script-owned artifacts:

```bash
stat -f '%N size=%z mtime=%Sm' -t '%Y-%m-%dT%H:%M:%S%z' \
  "$HOME/.hermes/logs/fleet-awareness.log" \
  "$HOME/.hermes/logs/.fleet-awareness-state"
tail -n 30 "$HOME/.hermes/logs/fleet-awareness.log"
cat "$HOME/.hermes/logs/.fleet-awareness-state"
```

## Reporting Guidance
If the poll succeeded but overlap artifacts appear, report monitor-health nuance only when the cron task is expected to produce a report. For silent monitors, return `[SILENT]` only if there are no real content changes and no actionable scheduler/logging failure.