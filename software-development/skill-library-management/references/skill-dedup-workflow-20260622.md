# Skill Deduplication Workflow (2026-06-22)

## Context

After generating the agent skills map (653 logical skills, 823 instances), the user asked to ensure skills are not duplicative. This document captures the dedup analysis and removal process.

## Analysis technique

1. Load `raw/skill-inventory.json` from the generated map.
2. For each logical skill, check `source_systems` for multiple entries.
3. Compute content hashes from the `hashes` field (SHA-256 of full SKILL.md text).
4. Classify duplicates:
   - **Identical**: same name, same hash across all instances (1 distinct hash).
   - **Content drift**: same name, different hashes (>1 distinct hash).
5. Detect near-duplicate names using `difflib.SequenceMatcher` with similarity > 0.85. These are distinct skills with similar names, not true duplicates.
6. Determine canonical source: `~/.hermes/skills/` (Hermes default profile).

## Findings

- 81 duplicate skill names (multiple instances across sources)
- 41 with content drift (same name, different content)
- 34 shared skills exist in both Hermes and other sources
- 17 near-duplicate name pairs (distinct skills, no action needed)
- 9 plugin pack copies (same skill shipped in 9 different Claude plugin packs — all identical)

## Removal approach

Conservative: remove only from `~/.claude/skills/` and `~/Documents/pilot-sandbox/` (user-controlled paths). Leave:
- Plugin cache (`~/.claude/plugins/cache/`) — restored on plugin reinstall
- Vault (`~/Documents/=notes/claude/skills/`) — may be intentional

### What was removed (31 files from ~/.claude/skills/)

Skills that had identical content in both Hermes and ~/.claude/skills/:
- agents-sdk, cloudflare-email-service, durable-objects, prompt-feedback, sandbox-sdk, turnstile-spin, web-perf, workers-best-practices
- Superpowers copies: brainstorming, dispatching-parallel-agents, executing-plans, finishing-a-development-branch, receiving-code-review, requesting-code-review, subagent-driven-development, systematic-debugging, test-driven-development, using-git-worktrees, using-superpowers, verification-before-completion, writing-plans, writing-skills
- Other: smart-graph, cloudflare, wrangler, fleet-ratify, peer-grill, peer-grill-with-agents, improve-codebase-architecture

### What was NOT removed (user denied or skipped)

- 24 pilot-sandbox copies (user denied terminal deletion command)
- 17 Superpowers plugin cache copies (managed by plugin system)
- 2 vault copies (apple-reminders, smart-graph)

## Pitfalls

- **execute_code sandbox lacks filesystem permissions for deletion outside cwd.** The Python sandbox returned `PermissionError: [Errno 1] Operation not permitted` for files in `~/.claude/skills/` and pilot-sandbox. Use the `terminal` tool for file deletion.
- **User may deny bulk deletion commands.** The terminal `rm -f` command for pilot-sandbox files was blocked by the consent guard. Have a report ready as fallback.
- **Empty directories should be cleaned after SKILL.md removal.** Use `find <path> -depth -type d -empty -delete`.
- **Near-duplicate names are not duplicates.** Pairs like `fiftyone-dataset-export` vs `fiftyone-dataset-import` or `sentry-performance-tracing` vs `sentry-performance-tuning` are distinct skills serving different purposes. Only flag them — don't merge.

## Report location

Dedup report written to: `okf/fleet/agent-skills-map/reports/dedup-report.md`