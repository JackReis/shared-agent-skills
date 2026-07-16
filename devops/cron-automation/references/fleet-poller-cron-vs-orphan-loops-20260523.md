# Fleet Poller: Cron vs Orphaned Background Loop Conflict

**Date:** 2026-05-23
**Context:** macOS crontab with `/usr/bin/lockf`; 3 orphaned `bash -lic` while-true processes dating back to Monday competed with the cron job.

## Problem

Crontab had a correct entry:
```
*/5 * * * * /usr/bin/lockf -t 0 ~/.../fleet_awareness.lock /bin/bash ~/.../poll-fleet-awareness.sh
```

Yet the poll log showed **duplicate entries at the same second**, e.g.:
```
2026-05-23 17:18:10 poll complete
2026-05-23 17:18:10 poll complete
```

Root cause: Three orphaned `bash -lic "while true; do ... sleep 300 ... done"` processes (PIDs 55101, 55103, 85607) had been running since Monday (`Mon12AM`, `Mon01AM`), independent of cron. They survive because `bash -lic` opens a login shell, and the `while true` loop was backgrounded (the `/opt/homebrew/bin/bash -lic ...` process was a detached PPID=1).

## Why `lockf -t 0` Did Not Prevent This

`/usr/bin/lockf -t 0 <lockfile> <cmd>` on macOS only serializes invocations that *actually try to acquire that lockfile*. The orphaned while-true loops never ran via `lockf`; they ran the script directly, so the lock was irrelevant to them.

This differs from duplicate *cron* invocations, where `lockf` would block the second cron tick. Here, the competing processes were not cron-launched — they were long-lived background loops from a prior session.

## Detection

```bash
# Count ALL poller processes, not just cron-spawned
ps -ef | grep -i "poll-fleet-awareness" | grep -v grep | wc -l
# Should be 0 (or 1 if currently running)

# Check PPID and start time
ps -ef | grep -i "poll-fleet-awareness" | grep -v grep
# Sample output:
# 501 85607     1   0 Mon01AM ??  /opt/homebrew/bin/bash -lic set +m; while true; do bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh; sleep 300; done
#  ^^^^^ PPID=1 = orphaned, "Mon01AM" = started Monday, survived multiple days

# Also check if scheduler-managed job exists
hermes cron list 2>/dev/null | grep "fleet-aware"
crontab -l | grep "fleet-aware"
```

## Resolution

1. **Kill all competing processes** (including orphaned loops):
```bash
# Graceful first
pgrep -f "poll-fleet-awareness" | xargs kill 2>/dev/null
sleep 2
# Force kill if remaining
pgrep -f "while true.*poll-fleet" | xargs kill -9 2>/dev/null
```

2. **Verify only cron remains**:
```bash
ps -ef | grep -i "poll-fleet" | grep -v grep
# Should return empty
pgrep -f "poll-fleet" || echo "none remaining"
```

3. **Let cron handle future scheduling** (do not start a new while-true loop).

## Key Insight: Detection Distinction

| Pattern | Detection Command | Root Cause |
|---------|------------------|------------|
| Duplicate *cron* invocations | `crontab -l` shows 1+ entries | No `lockf`/`flock` in cron line |
| Orphaned *background loops* | `ps -ef` shows `bash -lic while true` with PPID=1 | Prior session started a loop that survived disconnect |
| LaunchAgent duplicates | `launchctl list` + `ps -ef` | Misconfigured `KeepAlive` + `RunAtLoad` |
| Duplicate *lock* holders | Lock file + script running simultaneously | Lock implementation doesn't actually serialize |

## Prevention

Before starting any poller loop or cron job, always:
1. Check `crontab -l` for an existing schedule
2. Check `ps` for orphaned background loops (`bash -lic while true`)
3. Check `launchctl list` for LaunchAgents
4. If #1 exists, prefer it and *do not* start a loop
5. If #2 exists, kill them first, then rely on #1
6. If #3 exists, check if it's the intended mechanism

## Verification After Cleanup

Run the script once manually to confirm it works:
```bash
bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
```

Then check the log for a single, non-duplicate entry:
```bash
tail -5 ~/.hermes/cubby/logs/fleet_awareness.log
# Should show exactly one timestamp block
```

Wait for the next cron tick (up to 5 minutes) and confirm the log gains new non-duplicate entries:
```bash
tail -5 ~/.hermes/cubby/logs/fleet_awareness.log
```

## Related
- `fleet-poller-duplicate-instance-cleanup-20260517.md` — the first wave of duplicate-loop cleanup (12+ instances of the same `while-true` pattern)
- `macos-cron-locking-and-launchd-tcc.md` — macOS lockf specifics
