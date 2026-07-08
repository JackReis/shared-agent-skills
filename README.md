# Shared Agent Skills

Skills that work across both [Hermes Agent](https://hermes-agent.nousresearch.com)
and [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Contents

Skills here exist in both `~/.hermes/skills/` (or the worker profile) and
`~/.claude/skills/`. They are framework-agnostic SKILL.md files that follow
the common frontmatter format (`name`, `description`) understood by both
agent systems.

Current shared skills include Cloudflare Workers SDK tooling, sandbox
tooling, and code-review/skillify workflows.

## Sync

Synced from Aegis via `~/agent-configs/sync.sh`.
