# Runtime Fallbacks for Cron-Style Pollers

Session-derived notes for environments where a cron-style monitor is requested but the runtime does not provide a persistent scheduler.

## Observed failure mode
- The target container did not expose `crontab`.
- Shell-launched background loops exited after a short time instead of persisting.
- A background poller could run once and write a state/log artifact, but it could not be trusted to stay alive as a recurring monitor.

## Safe fallback pattern
1. Run the poller once as an ad hoc verification step.
2. Capture stdout/stderr to a local log file.
3. Write a state snapshot so the next run can detect changes.
4. Report the limitation explicitly: recurring monitoring is not active in this runtime.

## What not to do
- Do not claim the job is “running every 5 minutes” unless a durable scheduler is actually present.
- Do not hide scheduler absence behind a silent failure.
- Do not rely on detached shell wrappers as proof of persistence.

## Verification hints
- Check whether `crontab` exists before attempting installation.
- Poll the spawned process if the tool supports it; if it exits quickly, treat the loop as non-durable.
- Confirm the log file and state file were written by the one-shot execution.

## Related signals
This pattern showed up while working on the fleet awareness poller, but it applies to any recurring read-only monitor that must survive shell session boundaries.
