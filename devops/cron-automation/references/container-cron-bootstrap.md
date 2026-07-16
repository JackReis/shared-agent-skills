# Container Cron Bootstrap

## When this matters
Use this note when cron jobs are needed in a minimal Docker runtime that does not ship cron by default.

## Observed behavior
- `cron`, `crond`, and `crontab` may be absent initially.
- `apt-get install cron` can succeed, but package post-install hooks may not start the daemon because `policy-rc.d` blocks service startup.
- In that case, the cron daemon must be started manually with `/usr/sbin/cron`.

## Minimal verification
1. Confirm `crontab -l` works after installation.
2. Confirm the daemon is alive with `ps -ef | grep '[/]usr/sbin/cron'`.
3. Use absolute paths in cron entries.
4. Capture job output to a log file.
5. Use `flock` to prevent overlap.

## Example cron line
```cron
*/5 * * * * /usr/bin/flock -n /root/.hermes/cubby/runs/fleet-awareness-poller.lock /bin/bash /root/.hermes/cubby/scripts/poll-fleet-awareness.sh >> /root/.hermes/cubby/runs/fleet-awareness-poller.cron.log 2>&1
```