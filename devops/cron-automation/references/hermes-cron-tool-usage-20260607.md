# Hermes Agent Cron Subsystem — Tool-Based Usage (2026-06-07)

## Scenario
The `hermes` CLI was absent in this environment, but the **Hermes Agent cron subsystem** was available via the `cronjob` tool. This reference documents the exact tool-based workflow used to schedule, run, and verify a silent (no-agent) cron job.

## Steps

### 1. Write the Script
Write the audit script to `~/.hermes/scripts/config-drift-audit.sh`. The script must:
- Be self-contained (no external dependencies).
- Use `--no-agent` mode (no LLM token costs).
- Produce **no stdout** on PASS (silent).
- Append to a log file or inbox on DRIFT.

### 2. Schedule the Job
Use the `cronjob` tool to create the job:

```python
cronjob(
    action='create',
    name='config-drift-audit',
    schedule='0 7 * * 0',  # Sundays at 07:00 CT
    script='config-drift-audit.sh',
    no_agent=True,        # Silent, no LLM
    deliver='origin',      # This chat
    skills=['cron-automation']
)
```

### 3. Verify the Schedule
List jobs to confirm the new entry:

```python
cronjob(action='list')
```

Check for:
- `job_id` (e.g., `a8560bc5eda7`)
- `name` (`config-drift-audit`)
- `schedule` (`0 7 * * 0`)
- `script` (`config-drift-audit.sh`)
- `no_agent: true`
- `deliver: origin`

### 4. Run Immediately
Trigger the job for an immediate test run:

```python
cronjob(action='run', job_id='a8560bc5eda7')
```

The job runs on the next scheduler tick (within seconds).

### 5. Verify Execution
Poll the cron subsystem for output:

```python
cronjob(action='list')
```

Check for:
- `last_run_at` (timestamp)
- `last_status: ok` (exit code 0)

### 6. Verify Silent Output
Silent jobs (`no_agent=True`) produce **no stdout** on PASS. To verify:

- **PASS:** Confirm the script's intended log file (e.g., `memory/YYYY-MM-DD.md`) contains the PASS line (`config-drift-audit: PASS`).
- **DRIFT:** Confirm the script's inbox file (e.g., `claude/coordination/inbox-fleet.md`) was **not modified** (no new drift report).

### 7. Confirm Delivery
For jobs with `deliver: origin`, the cron subsystem delivers the script's stdout to the origin chat. On PASS, this is **empty** (silent). On DRIFT, the drift report is delivered as a message.

## Key Pitfalls
1. **Assuming `last_status: ok` means output was delivered.**
   - Silent jobs produce no stdout, so `origin` receives nothing even on success.
   - Always verify external artifacts (log files, inbox files).

2. **Assuming the daily log file exists.**
   - If `memory/YYYY-MM-DD.md` does not exist, the PASS line cannot be written.
   - The script should handle missing log files gracefully (e.g., create the file if needed).

3. **Assuming `cronjob(action='run')` runs synchronously.**
   - The job runs on the next scheduler tick, not immediately.
   - Poll `cronjob(action='list')` for `last_run_at` and `last_status`.

4. **Assuming the script path is absolute.**
   - The `script` parameter must be a filename under `~/.hermes/scripts/`.
   - Absolute paths or symlinks that escape the directory are rejected.

## Verification Checklist
- [ ] `cronjob(action='list')` shows the job with correct `schedule`, `script`, and `no_agent: true`.
- [ ] `last_status: ok` after the job runs.
- [ ] Script's log file or inbox file shows fresh timestamps (PASS) or no new entries (DRIFT).
- [ ] No unexpected stdout delivered to `origin` (silent PASS).