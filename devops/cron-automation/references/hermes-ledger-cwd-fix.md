# Hermes Ledger Cron Failure: `.git` scope + context recovery

## Symptom
`cronjob hermes-ledger` failed with:

- `[ledger] git log failed: fatal: not a git repository (or any of the parent directories): .git`
- status of other ledgers looked fine.

## Root cause
The scheduled command invoked `ledger.py append-commit` without explicit repository context, and the scheduler's current directory was not inside `/Users/jack.reis/Documents/=notes`.
`ledger.py append-commit` defaults to `Path.cwd()` and therefore depended on implicit CWD.

## Fix applied
1. Updated cron prompt/steps to call repository path explicitly:
   - `python3 /Users/jack.reis/Documents/=notes/claude/scripts/ledger.py append-commit --repo /Users/jack.reis/Documents/=notes --count 20`
2. Confirmed job definition includes:
   - `workdir: /Users/jack.reis/Documents/=notes`

## Post-fix verification
- Manual command check succeeds from non-repo dirs:
  - `python3 /Users/jack.reis/Documents/=notes/claude/scripts/ledger.py append-commit --repo /Users/jack.reis/Documents/=notes --count 20`
- Forced run via `cronjob(action="run", job_id=62f9a439503b)` succeeded.
- Latest output file confirmed success and clean status summary.

## Reusable pattern for future cron/automation jobs
- Never rely on scheduler-injected CWD.
- Prefer explicit absolute paths for:
  - repo-aware commands (`--repo`, `--path`, full input files)
  - output destinations and scratch paths.
- Add/adjust job `workdir` as a second guard rail.
- When fixing a failing job, test both:
  1) command directly in a known bad directory and
  2) the job object itself (`cronjob run`).

## Template
```bash
# in ~/.hermes/cron/jobs.json or cron prompt:
# command
python3 /Users/jack.reis/Documents/=notes/claude/scripts/ledger.py append-commit --repo /Users/jack.reis/Documents/=notes --count 20
```
