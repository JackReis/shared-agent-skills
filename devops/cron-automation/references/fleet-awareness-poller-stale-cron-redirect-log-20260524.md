# Fleet awareness poller: stale cron redirect log masking healthy execution (2026-05-24)

## Scenario

A cron-driven fleet-awareness poll runs every 5 minutes on macOS:

```cron
*/5 * * * * /bin/bash /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh \
  >> /Users/jack.reis/.hermes/cubby/logs/fleet_awareness_cron.log 2>&1
```

The script is self-contained, read-only, and writes its own structured log to
`~/.hermes/cubby/logs/fleet_awareness.log` (lines are appended; log is pruned to
≈2,000 lines on each run).

## Observation

- The cron redirect log (`fleet_awareness_cron.log`) was last written **9 days prior** (May 15) and contained 1,352 lines of repeated `fatal: not a git repository` errors.
- The script-owned log (`fleet_awareness.log`) was **fresh**, with poll headers stamped at `:00`, `:05`, `:10`, `:15`, `:20`, etc. and fully populated presence/handoffs/README/git entries.
- The script exits `0`, cron fires on schedule, and `crontab -l` confirms the entry.
- Manual execution (`env -i HOME=/Users/jack.reis ... bash poll-fleet-awareness.sh`) produced **zero stderr** — the old git errors did not reproduce.

## Root Cause

The cron `>>` redirect file is merely a shell-level stderr/stdout capture. It can
become **permanently stale** if:
- the script itself redirects all meaningful output internally to its own `LOG_FILE`
- prior stderr emissions were from a transient runtime condition (e.g., a prior
  script version without `git -C "$VAULT"`, a different initial working directory,
  or a one-time environment mismatch)
- the current healthy execution produces **no stderr**, so nothing re-opens or
  refreshes the redirect file

## Danger

A human (or agent) inspecting only the cron redirect log will conclude the job is
failing, even when the script-owned log shows continuous healthy polls.

## Verification Recipe

1. Run the script once manually (prefer through the same shell/cron wrapper path):
   ```bash
   env -i HOME=/Users/jack.reis SHELL=/bin/bash PATH=/usr/local/bin:/usr/bin:/bin \
     bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh
   ```
2. **Check the SCRIPT-OWNED log for a fresh timestamp**:
   ```bash
   tail -5 ~/.hermes/cubby/logs/fleet_awareness.log
   ```
3. **Check the REDIRECT LOG's mtime and content**:
   ```bash
   stat ~/.hermes/cubby/logs/fleet_awareness_cron.log
   tail -5 ~/.hermes/cubby/logs/fleet_awareness_cron.log
   ```
4. If the redirect log is stale and the script log is fresh, **treat the script
   log as authoritative**. The redirect log's old errors are historical noise.
5. Only report the redirect log's errors as actionable if they **also appear**
   in the script log or in a freshly reproduced manual run.

## Takeaway

When a cron redirect log contains only old stderr and the script-owned log/state
are current, do **not** let stale redirect noise invalidate a healthy schedule.
The script's internal `LOG_FILE` / `STATE_FILE` is the truth; the wrapper redirect
is secondary at best and misleading at worst.
