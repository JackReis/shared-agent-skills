# macOS cron locking and LaunchAgent TCC pitfalls

Session: 2026-05-10 fleet awareness poller.

## What happened
- The existing cron entry looked valid but used bare `flock`:
  ```cron
  */5 * * * * flock -n /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.lock bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh >> /Users/jack.reis/.hermes/cubby/runs/fleet-awareness-poller.log 2>&1
  ```
- On macOS, `/bin/sh` cron reported repeatedly:
  ```text
  /bin/sh: flock: command not found
  ```
- `command -v lockf` returned `/usr/bin/lockf`; `command -v flock` was empty.
- A direct manual run of the script succeeded and updated `~/.hermes/logs/fleet-awareness.log`, proving the script itself was usable.
- Attempts to install a corrected crontab from this Hermes runtime (`crontab file`, `crontab -`, and `crontab -e` with an editor shim) hung, even though `crontab -l` worked and `/usr/sbin/cron` was running.
- A temporary LaunchAgent fallback using `/usr/bin/lockf` started, but failed with:
  ```text
  /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh: line 105: /Users/jack.reis/Documents/Coordination/README.md: Operation not permitted
  ```
  This is consistent with macOS privacy/TCC differences for launchd jobs accessing `~/Documents`.

## Future pattern
1. On macOS, do not assume `flock`. Probe both:
   ```bash
   command -v flock || true
   command -v lockf || true
   ```
2. Prefer this cron shape on macOS:
   ```cron
   */5 * * * * /usr/bin/lockf -t 0 /path/to/job.lock /bin/bash /path/to/job.sh >> /path/to/job.log 2>&1
   ```
3. Verify by checking the redirected cron log for absence of `flock: command not found` and the script's own log/state file for a fresh timestamp.
4. If `crontab <file>` / `crontab -` times out, treat the edit as failed until `crontab -l` shows the new `lockf` line. A direct `bash script.sh` run can update the script's own log while the recurring cron entry remains broken.
5. For cron jobs with automatic final delivery and a `[SILENT]` option, do not suppress scheduler failures as “nothing new.” Fleet content may be unchanged, but repeated `flock: command not found` or an unchanged broken crontab is an operational finding to report.
6. Be cautious with LaunchAgent fallbacks for scripts that read `~/Documents`; they may need Full Disk Access or a different storage location. If a LaunchAgent fallback fails, unload/remove it and avoid leaving a broken recurring job.
7. If a failed fallback corrupts state or emits a partial poll, restore the state from the last known good complete poll before rerunning manually.

## Verification commands used
```bash
crontab -l | grep -F 'poll-fleet-awareness.sh'
tail -n 8 ~/.hermes/cubby/runs/fleet-awareness-poller.log
command -v lockf || true
command -v flock || true
bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
stat -f 'awareness_log_mtime=%Sm awareness_log_size=%z' -t '%Y-%m-%dT%H:%M:%S%z' ~/.hermes/logs/fleet-awareness.log
```
