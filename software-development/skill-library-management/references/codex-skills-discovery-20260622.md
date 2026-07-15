# Codex Skills Discovery — 2026-06-22

## Context

During the agent skills map session, the user asked "have we incorporated Ankit's?" — revealing that `~/.codex/skills/` was never scanned as a source root. The initial inventory covered Hermes, Claude Code, vault, pilot-sandbox, and Claude plugin cache, but missed Codex entirely.

## What was found

16 Codex skill instances at `~/.codex/skills/`:

### Codex-only skills (5, not in Hermes)

| Skill | Contributor | Description |
|---|---|---|
| atlas-scout-concept-pipeline | Ankit Patel | Atlas + Scout pattern for concept evaluation. Composes Hermes `atlas-chunking` + `scout-prioritization`. |
| group-call-takeaway-synthesis | Brian Proffitt | Turn community calls/meetings into crisp takeaways. |
| session-close-reconciliation | Ant | Close/compact agent sessions with fleet coordination reconciliation. |
| chronicle | — | Session history chronicling. |
| open-skills | — | Map Unlock AI Open Skills into local agent skills. |

### Shared skills (11, identical to Hermes canonical)

agents-sdk, cloudflare, cloudflare-email-service, durable-objects, kimi-webbridge, prompt-feedback, sandbox-sdk, turnstile-spin, web-perf, workers-best-practices, wrangler

## Provenance

From the AI Exec Circle community (Nate B. Jones's circle). Local adapters, not upstream Unlock AI skills. See `okf/fleet/tools/ai-exec-circle-community-skills.md` for full provenance.

Related GitHub repos (not cloned locally):
- `AnkitClassicVision/ankit_shared_skills` — run-project-bundle plugin (not installed)
- `AnkitClassicVision/Agent-flow`
- `AnkitClassicVision/aac-factory`
- `AnkitClassicVision/Claude-Code-Deep-Research`

## Fix applied

Added `~/.codex/skills/` as a scan root in the agent skills map inventory (`raw/skill-inventory.json`). Updated counts: 658 logical skills (was 653), 839 instances (was 823). Created source page at `okf/fleet/agent-skills-map/sources/codex-local-skills.md` and skill pages for all 5 Codex-only skills.

## Lesson

Any whole-library skill map must include `~/.codex/skills/` as a scan root. Codex is a first-class agent harness with its own skill tree, and community skill adapters are installed only there.