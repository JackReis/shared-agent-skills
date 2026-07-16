# Fleet heartbeat script path drift

Session: 2026-05-25

## Problem
The hourly heartbeat cron entry (`0 * * * *`) was executing `/Users/jack.reis/heartbeat-update.sh`, but the actual script lived at `/Users/jack.reis/.hermes/cubby/scripts/heartbeat-update.sh`. The crontab had drifted from the canonical location.

## Evidence
- `crontab -l` reported: `0 * * * * /usr/bin/lockf -t 0 /tmp/hermes_heartbeat.lock /bin/bash /Users/jack.reis/heartbeat-update.sh`
- `ls -la ~/.hermes/cubby/scripts/heartbeat-update.sh` confirmed the script existed and was executable.
- The wrapper log (`~/.hermes/cubby/logs/cron.log`) showed repeated `/bin/bash: /Users/jack.reis/heartbeat-update.sh: No such file or directory` for every hourly tick.
- The script's own log (`~/.hermes/logs/heartbeat.log`) showed the last successful run was from a *prior* `09:28` manual run, not from cron.

## Fix
1. Ran the script manually to verify it worked:   `HOME=/Users/jack.reis /usr/bin/lockf -t 0 ... /bin/bash ~/.hermes/cubby/scripts/heartbeat-update.sh`
2. Confirmed script-owned log updated with fresh `2026-05-25T10:30:38-05:00` timestamps.
3. Replaced the crontab with a clean single entry pointing to the correct absolute path, setting `HOME=/Users/jack.reis` explicitly for `set -u` compatibility.
4. Verified `crontab -l` after install to ensure no duplicate stale entries remained.

## Rules
- **When manually running a script that matches a crontab entry, always cross-check `crontab -l`.** If the scheduled path differs from the actual location, the cron job has been silently failing.
- **Do not assume the script's own log proves cron is healthy.** A script log may contain entries from *manual* runs. Check the wrapper/redirect log (`cron.log`) for `No such file or directory` or other cron-specific errors.
- **Read the full redirect log**, not just the script-owned log. The wrapper log is where cron-environment path errors appear.
- **When rewriting a crontab, write a complete clean file and verify `crontab -l` after install.** Partial appends or edits layered on top of an existing file can produce duplicate entries.
