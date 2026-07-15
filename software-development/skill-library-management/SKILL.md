---
name: skill-library-management
description: Discover, register, and maintain Hermes skills from non-default paths (plugin directories, external repos, shared vaults). Use when skills are missing from `skills_list()`, when the user mentions plugin skills, when symlinking skills, or when reconciling duplicate/overlapping skills between the default tree and external sources.
---

# Skill Library Management

Maintain a complete, deduplicated, discoverable skill library. The default `~/.hermes/skills/` tree is not the only source of skills — plugins, shared vaults, and external repos may hold additional skills that `skills_list()` cannot see.

## Glossary

- **Default skill tree** — `~/.hermes/skills/`, scanned by `skills_list()`.
- **Plugin skill directory** — skills shipped outside the default tree, e.g. `~/Documents/pilot-sandbox/plugins/skill-enhancers/`.
- **Shadow skill** — a skill that exists on disk but is invisible to `skills_list()` because it lives outside the default tree.
- **Skill overlap** — two skills (one default, one plugin) with the same name but potentially different content.

## Principles

1. **If a skill is missing from `skills_list()`, check non-default paths before concluding it does not exist.**
2. **Symlink, don't copy.** Register plugin skills via symlink into `~/.hermes/skills/` so updates to the plugin source propagate automatically.
3. **Verify after registering.** Run `skills_list()` and `skill_view()` on newly-registered skills to confirm they are discoverable and readable.
4. **Watch for overlaps.** Plugin skills may duplicate names already in the default tree. Compare descriptions and content when conflicts arise.
5. **Separate canon from distribution.** For team skills, prefer Git-backed `SKILL.md`/`AGENTS.md` files as the byte-level source of truth; use Paperclip company skills as an operational registry/distribution layer, ContextForge/Open Brain as decision/semantic memory, and Alfred/iCloud only as human convenience surfaces. See `references/team-canonical-skills-architecture-20260614.md`.
6. **Separate drafting, evolution, stamping, and projection.** A skill-creation prompt drafts; an evaluator proves improvement; a publisher installs approved artifacts; a scanner-generated registry only projects installed state. Do not let one layer silently impersonate another.

## Team canonical skills architecture

When Jack asks where team/canonical skills should live, use this default stance:

- **Canonical reusable workflows:** Git-backed class-level `SKILL.md` library with `references/`, `templates/`, and `scripts/` support files.
- **Canonical repo-local behavior:** repo-root `AGENTS.md`, created only after scan-first validation of existing repo docs, commands, CI, and pitfalls.
- **Operational distribution:** Paperclip company skills imported/synced from Git-backed skills; do not make Paperclip the only canonical store.
- **Decision/index memory:** ContextForge/Open Brain stores rationale, relationships, and retrieval metadata, not byte-level skill truth.
- **Human shortcuts:** Alfred snippets and iCloud folders are convenience mirrors/snippets only.

Roll out repo `AGENTS.md` files incrementally: inventory first, prioritize active/high-risk repos, pilot on 2–3 repos, verify each runtime actually consumes the file, then broaden. Avoid blanket-generated instruction files and avoid parallel hand-maintained `AGENTS.md` + `CLAUDE.md` + `.cursorrules` drift.

### Skill evolution and cross-harness stamping

When a workflow combines self-improvement with fleet distribution:

1. Inspect the authority host or canonical repository before mirrors and execution edges.
2. Audit source, tests, plans, and generated artifacts separately; README status is not completion evidence.
3. Preserve a baseline and evaluate candidates against validation plus untouched holdout cases.
4. Keep orchestration, checkpoints, constraints, and receipts deterministic where possible; models propose and judge content.
5. Promote only after explicit human approval into one Git-backed canonical package.
6. Stamp adapters into target harnesses, verify runtime visibility, then regenerate registry projections from their scanners.
7. If another session is already writing the implementation, assign this lane as a read-only independent reviewer instead of producing competing files.

Detailed architecture and verification gates: `references/evolution-and-cross-harness-stamping.md`.

### BYOM skills and quick-to-ideal migration

Keep portable skills provider-neutral: the skill owns workflow and evidence contracts, thin adapters bind harness tools, and the execution runtime owns model selection, credentials, retries, cancellation, and attribution. For a fast path plus a cleaner successor, freeze one versioned contract, spike both lanes in isolated worktrees, retain the quick adapter as fallback, and promote the ideal runtime only through an explicit compatibility gate.

