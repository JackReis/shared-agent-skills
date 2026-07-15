# Community Skill Promotion — 2026-06-23

## Context

User asked "have we incorporated Ankit's skills and workflow?" Investigation revealed Ankit Patel's `atlas-scout-concept-pipeline` (plus Brian Proffitt's `group-call-takeaway-synthesis` and Ant's `session-close-reconciliation`) were installed as Codex adapters at `~/.codex/skills/` and registered in the OKF agent-skills-map knowledge graph, but NOT installed in `~/.hermes/skills/`. Hermes agents could not load them — only Codex could.

A plan was written to promote them (`~/.hermes/plans/2026-06-23-ankit-skills-incorporation.md`) covering 5 gaps:
1. Codex-only → Hermes canonical skill installation
2. Stale `knowledge-graph.json` (3 days old, missing 3 skill nodes)
3. VoltAgent console data loader missing new Hermes paths
4. Ankit's 4 GitHub repos referenced but not cloned/triaged
5. Ankit's 48 ingested principles not verified in OBn ChromaDB

## Promotion pattern

### What to promote

| Skill | Contributor | Hermes category | Composes |
|---|---|---|---|
| atlas-scout-concept-pipeline | Ankit Patel | autonomous-ai-agents | atlas-chunking + scout-prioritization |
| group-call-takeaway-synthesis | Brian Proffitt | note-taking | (standalone) |
| session-close-reconciliation | Ant | note-taking | Hermes session-close-ritual |

### Decision gate

Promote when:
- The skill composes existing Hermes skills and adds orchestration value
- The skill is harness-agnostic (doesn't depend on Codex CLI internals)
- The source is confirmed free (paid policy enforced)

Do NOT promote when:
- The skill is a thin wrapper around a harness-specific CLI
- The skill duplicates an existing Hermes skill with no added value
- User hasn't approved the promotion

### Steps executed (from the plan)

1. `mkdir -p ~/.hermes/skills/autonomous-ai-agents/atlas-scout-concept-pipeline/`
2. Copy `~/.codex/skills/atlas-scout-concept-pipeline/SKILL.md` to the new dir
3. Verify frontmatter has `version` field (add if missing)
4. `skill_view(name='autonomous-ai-agents/atlas-scout-concept-pipeline')` to confirm
5. Update `okf/fleet/tools/ai-exec-circle-community-skills.md` to note dual-install
6. Re-run Understand Anything parse/merge to refresh `knowledge-graph.json`

## Stale graph timing issue

The `knowledge-graph.json` was last regenerated Jun 20 22:25. Three new skill pages were added Jun 22 via the four-layer maintenance pattern, but the JSON graph was not refreshed. This went unnoticed for 3 days because the markdown layer looked current.

**Lesson:** After any skill addition, removal, or promotion, re-run the Understand Anything parse/merge in the same session. Do not defer — deferral compounds.

```bash
cd ~/.claude/plugins/cache/understand-anything/understand-anything/<version>
python3 skills/understand-knowledge/parse-knowledge-base.py <wiki-dir>
python3 skills/understand-knowledge/merge-knowledge-graph.py <wiki-dir>
cp <wiki-dir>/.understand-anything/intermediate/assembled-graph.json \
   <wiki-dir>/.understand-anything/knowledge-graph.json
```

## Related

- Plan: `~/.hermes/plans/2026-06-23-ankit-skills-incorporation.md`
- OKF provenance: `okf/fleet/tools/ai-exec-circle-community-skills.md`
- Codex discovery reference: `references/codex-skills-discovery-20260622.md`
- Map maintenance reference: `references/agent-skills-map-understand-anything-20260621.md`