# Low-resource OB1 shelfmark scout (2026-06-07)

## Context

A Librarian-profile cron was created to maintain a quiet map of major local "volumes" and "indexes" using OB1 as a searchable mirror. The goal was not full indexing on every tick; it was a low-resource card-catalog heartbeat that notices material shelfmark changes and stays silent otherwise.

## Pattern

Use Hermes cron in `no_agent` mode with a deterministic script that:

1. Reads a small, curated set of canonical files, not the whole vault.
2. Computes stable fingerprints for those files and for shallow/top-level directory summaries.
3. Ignores volatile names that would make the job report on its own footsteps: `.git`, `.DS_Store`, cron runtime files, state DB WAL/SHM files, profile `state/`, and skill `.usage.json`.
4. Writes a compact state JSON under the active profile.
5. Captures an OB1 thought only when the stable fingerprint changes.
6. Prints nothing when unchanged so `no_agent` cron remains silent.

## Good watched surfaces for the Librarian role

- OBn / vault-local memory: `claude/memory/HERMES-MEMORY.md`
- Current daily note
- `LATEST_HANDOFF.md`
- `SESSION-CONTEXT.md`
- priority / WIP dashboards
- `AGENTS.md` surfaces
- OB1 personalization convention
- profile `config.yaml`
- `.mcp.json`
- shallow summaries of important roots: vault, handoffs, plans, profile, skills, ai-dev, Codex sessions, Claude config

## Verifier

After scheduling:

```bash
python3 /path/to/scout.py              # first run should print a material baseline/capture
python3 /path/to/scout.py              # second run should print nothing
hermes cron tick --profile <profile>
hermes cron list --profile <profile>  # Last run: ... ok
python3 /path/to/ob1-pull --query "[LIBRARIAN_STACKS]" --limit 3
```

## Pitfalls

- Do not include `.git` or profile cron/state files in top-level mtime summaries; otherwise every run mutates its own state and triggers the next run.
- Do not shell out to `hermes cron list` as part of the fingerprint unless you strip `last_run_at`, `next_run_at`, and `completed` counters. Prefer reading the profile `cron/jobs.json` and keeping only stable job fields.
- In named Hermes profiles, `$HOME` may point at the profile home. OB1 token helpers that need Jack's real `~/.secrets` should be invoked with the real-home form: `HOME=/Users/jack.reis /Users/jack.reis/.hermes/bin/ob1-token.sh ...`.
- Treat OB1 as a mirror/retrieval layer. The daily note and handoff files remain canonical for session state.
