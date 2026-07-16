# Reference: fleet-poller-cron-vs-orphan-loops-20260525
## Prevent competing while-true loops when cron already handles the job

**Symptom:** After a prior session spawned a `while true; sleep 300; …` background loop, multiple poller instances competed with the scheduled cron job, causing duplicate log entries and potential lock contention.

**Root cause:** The agent created a background loop without checking whether cron already maintained the job. Cron and the loop ran in parallel, both executing the same script.

**Detection:**
```bash
ps aux | grep poll-fleet-awareness | grep -v grep
```
If more than one bash process is running the script, duplicates exist.

**Fix:**
1. Before starting any background loop for polling, run: `crontab -l | grep <script-name>`
2. If a cron entry exists at the requested interval, do NOT create a loop. Clean up any orphaned loop processes:
   ```bash
   ps aux | grep poll-fleet-awareness | grep -v grep | awk '{print $2}' | xargs kill -9
   ```
3. The cron entry is the authoritative mechanism; remove competing while-true loops to prevent log corruption and false change signals.

**Verification:** After cleanup, confirm only zero or one poller processes remain.
