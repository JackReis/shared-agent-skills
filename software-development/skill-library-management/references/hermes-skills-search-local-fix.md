# Hermes Skills Search — Local Skill Source Fix (2026-06-12)

## Problem

`hermes skills search <query>` only searched external registries (skills.sh, ClawHub, GitHub, Claude Marketplace, LobeHub, Browse.sh) and the optional-skills directory. It never scanned `~/.hermes/skills/` — the directory where builtin, hub-installed, and local skills actually live.

This meant builtin skills like `spike`, `plan`, `test-driven-development`, and `writing-plans` were invisible to search even though they were installed and usable via `skill_view()` and `skills_list()`.

## Root Cause

`create_source_router()` in `tools/skills_hub.py` (line ~3736) built a list of 10 source adapters — none of them scanned the local skills directory:

```python
sources = [
    OptionalSkillSource(),        # Official optional skills
    HermesIndexSource(auth=auth), # Centralized index
    SkillsShSource(auth=auth),
    WellKnownSkillSource(),
    UrlSource(),
    GitHubSource(auth=auth, extra_taps=extra_taps),
    ClawHubSource(),
    ClaudeMarketplaceSource(auth=auth),
    LobeHubSource(),
    BrowseShSource(),
]
```

Additionally, `unified_search()` returned results in thread-completion order with no relevance sorting, so even if a local result existed it would be buried among 35+ external results.

## Fix Applied

Two changes to `/Users/jack.reis/.hermes/hermes-agent/tools/skills_hub.py`:

### 1. New `LocalSkillSource` class

Scans `~/.hermes/skills/` and exposes all installed skills to the search pipeline. Implements the full `SkillSource` interface: `search()`, `fetch()`, `inspect()`, `source_id()`, `trust_level_for()`.

Added as the **first** source in `create_source_router()` so it runs on every search:

```python
sources: List[SkillSource] = [
    LocalSkillSource(),           # Installed skills (builtin + hub + local) — highest priority
    OptionalSkillSource(),        # Official optional skills
    HermesIndexSource(auth=auth), # Centralized index
    # ... rest unchanged
]
```

### 2. Result ordering in `unified_search()`

Sorts results so installed/local skills always appear before external catalog noise:

```python
def _sort_key(meta: SkillMeta) -> tuple:
    is_local = 0 if meta.source == "local" else 1
    name_match = 0 if meta.name.lower() == query_lower else 1
    trust = -_TRUST_RANK.get(meta.trust_level, 0)
    return (is_local, name_match, trust, meta.name.lower())
```

Also added `"local": 3` to `_TRUST_RANK` so local skills win dedup conflicts.

## Verification

- `hermes skills search spike` → spike is result #1 (was: 0 relevant results)
- `hermes skills search plan` → plan is result #1
- `hermes skills search tdd` → 3 local skills appear before external entries

## Batch Install Pitfall

`hermes skills install` requires interactive confirmation by default. When batch-installing all 97 official optional skills, every single one failed because the script didn't pass `--yes` / `-y`. Always use `--yes` for non-interactive installs.

## Related

- `skill-library-management` skill — covers skill discovery, registration, and the `skills_list()` vs `skills_search()` gap
- `hermes-agent` skill — CLI reference, `hermes skills` subcommands

## Quintet Isolation Rule

This patch is **local only** — not committed or pushed to `NousResearch/hermes-agent` upstream. Per the Bifrost Quintet protocol, any Hermes source-code change must be reviewed by the full quintet (Hermes, Antigravity, Claude Code, Codex, Kimi) with complete context before going upstream. If `hermes update` overwrites this patch, re-apply from the handoff at `session-hermes-skills-master-learner-20260612.md`.
