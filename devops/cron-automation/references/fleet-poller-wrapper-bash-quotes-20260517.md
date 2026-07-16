# Fleet Poller Wrapper - Bash Quote Nesting Fix (2026-05-17)

## Session Context
Fleet awareness poller needed a wrapper script to run as a background persistent loop (cron unavailable). The wrapper script initially failed due to a quote nesting error in the startup log message.

## The Problem

**Initial (broken) line:**
```bash
echo "$(date '+%Y-%m-%d %H:%M:%S) INFO: Fleet poller started (PID: $$)" >> "$LOG_FILE"
                                                                     ^
                                                                     |
                                                          bash interpreted this ' as closing $(date '...')
```

**Error:**
```
/Users/jack.reis/.hermes/cubby/scripts/fleet-poller-loop.sh: line 19: unexpected EOF while looking for matching `''
/Users/jack.reis/.hermes/cubby/scripts/fleet-poller-loop.sh: line 25: syntax error: unexpected end of file
```

**Root cause:** The single quotes around the date format string (`'+%Y-%m-%d %H:%M:%S'`) were nested inside double quotes but the closing `)` of the command substitution happened INSIDE the quoted string, confusing the parser.

## The Fix

**Corrected line:**
```bash
echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: Fleet poller started (PID: $$)" >> "$LOG_FILE"
                                 ^
                                 |
                        Properly close the command substitution before the string continues
```

## Key Lesson

When nesting single quotes inside command substitution within double quotes:
- The single quotes for the date format ARE properly escaped by being inside double quotes
- BUT the command substitution `$(...)` must fully complete before the surrounding string continues
- The original error was the placement of the closing `)` inside the double-quoted string content

## Secondary Pattern: Duplicate Process Cleanup

**Discovery:** Multiple pollers from previous sessions were running concurrently:
```
jack.reis  79038  ... bash /tmp/fleet-poller.sh
jack.reis  79254  ... bash /Users/jack.reis/.hermes/cubby/scripts/fleet-poller-loop.sh
jack.reis  45406  ... bash /Users/jack.reis/.hermes/cubby/scripts/fleet-poller-daemon.sh
```

**Cleanup pattern used:**
```bash
pkill -f "fleet-poller" 2>/dev/null
sleep 1
rm -f /tmp/fleet-poller.lock
```

**Better approach (do at startup):** Always check for existing PIDs matching the pattern before starting, even if using a lockfile. Lockfiles can be stale if the cleanup script or previous run didn't complete properly.

## Working Wrapper Template

```bash
#!/bin/bash
# fleet-poller-loop.sh — Background fleet awareness poller

LOG_SCRIPT="/Users/jack.reis/.hermes/cubby/scripts/poll-fleet-awareness.sh"
LOCKFILE="/tmp/fleet-poller.lock"
LOG_FILE="/Users/jack.reis/.hermes/cubby/logs/fleet_awareness.log"

# Cleanup stale lockfile and existing processes
if [ -f "$LOCKFILE" ]; then
    OLD_PID=$(cat "$LOCKFILE" 2>/dev/null)
    if [ -n "$OLD_PID" ] && kill -0 "$OLD_PID" 2>/dev/null; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: Existing poller running (PID $OLD_PID), exiting" >> "$LOG_FILE"
        exit 0
    fi
    rm -f "$LOCKFILE"
fi

echo $$ > "$LOCKFILE"
echo "$(date '+%Y-%m-%d %H:%M:%S') INFO: Fleet poller started (PID: $$)" >> "$LOG_FILE"

while true; do
    bash "$LOG_SCRIPT" >> "$LOG_FILE" 2>&1
    sleep 300
done
```

## Reference
See also `references/fleet-awareness-poller-manual-overlap-20260511.md` for lock overlap handling.
