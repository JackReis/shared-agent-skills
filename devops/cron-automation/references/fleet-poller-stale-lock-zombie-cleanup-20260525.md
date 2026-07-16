# Reference: Fleet Poller Stale Lock + Zombie Cleanup (2026-05-25)

## Context
Fleet awareness poller already scheduled in crontab at `*/5 * * * *` via `lockf -t 0` wrapper. The session initially attempted to start a competing `while true; sleep 300; …` background loop, creating a duplicate poller process.

## Stale Lock File
A file existed at `~/.hermes/cubby/logs/.fleet-awareness.lock` but it was **not** an active lock. On macOS, `lockf` uses BSD-style `flock(2)` on the file descriptor — the lock lives in the kernel and is released when the process exits. The file on disk is just an inode anchor. Even if a previous process died without releasing the lock, the lock is gone once the FD closes. Removing the stale file is safe and prevents transient `lockf` confusion with `-t 0` (non-blocking), though `-t 0` would normally return immediately even if locked.

**Takeaway:** Seeing a lock file on disk does not mean a lock is held. `lockf` cleans up its file on exit unless `-k` is used. Remove stale lock files when found, but do not rely on their absence as proof the poller is not running.

## Zombie / Defunct Process
After `kill`-ing the competing background loop (PID 41442), it persisted as `<defunct>` in `ps` with `STAT=Z`. A brief `sleep 3` allowed the parent process to reap it. `ps -p 41442` afterward showed `Zombie reaped ✓`.

**Detection pattern:**
```bash
ps -o pid,ppid,stat,args -p <PID>
# STAT=Z → zombie
```

**Cleanup pattern:**
```bash
kill -9 <PID>          # ensure SIGKILL sent
sleep 3                 # give parent time to reap
ps -p <PID>             # should return "PID: process ID out of range" or absent
```

## Prevention
Before starting any `while true; sleep 300;` poller loop, always:
1. `crontab -l | grep <script>` — if cron already handles the interval, do not create a loop.
2. `ps aux | grep <script> | grep -v grep | wc -l` — if count > 0, treat as duplicate risk.
3. Clean up competing processes and stale locks before claiming "done."

## Verification After Cleanup
- No `while true` poller processes remain (`ps aux`)
- Next cron tick produces a clean, single log entry at `~/.hermes/cubby/logs/fleet_awareness.log`
- The 5-minute interval aligns with cron schedule (timestamps ending in `:00`, `:05`, etc.)
- Script-owned log (not a wrapper redirect-only log) confirms operational health
