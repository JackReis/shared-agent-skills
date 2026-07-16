# Reference: fleet-awareness-poller-stale-cron-redirect-log-20260525
## Stale redirect logs when the script-owned log is healthy

**Symptom:** The crontab redirect log `~/.hermes/cubby/logs/fleet_awareness_cron.log` stopped at 2026-05-15 15:45 with 9-day-old git errors, while the script-owned log `~/.hermes/cubby/logs/fleet_awareness.log` showed continuous healthy polls every 5 minutes (including timestamps on 2026-05-25 03:15, 03:20, 03:23, etc.).

**Root cause:** The cron redirect path and the script's internal `LOG_FILE` path diverged silently:
- Cron redirected stdout/stderr to `fleet_awareness_cron.log`
- The script logged actual poll state to `fleet_awareness.log`
- A prior cron line attempted to run `flock` (not present on macOS) which silently failed, and the wrapper loop died; later manual runs succeeded but wrote to the script's own log
- The redirect log accumulated only old pre-fix errors and never got rotated or cleaned

**Impact:** The redirect log looked broken even though the poller was working. This mislead verification checks.

**Fix / verification recipe:**
1. Always check BOTH the redirect log and the script-owned log when verifying a cron job.
2. If redirect log is stale but script-owned log is fresh, the redirect path may be orphaned from a prior cron line or wrapper failure. Clean it by `rm` and let the next cron tick recreate if needed.
3. Prefer redirecting to a single canonical log file, or ensure both paths are inspected.

**Related:** `references/fleet-awareness-poller-macos-script-log-vs-wrapper-log.md` — same pattern on a prior date where script-owned log was fresh despite stale wrapper errors.