Do not let parallel lanes independently edit the shared contract or become competing routing authorities. Separate approval-gated setup writes—tracker parent, child fan-out, and worktrees—so one timed-out approval does not obscure which mutations occurred.

Worked contract, packet fields, dual-track sequence, and promotion gate: `references/byom-skill-runtime-boundary.md`.

## Process

### Generate a whole-library skill map

When the user asks to map, audit, or visualize all agent skills, build a class-level knowledge map instead of a one-off prose summary:

1. Inventory canonical and shadow skill roots (`~/.hermes/skills`, `~/.claude/skills`, `~/.codex/skills`, vault-local `claude/skills`, known plugin skill-enhancer paths, and installed Claude plugin cache skill roots from `~/.claude/plugins/installed_plugins.json`). Exclude transient `~/.claude/jobs/**` snapshots. **`~/.codex/skills/` is a first-class source system** — Codex local skills (including AI Exec Circle community adapters from Ankit Patel, Brian Proffitt, Ant) live there and are invisible to `skills_list()`. Omitting it from the scan loses community-contributed skills that don't exist in Hermes or Claude Code.
2. Normalize physical `SKILL.md` instances into logical skills grouped by frontmatter `name`, retaining source system, category, hashes, support-file counts, tags, and explicit `related_skills`.
3. Generate a wikilinked Karpathy-pattern wiki (`index.md`, `log.md`, schema note, `skills/`, `clusters/`, `categories/`, `sources/`, `reports/`, `raw/`). This lets both Obsidian Smart Graph and Understand Anything consume the result.
4. Use Smart Graph extraction as a baseline for current Obsidian index health, then use Understand Anything's `understand-knowledge` parser/merger to create `.understand-anything/knowledge-graph.json`.
5. Verify with concrete counts: markdown files, graph nodes/edges/layers/tour steps, and zero dangling edges.

Session-specific recipe (initial generation): `references/agent-skills-map-understand-anything-20260621.md`.
Session-specific recipe (incremental maintenance — adding skills to an existing graph): same reference, § "Maintenance: adding skills to an existing graph".
Session-specific recipe (merge analysis — deciding which near-duplicates to merge): `references/skill-merge-analysis-20260622.md`.

### Deduplicate skills across source systems

When the user asks to ensure skills are not duplicative, or after generating a skill map:

1. Load the inventory from `raw/skill-inventory.json` (from the generated map).
2. Classify each skill by source system (Hermes default, Claude Code user, vault, plugin cache, pilot-sandbox).
3. Find duplicates: skills with the same frontmatter `name` appearing in multiple source systems.
4. Compute content hashes (SHA-256 of full SKILL.md text) for each instance. Group by:
   - **Identical duplicates** (same name, same hash across sources) — safe to remove non-canonical copies.
   - **Content drift** (same name, different hashes) — need manual review; Hermes version is canonical.
5. Detect near-duplicate names using `difflib.SequenceMatcher` with similarity > 0.85. These are usually distinct skills with similar names, not true duplicates.
6. Canonical source is `~/.hermes/skills/`. Remove non-Hermes copies from `~/.claude/skills/` and pilot-sandbox. Leave plugin cache copies (restored on reinstall) and vault copies (may be intentional) unless the user approves removal.
7. Clean up empty directories after removal.
8. Write a dedup report to `reports/dedup-report.md` in the map directory.

