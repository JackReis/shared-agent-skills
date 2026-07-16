# Hermes Cron: Contained Wrapper for Versioned Implementations

Hermes validates the real path of the configured `script` under `~/.hermes/scripts/`. A symlink whose target lives in a vault or repository outside that directory is rejected before execution, even when the target exists and runs manually.

## Durable deployment pattern

Keep implementation code versioned in its canonical repository, but deploy a regular local wrapper as the cron entrypoint:

```bash
#!/usr/bin/env bash
set -euo pipefail
exec "$HOME/Documents/project/path/to/versioned-job.sh" "$@"
```

The configured job uses only the wrapper filename:

```text
script: versioned-job-wrapper.sh
no_agent: true
```

## Verification

1. `test -f ~/.hermes/scripts/versioned-job-wrapper.sh`
2. `test ! -L ~/.hermes/scripts/versioned-job-wrapper.sh`
3. `bash -n ~/.hermes/scripts/versioned-job-wrapper.sh`
4. Run the wrapper manually with a non-mutating/dry-run mode when supported.
5. Trigger the actual cron job and verify `last_run_at` changed and `last_status=ok`.
6. Check `last_delivery_error` separately. A task can execute successfully while message delivery fails.
7. Verify the job-owned artifact/log, not only scheduler metadata.

## Why not copy the implementation?

Copying works but creates two sources that drift. The regular-wrapper pattern preserves scheduler containment and a single versioned implementation. Symlinking the implementation does not satisfy the containment guard; wrapping it does.
