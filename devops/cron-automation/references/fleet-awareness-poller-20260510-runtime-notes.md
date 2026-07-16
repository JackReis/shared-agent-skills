# Fleet awareness poller runtime notes (2026-05-10)

## What happened
- Live runtime initially reported:
  - `crontab` absent
  - `/root/.hermes/presence` missing
  - `session-handoffs/` missing
  - Coordination README missing
- The poller script existed at `/root/.hermes/cubby/scripts/poll-fleet-awareness.sh`.
- Installing `cron` via `apt-get` succeeded, but package post-install did not start the daemon because `policy-rc.d` blocked service startup in the container.
- Starting `/usr/sbin/cron` manually worked.
- A 5-minute cron entry with `flock -n` was installed and verified with `crontab -l`.
- A direct poll run produced the expected baseline output:
  - `Active agents: presence directory not found`
  - `Handoffs: session-handoffs directory not found`
  - `Coordination README: not found`
  - `Git status: no git repo found in candidate roots`
  - `Poll complete`
- The poller wrote state to `/root/.hermes/cubby/runs/fleet-awareness-poller.state.json`.
- The resulting state was all-null (`git`, `handoffs`, `presence`, `readme`) with a fresh timestamp, which should be treated as a baseline for an unmounted/unreachable coordination tree rather than proof of no changes.

## Practical takeaway
When a cron-based fleet monitor is requested in a container:
1. Probe `crontab` and `cron` first.
2. If cron is missing, install it if permitted.
3. If package installation doesn’t start the daemon, start `/usr/sbin/cron` manually.
4. Verify the job with both `crontab -l` and a fresh log/state update.
5. Treat repeated `... not found` lines plus all-null state as environment mismatch, not a content diff.
