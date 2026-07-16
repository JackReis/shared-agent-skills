# Cronjob Chaining with `context_from`

## Workflow
1. **Create Pre-Flight Job**:
   ```yaml
   action: create
   schedule: "30 3 * * *"
   name: "Daily Briefing Pre-Flight"
   prompt: |
     Fetch Google Daily Briefing and session logs.
     Save results to a temporary file.
   skills: [hermes-agent, obsidian]
   deliver: local
   ```

2. **Create Assembly Job**:
   ```yaml
   action: create
   schedule: "0 4 * * *"
   name: "Daily Briefing Assembly"
   prompt: |
     Load pre-flight results and create/update the daily note.
   skills: [obsidian, obsidian-librarian]
   context_from: ["Daily Briefing Pre-Flight"]
   deliver: origin
   ```

## Pitfalls
- **Job ID Mismatch**: Ensure `context_from` references the correct `job_id`.
  - **Debugging**: Use `cronjob(action='list')` to verify job IDs.
- **Order of Execution**: Pre-flight must run before assembly.
  - **Fix**: Schedule pre-flight at least 30 minutes before assembly.