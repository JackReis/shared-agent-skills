# No-agent cron content: source, state, and wrapper verification

Use this when a scheduled message is stale but the job has `no_agent=true`.

## Core rule

For a no-agent job, the scheduler does not reason over `prompt` or attached skills. The script runs directly and its stdout is delivered verbatim. Editing the prompt may improve operator-facing metadata, but it does **not** change the delivered content.

## Repair sequence

1. Identify the actual profile/store containing the job. Do not assume a service-specific profile owns the cron; inspect the default store and relevant profile stores separately.
2. Read the job's `script` field and follow every wrapper until reaching the implementation that produces stdout.
3. Identify whether stale output comes from:
   - hard-coded IDs/text in the script,
   - current source data such as open/ready task rows,
   - a stale generated meta-record,
   - or an LLM prompt (only for agent-driven jobs).
4. Verify live state before removing a go-live/setup item. A running service alone is insufficient when the task also names a transport, bot identity, or data path; probe those independently.
5. Resolve the authoritative task/state record, then remove obsolete hard-coded classification entries if they can keep resurfacing it.
6. Keep the implementation versioned outside `~/.hermes/scripts/` and use a regular contained wrapper there.
7. Ensure every shell wrapper forwards arguments:

```bash
exec "$HOME/path/to/versioned/job.sh" "$@"
```

Without `"$@"`, flags such as `--dry-run` are silently discarded and a supposedly safe verification may execute the mutating path.
8. Verify with language syntax checks, shell syntax checks, and a real dry-run whose banner/output proves the flag reached the implementation. Confirm the stale item is absent from stdout.
9. Re-list the cron job and verify schedule, delivery, `no_agent`, and script filename were unchanged unless intentionally modified.

## Reporting distinction

Report subsystem health precisely. For example, a gateway and Telegram transport can be live while an iMessage sidecar is degraded. Do not collapse partial transport failure into either “fully down” or “fully healthy.”
