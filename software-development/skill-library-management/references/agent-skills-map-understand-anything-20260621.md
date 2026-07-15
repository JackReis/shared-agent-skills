# Agent Skills Map with Understand Anything + Smart Graph (2026-06-21)

## When to use

Use this pattern when the user asks to map, audit, visualize, reconcile, or explore the whole local skill library across Hermes skills, Claude Code skills, plugin caches, and vault-local skill directories.

## Proven workflow

1. Load the class-level skills first:
   - `skill-library-management` for canonical skill/library rules.
   - `smart-graph` for Obsidian Smart Connections graph extraction.
   - `obsidian` if writing the output into the vault.
   - `hermes-agent` only for Hermes-specific CLI/plugin details; do not edit it if protected.

2. Inventory canonical and shadow skill roots, excluding transient job snapshots:
   - `~/.hermes/skills/`
   - `~/.claude/skills/`
   - `~/Documents/=notes/claude/skills/`
   - `~/Documents/pilot-sandbox/plugins/skill-enhancers/`
   - Installed Claude plugin cache skill roots from `~/.claude/plugins/installed_plugins.json`
   - Optional plugin cache fallback: `~/.claude/plugins/cache/*/*/*/skills`
   - Exclude `~/.claude/jobs/**`; these are historical/sandbox workspaces, not canonical installed skills.

3. Normalize each `SKILL.md` into both:
   - a physical instance record: path, real path, source system, category, hash, line/word counts, support-file counts.
   - a logical skill record grouped by frontmatter `name` or skill directory name.

4. Generate a Karpathy-pattern wiki so Understand Anything can parse it deterministically:
   - `index.md` with wikilinked cluster sections.
   - `log.md` with a dated operation entry.
   - `AGENTS.md` or `CLAUDE.md` schema note.
   - `skills/*.md`, `clusters/*.md`, `categories/*.md`, `sources/*.md`, `reports/*.md`.
   - `raw/skill-inventory.json` and `raw/smart-graph-baseline.json` as evidence sidecars.

5. Use Understand Anything's deterministic knowledge-base scripts directly when slash commands are unavailable:
   - Plugin root observed here: `~/.claude/plugins/cache/understand-anything/understand-anything/<version>`.
   - Parser: `skills/understand-knowledge/parse-knowledge-base.py <wiki-dir>`.
   - Merger: `skills/understand-knowledge/merge-knowledge-graph.py <wiki-dir>`.
   - The parser writes `.understand-anything/intermediate/scan-manifest.json`.
   - The merger writes `.understand-anything/intermediate/assembled-graph.json`; copy it to `.understand-anything/knowledge-graph.json` and write `.understand-anything/meta.json`.
   - You can add a deterministic `analysis-batch-0.json` before merge to add entity/claim nodes and extra edges without invoking LLM subagents.

6. Use Smart Graph as a baseline, not as the only graph:
   - Run `extract-graph.py --vault <vault>` to record current Smart Connections stats.
   - If the target note/skill is disconnected in Smart Connections, generate a dense wikilinked wiki anyway; Obsidian can index it later.

7. Verify before reporting:
   - Count markdown pages produced.
   - Load `knowledge-graph.json` and count nodes/edges/layers/tour steps.
   - Check every edge source/target exists; dangling edge count must be zero.
   - If launching the dashboard, verify `/knowledge-graph.json?token=...` returns JSON and the expected node/edge counts.

## Dashboard setup pitfall

Understand Anything's dashboard may require building the plugin workspace first. The useful fix is setup, not a negative memory about the tool:

```bash
cd ~/.claude/plugins/cache/understand-anything/understand-anything/<version>
pnpm approve-builds --all
pnpm install --frozen-lockfile
pnpm --filter @understand-anything/core build
cd packages/dashboard
UNDERSTAND_ACCESS_TOKEN=<random-hex-token> \
GRAPH_DIR=<wiki-dir> \
npx vite --host 127.0.0.1
```

If process logs are empty, probe localhost ports and fetch the protected endpoint with the token; Vite may be running even if Hermes captured no stdout.

## Example verification from the original session

Generated under `/Users/jack.reis/Documents/=notes/okf/fleet/agent-skills-map`:

