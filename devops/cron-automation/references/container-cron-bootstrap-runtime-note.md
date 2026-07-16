# Container Cron Bootstrap Runtime Note

Session-specific verification from a Debian 13 container.

## What happened
- `crontab` was initially unavailable.
- `apt-get install -y cron` succeeded.
- Package startup hooks were blocked by `policy-rc.d`, so the daemon did **not** start automatically.
- Starting `/usr/sbin/cron` manually brought the daemon up.
- The scheduled fleet poller was installed with absolute paths and a `flock` lockfile.

## Verified commands
- `crontab -l` shows the scheduled entry.
- `ps -ef | grep '[/]usr/sbin/cron'` confirms the daemon is running after manual start.
- Log path: `/root/.hermes/cubby/runs/fleet-awareness-poller.cron.log`
- State path: `/root/.hermes/cubby/runs/fleet-awareness-poller.state.json`

## Observed baseline output
- Presence directory: not found
- Session handoffs: not found
- Coordination README: not found
- Git repo: not found in candidate roots

## Practical takeaway
In minimal Docker runtimes, treat cron as a package-and-daemon problem, not just a crontab problem:
1. install `cron`
2. verify `crontab`
3. manually start `/usr/sbin/cron` if package hooks are blocked
4. verify the daemon with `ps`
5. confirm the job by tailing the log
