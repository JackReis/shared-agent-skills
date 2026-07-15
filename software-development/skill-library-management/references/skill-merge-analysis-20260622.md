# Skill Near-Duplicate Merge Analysis (2026-06-22)

## Context

After the dedup pass removed 31 identical copies from `~/.claude/skills/`, the user asked to "merge the over-similar skills but leave multiple copies if there is any risk of an agent not finding the canonical version(s)."

## Method

1. Loaded `raw/skill-inventory.json` from the generated agent-skills-map.
2. Computed near-duplicate pairs using `difflib.SequenceMatcher` on lowercased skill names, threshold similarity > 0.85.
3. Found 17 near-duplicate pairs.
4. For each pair, compared: descriptions, clusters, source systems, content hashes, and actual SKILL.md content (where accessible).
5. Classified each as MERGE, DISTINCT, or PLUGIN_MANAGED.

## Result: 0 merges needed

All 17 pairs are distinct skills with similar names. No merges required.

### Key findings per pair

| Pair | Sim | Verdict | Reason |
|---|---:|---|---|
| langchain-core-workflow-a / -b | 0.96 | DISTINCT | A=chains/prompts, B=agents/tools |
| obsidian-core-workflow-a / -b | 0.958 | DISTINCT | A=file ops, B=UI components |
| granola-core-workflow-a / -b | 0.957 | DISTINCT | A=pre-meeting, B=post-meeting |
| linear-core-workflow-a / -b | 0.955 | DISTINCT | A=issue CRUD, B=project/cycle mgmt |
| printing-press-polish / -publish | 0.93 | DISTINCT | Sequential pipeline steps |
| cua-driver-rs / cua-driver-rust | 0.929 | DISTINCT | Rust crate ref vs Hermes skill |
| linear-ci-integration / linear-integration | 0.923 | DISTINCT | CI+GitHub Actions vs Hermes+Linear API |
| github-pr-workflow / github-workflow | 0.909 | ALREADY MERGED | PR workflow in .archive/, umbrella covers it |
| fiftyone-dataset-export / -import | 0.913 | DISTINCT | Opposite operations |
| sentry-performance-tracing / -tuning | 0.902 | DISTINCT | Setup vs optimization |
| firecrawl-instruct / -interact | 0.889 | PLUGIN MANAGED | Renamed across plugin versions |
| fiftyone-develop-plugin / -eval-plugin | 0.884 | DISTINCT | Create vs review |
| kanban-orchestrator / kanban-video-orchestrator | 0.864 | DISTINCT | General vs video-specific |
| gitnexus-explorer / gitnexus-exploring | 0.857 | DISTINCT | Index+serve vs query+navigate |
| grill-me-agents / grill-me-with-agents | 0.857 | DISTINCT | Greenfield design vs existing-stack changes |
| printing-press-reprint / -retro | 0.857 | DISTINCT | Regenerate vs retrospective |
| langchain-reference-architecture / linear-reference-architecture | 0.852 | DISTINCT | Different systems entirely |

## Pitfall

- **Name similarity does not imply functional duplication.** The `grill-me-agents` vs `grill-me-with-agents` pair looked like a merge candidate (same cluster, same sources, B described as "code-aware variant of A"). But reading the actual descriptions revealed: A is for greenfield multi-agent design from scratch, B is for stress-testing changes to an already-implemented stack. They're explicitly different use cases. Always read descriptions and content before merging.
- **`*-workflow-a` / `*-workflow-b` naming is a plugin pack convention** for complementary primary/secondary workflows, not duplicate detection signal. Same pack, different concerns.

## Report location

Full merge analysis: `okf/fleet/agent-skills-map/reports/merge-analysis.md`