Pitfall: The `execute_code` Python sandbox may lack filesystem permissions to delete files outside its working directory. Use the `terminal` tool for file deletion operations.
Pitfall: Name similarity does not imply functional duplication. Always read actual SKILL.md descriptions and content before merging near-duplicate names. Pairs like `grill-me-agents` (greenfield design) vs `grill-me-with-agents` (existing-stack changes) look like merge candidates but serve explicitly different use cases. See `references/skill-merge-analysis-20260622.md`.
Pitfall: `*-workflow-a` / `*-workflow-b` naming is a plugin pack convention for complementary primary/secondary workflows (e.g. LangChain chains-vs-agents, Obsidian files-vs-UI), not a duplicate detection signal.
Pitfall: **Leave multiple copies if there is any risk of an agent not finding the canonical version.** The user's explicit preference: "leave multiple copies if there is any risk of an agent not finding the canonical version(s)." Removing duplicates from `~/.claude/skills/` is safe (agents fall through to Hermes). But pilot-sandbox copies, plugin cache copies, and vault copies serve agents that search those specific directories. Only remove copies when you can verify the agent will find the Hermes canonical version. When in doubt, leave it.
Pitfall: **Community-contributed skills in `~/.codex/skills/` are easily missed.** The AI Exec Circle (Nate B. Jones's community) has contributed skills from Ankit Patel (atlas-scout-concept-pipeline), Brian Proffitt (group-call-takeaway-synthesis), and Ant (session-close-reconciliation) that live only in Codex. Any whole-library map or dedup pass must include `~/.codex/skills/` as a scan root.

### 1. Search for shadow skills

When the user asks for a skill that `skills_list()` does not show:

```bash
find ~/Documents/pilot-sandbox/plugins/skill-enhancers -name "SKILL.md" -exec dirname {} \;
```

Also check:
- Other plugin directories the user has mentioned
- Shared vault paths (e.g. `=notes/.hermes/skills/` if vault-local skills exist)
- Any path the user explicitly points to

### 2. Register via symlink

For each shadow skill found:

```bash
cd ~/.hermes/skills
ln -s <full-plugin-path>/<skill-name> .
```

### 3. Verify registration

Run `skills_list()` and confirm the new skill appears. Then run `skill_view(name='<skill-name>')` to confirm content is readable.

### 4. Check for overlaps

Compare newly-registered skills against existing ones:
- Same name? → Load both via `skill_view()` and compare content. The plugin version may be more recent.
- Similar description? → Note the overlap to the user. The curator handles consolidation at scale.

### 5. Document discovery

Write a brief plan note in `~/.hermes/plans/` documenting:
- Which skills were found
- Where they lived
- Why they were invisible
- What was done to register them
- Any overlaps detected

## Known plugin paths (this environment)

| Path | Skills found |
|------|-------------|
| `~/Documents/pilot-sandbox/plugins/skill-enhancers/athenaeum/` | athenaeum-audit, athenaeum-design, athenaeum-ratify, athenaeum-reconcile, smart-graph |
| `~/Documents/pilot-sandbox/plugins/skill-enhancers/grill-each-other/skills/` | agent-show-and-tell, dialectic-vocabulary, fleet-ratify, grill-me, grill-me-agents, grill-me-with-agents, grill-with-docs, peer-grill, peer-grill-with-agents, permutation |
| `~/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/` | diagnose, improve-codebase-architecture, setup-matt-pocock-skills, tdd, to-issues, to-prd, triage, zoom-out |
| `~/.codex/skills/` | atlas-scout-concept-pipeline, chronicle, group-call-takeaway-synthesis, open-skills, session-close-reconciliation, plus shared copies of 11 Hermes skills (agents-sdk, cloudflare, etc.) |

### Promote community skills to Hermes canonical

When a community-contributed skill (AI Exec Circle, Open Skills registry, external repo) exists only in `~/.codex/skills/` or another non-Hermes harness, it is invisible to `skills_list()` and Hermes agents cannot load it. Promotion is a deliberate step, not automatic.

**Decision gate before promoting:**
1. Does the skill compose existing Hermes skills? (e.g. `atlas-scout-concept-pipeline` composes `atlas-chunking` + `scout-prioritization`). If yes, promotion adds value.
2. Is the skill harness-specific? If it references Codex-only internals (CLI flags, config paths), it may need adaptation before copying.
3. Does the user's paid policy apply? Verify the source is free before promoting. See `content-extraction-pipelines` Pattern E for the paid-auto-skip policy.

**Promotion steps:**
1. Read the source `SKILL.md` to determine the correct Hermes category (autonomous-ai-agents, note-taking, software-development, etc.).
2. `mkdir -p ~/.hermes/skills/<category>/<skill-name>/`
3. Copy `SKILL.md` (not symlink — community adapters may diverge from their Codex version as Hermes-specific adjustments are made).
4. Check frontmatter compatibility: Hermes skills may need a `version` field. Add if missing.
5. Verify: `skill_view(name='<category>/<skill-name>')` returns content.
6. Update OKF provenance docs (`okf/fleet/tools/ai-exec-circle-community-skills.md`) to reflect dual-install status.
7. Re-run the Understand Anything parse/merge to refresh `knowledge-graph.json` (see pitfall below).

**When NOT to promote:**
- The skill is a thin wrapper around a harness-specific CLI (e.g. `kimi-webbridge`, `codex-primary-runtime`).
- The skill duplicates an existing Hermes skill with no added value.
- The user hasn't approved promotion — ask first.

Reference: `references/community-skill-promotion-20260623.md`

## Cross-Machine Skill Import

When importing skills from a fleet peer host (e.g., Talaris → Aegis):

1. **Compare inventories**: SSH to the remote host and list all
   `SKILL.md` files under `~/.hermes/skills/`. Diff against the local
   inventory using `comm -23`.
2. **Evaluate candidates**: Read each remote-only SKILL.md. Check for
   functional duplicates under different category paths on the target
   (e.g., `productivity/fleet-knowledge-quiz` on Talaris vs
   `fleet/fleet-knowledge-quiz` on Aegis — same `name` field, different
   directory). Skip duplicates.
3. **Copy skill directories**: `scp -rq` the full skill directory
   (including `references/`, `templates/`, `scripts/`) from the remote
   host into the matching category path locally.
4. **Adapt host-specific paths**: `grep -rn` for the remote user's home
   path, vault paths, binary locations, and sync mechanisms. Patch each
   SKILL.md with the `patch` tool. Leave references to the remote host
   in explanatory text — those provide useful context for the fleet.
5. **Verify**: Confirm all imported skills have valid frontmatter, no
   unadapted executable paths remain, and `skill_view()` can load each
   imported skill.

Key path mappings vary by host pair. For Aegis ← Talaris:
- `~/Documents/=notes` → `~/=notes` (vault mirror)
- `~/Documents/=notes/.beads/` → `~/Documents/.beads/` (beads DB)
- `git pull` of `=notes` → `bd dolt pull` (sync mechanism)
- `~/.local/bin/bd` → `~/bin/bd` (binary location)

See `references/cross-machine-skill-import-20260701.md` for the full
worked example with path mapping table and per-skill adaptation log.

## Cross-Machine Skill Sync (User Preference)

### Jack's GitHub-first distribution convention

For Jack's fleet, an approved skill package must not exist only in a dirty `~/.hermes` runtime checkout or as ad-hoc cross-host copies.

1. Publish the exact package to **both** `JackReis/hermes-skills` and `JackReis/shared-agent-skills` for now — combine first and deduplicate the repositories later.
2. Use scoped commits containing only the intended skill package and any directly necessary registry/automation repair; never sweep unrelated `~/.hermes` changes into the commit.
3. Treat GitHub as the durable distribution source and installed default/profile trees on Talaris/Aegis as runtime projections.
4. Verify the remote `SKILL.md` and support-file hashes after push, confirm the pushed commit remains reachable if registry automation adds a follow-up commit, and verify each generated registry contains the skill.
5. If GitHub registry automation scans empty runner home directories and emits zero skills, fail closed and repair it with a deterministic repository-root mode; do not accept a successful workflow badge as registry-content proof.
6. After GitHub verification, project identical bytes to every discovered active profile on both machines and compare hashes.

After patching a skill, proactively synchronize every installed consumer, but discover each host's topology first.

### Procedure

1. **Inventory each host before acting**:
   - Is `~/.hermes/.git` present, and is the checkout compatible with the source host?
   - Which profile directories actually exist under `~/.hermes/profiles/`?
   - Does the destination skill package already exist in the default tree and each profile?
   - Does any repository or workspace instruction file alter the sync procedure?
2. **Publish the canonical source**. Commit and push the precise skill files when the source tree is Git-backed. This publishes the artifact; it does not prove the remote host can pull it.
3. **Choose the remote transport from evidence**:
   - Compatible Git checkout: fetch/pull after confirming target files are clean.
   - Non-Git runtime tree: transfer the exact skill package or changed support file to the verified destination.
4. **Enumerate profiles per host**. Never reuse local profile names on another machine. Copy or patch the skill into the profile directories actually discovered there.
5. **Prefer additive, non-destructive installation**. When the destination is absent, create parents and copy it directly. When it exists, compare content and use a targeted overwrite/patch. Avoid pre-emptive directory deletion.
6. **Keep approval-gated actions isolated**. Never combine a destructive replacement such as `rm -rf` with safe directory creation, transfer, or verification in one command. If approval times out, the entire compound action may be blocked and no safe work is completed.
7. **Verify consumers, not intent**. Read back a unique marker or compare hashes across the canonical copy, remote default copy, and every profile copy. Report any host not updated.

**Why profiles drift**: Profile skill directories are physical snapshots in many Hermes installations. Default-tree edits do not automatically propagate, and fleet hosts can use different profile names and storage layouts.

## Pitfalls

- **Cross-category duplicates.** The same skill may exist under different
  category paths on different hosts (e.g., `productivity/X` on one host,
  `fleet/X` on another). Compare by frontmatter `name` and functional
  description, not just by directory path.
- **`hermes skills search` does NOT search locally installed skills.** It only queries external registries (skills.sh, ClawHub, GitHub, etc.) and the optional-skills directory. Builtin skills (like `spike`, `plan`, `tdd`) and hub-installed skills are invisible to search even though they're installed and usable. This is a source-code gap — see `references/hermes-skills-search-local-fix.md` for the fix applied 2026-06-12.
- **Creating hard copies instead of symlinks.** Hard copies drift from the plugin source. Always symlink.
- **Ignoring overlaps.** Duplicate skill names between default and plugin trees can cause confusion about which version loads.
- **Not verifying.** A broken symlink or permission issue can make a skill appear in `skills_list()` but fail on `skill_view()`.
- **`hermes skills install` requires `--yes` for non-interactive use.** Batch installs (e.g., installing all 97 official optional skills) will fail silently without `--yes` / `-y` because each install prompts for confirmation. Always use `--yes` in scripts and automation.
- **Hermes source patches must be isolated from upstream until quintet review.** When patching Hermes source code (e.g., `tools/skills_hub.py`, `hermes_cli/`, `agent/`), do NOT commit or push to `NousResearch/hermes-agent` upstream until the Bifrost Quintet (Hermes, Antigravity, Claude Code, Codex, Kimi) has reviewed the change with full context. Keep the patch local. If an upstream `hermes update` overwrites it, re-apply from the handoff or reference doc. The quintet is the gate — no solo upstream pushes.
- **`execute_code` glob on `/var/folders` can crash with FileNotFoundError.** When scanning for persisted tool output files under macOS temp dirs, `Path('/var/folders').glob('**/...')` can hit race conditions on ephemeral pip-build-tracker directories. Use `search_files` (ripgrep-backed) or scope the glob to a known stable subdirectory like `/var/folders/<hash>/T/hermes-results/` instead of the full `/var/folders` root.
- **Forgetting `~/.codex/skills/` in skill inventories.** Codex has its own local skill tree at `~/.codex/skills/` that is invisible to both `skills_list()` and filesystem scans that only target Hermes/Claude paths. Community skill adapters (AI Exec Circle members like Ankit Patel, Brian Proffitt, Ant) are installed only in Codex. Any whole-library skill map must include `~/.codex/skills/` as a scan root or it will silently miss community-contributed skills.
- **Stale `knowledge-graph.json` after skill inventory changes.** When you add skill pages to the OKF agent-skills-map wiki (the four-layer maintenance pattern above), the markdown layer is immediately current for Obsidian/Smart Graph navigation. But `.understand-anything/knowledge-graph.json` is NOT auto-updated — it requires re-running the parse/merge scripts. A stale graph can persist for days (observed: 3 days stale Jun 20→23) without anyone noticing because the markdown looks fine. After any skill addition, removal, or promotion, re-run the Understand Anything parse/merge within the same session, or add a calendar reminder if deferring.

## References

- `references/plugin-skill-paths.md` — session-specific detail on plugin directories and skills found in this environment
- `references/skill-dedup-workflow-20260622.md` — dedup analysis technique, removal approach, and pitfalls for cleaning duplicate skills across source systems
- `references/codex-skills-discovery-20260622.md` — discovering `~/.codex/skills/` as a missing source root, including AI Exec Circle community adapters (Ankit Patel, Brian Proffitt, Ant)
- `references/community-skill-promotion-20260623.md` — promoting community skills from Codex-only to Hermes canonical; decision gate, steps, stale graph timing issue

## Consolidated sibling guidance (2026-06 umbrella pass)

### Former `skill-curation`

Skill library management includes curation and consolidation, not just path discovery. Prefer class-level skills with rich bodies and support files over one-session-one-skill micro-entries. During curation, inspect the complete skill package before archiving, preserve support files by archiving the full package or re-homing them, and record every consolidation in a structured summary so downstream references can migrate safely.
