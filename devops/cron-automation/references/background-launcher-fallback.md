# Background launcher fallback notes

Session observed: a silent fleet-awareness monitor was started with `terminal(background=true)`, but the process API later reported the session exited almost immediately. A direct `bash` loop launch also failed to persist. A plain Python launcher that double-forked and wrote a PID file succeeded in keeping the monitor alive.

## What to verify
- Launch prints a PID or writes a PID file
- `ps -p <pid>` shows the process alive after startup
- The log file gets a fresh timestamped entry after launch
- The monitor stays alive long enough to survive the first sleep interval

## Practical lesson
Do not trust a green launch message alone for unattended polling. In this runtime, prefer a scheduler or a launcher that can be independently verified after it starts.