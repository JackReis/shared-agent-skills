# Team Canonical Skills Architecture — 2026-06-14

Session context: Jack asked whether Paperclip should be the canonical home for team skills, then asked to run Aegis's per-repo agent-instructions proposal past Gemini, Codex, and Claude.

## Consensus policy

Use **Git-backed Markdown** as the canonical source for team skills and repo-local contracts.

- `SKILL.md` libraries are canonical for reusable procedures and class-level workflows.
- Repo-root `AGENTS.md` is canonical for repo-local execution rules after scan-first validation.
- Paperclip company skills are an operational registry/distribution surface, ideally imported or synced from Git-backed skills.
- ContextForge/Open Brain stores durable decisions, rationale, semantic recall, and cross-repo relationships; it does not arbitrate byte-level skill content.
- Hermes global memory stores Jack-level preferences and stable operator facts, not repo-specific commands.
- iCloud folders and Alfred snippets are human convenience layers only; never treat them as canonical agent policy.

## Rollout rule

Do **not** blanket-generate `AGENTS.md`/skill files across every repo. Instead:

1. Inventory existing instruction surfaces: `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `.cursor/rules`, `README`, `Makefile`, CI config, package scripts, test configs.
2. Prioritize active/high-risk repos first.
3. If a richer repo-local file already exists, make `AGENTS.md` a thin pointer/adapter rather than a duplicate.
4. Pilot on 2–3 repos, verify actual consumption by Hermes/Codex/Claude/Gemini/Paperclip, then expand.
5. Add verification metadata and lint checks before broad rollout.

## Guardrails

- One canonical repo-local instruction file per repo; derived files must say they are derived or point back to canon.
- No secrets, decrypted env, private topology, Tailnet/IP details, or credentials in committed instructions.
- Prefer pointers to runtime sources of truth (`Makefile`, `package.json`, CI, `pytest.ini`) over transcribed commands.
- If a documented command fails and the corrected command is discovered, update the instruction file before claiming completion.
- Keep files short; link to skills and canonical docs for detail.

## Useful phrasing

> Git is the canonical skill source. Paperclip is the operational registry. Alfred/iCloud are convenience mirrors/snippets only.

> Standardize on Git-backed repo-root `AGENTS.md` as the default repo-local contract, but roll it out through scan-first prioritization.
