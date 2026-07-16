# Hermes Cron Subsystem: Script Paths, Duplicate Detection, and Silent Mode

Session: 2026-05-25 — Fleet Awareness Poller scheduling

## Discovery Context
User asked to run `poll-fleet-awareness.sh` every 5 minutes via the Hermes cron subsystem.

## Key Findings

### 1. CLI Subcommand Is `cron`, Not `cronjob`
`hermes cronjob create` fails with:
```
error: argument command: invalid choice: 'cronjob'
```
Correct usage: `hermes cron create ...`

### 2. `--script` Requires Filename Only, Resolved Under `~/.hermes/scripts/`
Passing an absolute or home-relative path to `--script` errors with:
```
Script path must be relative to ~/.hermes/scripts/.
```
Passing a symlink that points outside `~/.hermes/scripts/` errors with:
```
Script path escapes the scripts directory via traversal
```
**Fix:** Copy the script into `~/.hermes/scripts/` first, then pass just the filename.
```bash
cp /path/to/script.sh ~/.hermes/scripts/
hermes cron create "*/5 * * * *" --name my-job --script script.sh --no-agent
```

### 3. `--no-agent` Is Critical for Silent Monitors
Without `--no-agent`, each tick launches a full LLM session. This:
- Incurs token cost
- May generate unwanted notifications
- Defeats the purpose of silent read-only monitoring

Always add `--no-agent` for polls, log captures, and health checks.

### 4. Duplicate Detection Is Manual
`hermes cron list` showed an existing `fleet-awareness-poller` (`c5dc5751cd0f`) already at `every 5m`. Creating a new job without checking produced a duplicate. The correct flow:
```bash
hermes cron list                     # check for duplicates by name
# if duplicate exists and is stale:
hermes cron rm <old-job-id>           # remove stale duplicate
# verify removal:
hermes cron list                     # confirm only one remains
```

### 5. Verification Flow
After creating the job, verify:
1. `hermes cron list` — job appears with correct schedule, `--no-agent` mode, and next-run time.
2. Run script manually: `bash ~/.hermes/scripts/<filename>`.
3. Inspect script-owned log for fresh timestamps.
4. Confirm no duplicate names remain.

## Commands Used in This Session
```bash
# Discover correct CLI
hermes cron --help
hermes cron create --help

# Attempt (fails — absolute path rejected)
hermes cron create "*/5 * * * *" --name fleet-awareness-poller \
  --script /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh \
  --no-agent --workdir /Users/jack.reis/.hermes/cubby

# Fix: copy into scripts dir
cp /Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh \
   ~/.hermes/scripts/poll-fleet-awareness.sh

# Attempt (fails — symlink escapes scripts dir)
# (symlink was created earlier; removed, then copied)

# Create successfully
hermes cron create "*/5 * * * *" --name fleet-awareness-poller \
  --script poll-fleet-awareness.sh --no-agent \
  --workdir /Users/jack.reis/.hermes/cubby
# → Created job: 350ca37f891e

# Detect and remove duplicate
hermes cron list
hermes cron rm c5dc5751cd0f

# Verify final state
hermes cron list
bash ~/.hermes/scripts/poll-fleet-awareness.sh
```

## Environment
- macOS 15.7.5
- Hermes cron subsystem with `--no-agent` support
- Script: `poll-fleet-awareness.sh` (read-only fleet state snapshot)
