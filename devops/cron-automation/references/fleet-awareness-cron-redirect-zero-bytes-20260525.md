# Fleet awareness poller: cron redirect log is 0 bytes while script-owned log is healthy (2026-05-25)

## Scenario

User asked to "run the fleet awareness poller script every 5 minutes." On
investigation, a cron entry already exists on macOS:

```cron
*/5 * * * * PATH=/Library/Developer/CommandLineTools/usr/bin:/usr/local/bin:/usr/bin:/bin \
  /usr/bin/lockf -t 0 /Users/jack.reis/.hermes/cubby/logs/.fleet-awareness.lock \
  /bin/bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh \
  >> /Users/jack.reis/.hermes/cubby/logs/fleet_awareness_cron.log 2>&1
```

The script is self-contained, read-only, and writes all output via its own
internal `LOG_FILE` variable:

```bash
LOG_FILE="$HOME/.hermes/cubby/logs/fleet_awareness.log"
log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $*" >> "$LOG_FILE"; }
```

## Observation

- The cron redirect log (`fleet_awareness_cron.log`) was **exactly 0 bytes** and
  last modified at May 25 03:24 — days before manual verification. It had never
  accumulated any content.
- The script-owned log (`fleet_awareness.log`) was **~105 KB**, actively updated
  every 5 minutes (`:00`, `:05`, `:10`, etc.), with complete presence/handoffs/
  README/git entries at each timestamp.
- Previous redirect logs (`fleet-awareness.log` from May 15, `fleet_cron_wrapper.log`
  from May 16, `fleet_poller_daemon.log` from May 17) showed similar patterns of
  abandonment after transient stderr emissions.

## Root Cause

The script's `log()` function redirects every bit of output to its own
`LOG_FILE`; it never writes to stdout or stderr. When cron appends stdout/stderr
via `>>`, there is nothing to capture. The redirect file is created (because the
shell opens it) but never grows. Over time it sits at 0 bytes, indistinguishable
from a job that has never run.

## Danger

A 0-byte redirect log is even more misleading than a stale log with old errors:
both a broken job (never started) and a perfectly healthy job (no stdout/stderr)
produce the same artifact. Without checking the script-owned log, an agent may
conclude cron is not firing at all.

## Verification Recipe

1. Check the redirect file:
   ```bash
   stat ~/.hermes/cubby/logs/fleet_awareness_cron.log
   wc -c ~/.hermes/cubby/logs/fleet_awareness_cron.log
   ```
2. Check the script-owned log for fresh timestamps:
   ```bash
   tail -5 ~/.hermes/cubby/logs/fleet_awareness.log
   ```
3. If the redirect file is 0 bytes but the script log is fresh, the schedule
   is healthy. The redirect file's emptiness does not indicate failure — it
  merely reflects that the script has no stderr/stdout of its own.
4. Confirm `crontab -l` still shows the entry and `lockf` is present:
   ```bash
   crontab -l | grep fleet
   which lockf || ls /usr/bin/lockf
   ```

## Takeaway

When evaluating a silent monitor cron job, zero bytes in the redirect log is
**not** evidence of a broken schedule if the script's internal `LOG_FILE` is
fresh and the `crontab` entry is intact. Some scripts are deliberately silent
at the shell level by design. Always check the script-owned log before declaring
a cron job non-functional.
