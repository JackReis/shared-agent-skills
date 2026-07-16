# Fleet Poller Duplicate Instance Cleanup Pattern

**Date:** 2026-05-17
**Context:** Running fleet awareness poller every 5 minutes in background

## Problem

Multiple instances of the same poller were running simultaneously (10+ copies), each polling every 5 minutes. This caused:
- Overlapping timestamps in logs (multiple polls at same second)
- Log corruption from concurrent writes
- False "change detected" signals from race conditions
- Inflated system load

## Detection

```bash
# Count instances of the poller
ps aux | grep "poll-fleet-awareness" | grep -v grep | wc -l
# Result: 10 (should be 0 or 1)

# View all instances
ps aux | grep "poll-fleet-awareness" | grep -v grep
```

Sample output showing duplicates:
```
jack.reis  68540  ... while true; do nice -n 10 bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh; sleep 300; done
jack.reis  68539  ... while true; do nice -n 10 bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh; sleep 300; done
jack.reis  96255  ... while true; do bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh 2>&1; sleep 300; done
jack.reis  64372  ... mkdir -p ~/.hermes/cubby/run && while true; do bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh; sleep 300; done
...
```

## Resolution

### Step 1: List all PIDs
```bash
ps aux | grep "poll-fleet-awareness" | grep -v grep | awk '{print $2}'
```

### Step 2: Terminate gracefully
```bash
for pid in <list of PIDs>; do kill $pid 2>/dev/null; done
sleep 2
ps aux | grep "poll-fleet-awareness" | grep -v grep | wc -l
```

### Step 3: Force kill if needed
If graceful termination leaves orphans:
```bash
ps aux | grep "poll-fleet-awareness" | grep -v grep | awk '{print $2}' | xargs -I {} kill -9 {} 2>/dev/null
```

### Step 4: Verify cleanup
```bash
ps aux | grep "poll-fleet-awareness" | grep -v grep
# Should return empty
```

### Step 5: Start single clean instance
```bash
# Verify scripts/log dir exists
mkdir -p ~/.hermes/cubby/logs

# Run once to establish clean state
bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh

# Start background loop with 300s (5-min) interval
while true; do sleep 300; bash ~/.hermes/cubby/scripts/poll-fleet-awareness.sh; done
```

## Prevention in Future Sessions

Always check for duplicates before starting background pollers:

```bash
# Pre-flight check
EXISTING=$(ps aux | grep "poll-fleet-awareness" | grep -v grep | wc -l | tr -d ' ')
if [ "$EXISTING" -gt 0 ]; then
    echo "Warning: $EXISTING poller instance(s) already running"
    ps aux | grep "poll-fleet-awareness" | grep -v grep
    # Optionally: ask user to clean up first, or auto-clean
fi
```

## Verification Post-Cleanup

```bash
# Confirm single instance
ps aux | grep "poll-fleet-awareness" | grep -v grep | wc -l
# Expected: 1

# Confirm log is getting fresh timestamps
tail -5 ~/.hermes/cubby/logs/fleet_awareness.log
```

## Related
- `fleet-poller-wrapper-bash-quotes-20260517.md` — quote nesting issues with while loops

## Update: Scheduler vs Loop Distinction (2026-05-17)

**Lesson:** After cleaning up duplicate loops, discovered that `hermes cron list` showed the job (`c5dc5751cd0f fleet-awareness-poller`) was already properly scheduled via the Hermes cron system. The while-true loops were legacy artifacts competing with the proper scheduler.

**Revised Pattern:**
1. Check for scheduler-managed jobs BEFORE starting background loops
2. If cron already manages the interval (e.g., `every 5m`), don't create a while-true loop
3. Clean up orphaned loops and let the scheduler handle it

```bash
# Check scheduler status
hermes cron list 2>/dev/null | grep "fleet-aware"

# If found (e.g., "every 5m"), verify it's active and clear competing loops
# Don't start new loops when scheduler exists