- 653 logical skills
- 823 skill file instances
- 396 explicit related-skill edges
- Understand Anything graph: 865 nodes, 14,732 edges, 13 layers, 12 tour steps
- Dangling edges: 0
- Smart Graph baseline: 15,998 indexed notes, 32,016 internal links, 11,745 orphans

## Maintenance: adding skills to an existing graph

When skills are already registered in the Open Skills registry (`registry.json`) or a catalog page (e.g. `okf/fleet/tools/ai-exec-circle-community-skills.md`) but missing from the knowledge graph itself, use this incremental update pattern instead of regenerating the whole map.

### Symptom

- `registry.json` lists the skill with `harnesses` and `instances` — it is installed and enabled.
- A catalog page (e.g. AI Exec Circle Community Skill Adapters) references the skill with contributor attribution and local path.
- But `skills/` has no page for it, `clusters/` doesn't list it, and `index.md` has no entry.

This happens when skills are installed to a harness (e.g. `~/.codex/skills/`) and registered via the aggregator script, but the agent-skills-map wiki was generated from a different skill-root snapshot that didn't include them.

### Fix pattern (four layers, top-down)

1. **Skill page** — Create `skills/<skill-name>.md` with:
   - Frontmatter: `type: agent-skill`, `name`, `cluster`, `instance_count`, `content_hash_count`, `contributor` (if a community adapter).
   - Body: description, map placement (cluster, source category, source system), relationships (explicit related skills, nearby skills), tags, source instances with real paths, full provenance block (contributor, local sources, GitHub repos, OKF catalog cross-link, registry reference), and notes.
   - Cross-link to the catalog page via `[[tools/<catalog-page>|<Catalog Title>]]`.

2. **Cluster page** — In `clusters/<cluster>.md`:
   - Add to the "High-signal skills" section with a one-line description.
   - Add to the "All skills" alphabetical list.
   - Update the `skill_count` in frontmatter.
   - Update the "… N more" counter in the high-signal section.

3. **Index** — In `index.md`, add the skill in alphabetical position in the main skill list.

4. **Log** — In `okf/fleet/log.md`, append a dated entry listing what was added and noting the `knowledge-graph.json` regeneration caveat.

### Caveat: knowledge-graph.json is not auto-updated

The markdown layer (skill pages, cluster, index) is the human/Obsidian-navigable surface and is current after the manual update above. But `.understand-anything/knowledge-graph.json` and `meta.json` are generated by the parse/merge scripts and will NOT pick up the new nodes until re-run:

```bash
cd ~/.claude/plugins/cache/understand-anything/understand-anything/<version>
python3 skills/understand-knowledge/parse-knowledge-base.py <wiki-dir>
python3 skills/understand-knowledge/merge-knowledge-graph.py <wiki-dir>
cp <wiki-dir>/.understand-anything/intermediate/assembled-graph.json \
   <wiki-dir>/.understand-anything/knowledge-graph.json
```

The manual markdown update is sufficient for Obsidian Smart Graph navigation and human reference. The JSON regeneration is only needed if the Understand Anything dashboard must reflect the new nodes immediately.

### Verification

```bash
# Skill pages exist
ls -la <wiki-dir>/skills/<skill-name>.md

# Cluster mentions each skill (should be >= 2: high-signal + full list)
grep -c "<skill-name>" <wiki-dir>/clusters/<cluster>.md

# Index mentions each skill (should be 1)
grep -c "<skill-name>" <wiki-dir>/index.md

# Log entry exists
grep -c "<date>" <okf-dir>/log.md
```

### Example session (2026-06-22)

Added three AI Exec Circle community adapter skills to `okf/fleet/agent-skills-map/`:
- `atlas-scout-concept-pipeline` (Ankit Patel) — was in registry.json with harness=codex but no graph page
- `group-call-takeaway-synthesis` (Brian Proffitt) — same gap
- `session-close-reconciliation` (Ant) — same gap

All three were installed as Codex adapters (`~/.codex/skills/`) and listed in `ai-exec-circle-community-skills.md` catalog, but absent from the knowledge graph wiki. Fixed via the four-layer pattern above. Commit: `99efb4978` on `neural-garden`.

## What not to save

Do not save one-off dashboard tokens, process IDs, exact temporary files, or transient missing-dependency errors. Save the repeatable setup and verification pattern instead.
