# Plugin Skill Paths — Session Detail

Author: Hermes — session 2026-05-31

## Environment

Host: macOS 15.7.5
User: jack.reis
Default skill tree: `~/.hermes/skills/`
Plugin root: `/Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/`

## Discovery command used

```bash
find /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers -name "SKILL.md" -exec dirname {} \; | sort
```

## Skills found and registered

### athenaeum/ (fleet governance)

| Skill | Path | Description |
|-------|------|-------------|
| athenaeum-audit | `athenaeum/athenaeum-audit` | Code-aware agent-stack audit + reconcile |
| athenaeum-design | `athenaeum/athenaeum-design` | Rigorous design grilling for agent stacks |
| athenaeum-ratify | `athenaeum/athenaeum-ratify` | Fleet-wide attestation with dissent recorded |
| athenaeum-reconcile | `athenaeum/athenaeum-reconcile` | Divergent understanding reconciliation |
| smart-graph | `athenaeum/skills/smart-graph` | Vault note relationships, backlinks, orphans |

### grill-each-other/skills/ (dialectics)

| Skill | Path | Description |
|-------|------|-------------|
| agent-show-and-tell | `grill-each-other/skills/agent-show-and-tell` | Fleet visibility via independent reports |
| dialectic-vocabulary | `grill-each-other/skills/dialectic-vocabulary` | Scholastic/Greek vocab for peer-grill |
| fleet-ratify | `grill-each-other/skills/fleet-ratify` | N-agent sign-off on artifacts |
| grill-me | `grill-each-other/skills/grill-me` | Grill user on plan/design |
| grill-me-agents | `grill-each-other/skills/grill-me-agents` | Grill multi-agent collaboration |
| grill-me-with-agents | `grill-each-other/skills/grill-me-with-agents` | Code-aware grill of existing stack |
| grill-with-docs | `grill-each-other/skills/grill-with-docs` | Grill against domain model, update docs inline |
| peer-grill | `grill-each-other/skills/peer-grill` | File-based dialectical debate |
| peer-grill-with-agents | `grill-each-other/skills/peer-grill-with-agents` | Two+ agents audit stack, then reconcile |
| permutation | `grill-each-other/skills/permutation` | Ratify NxN fleet relationship matrix |

### pocock-engineering/skills/ (engineering workflow)

| Skill | Path | Description |
|-------|------|-------------|
| diagnose | `pocock-engineering/skills/diagnose` | Bug/perf diagnosis loop |
| improve-codebase-architecture | `pocock-engineering/skills/improve-codebase-architecture` | Deepening opportunities |
| setup-matt-pocock-skills | `pocock-engineering/skills/setup-matt-pocock-skills` | Bootstrap AGENTS.md context |
| tdd | `pocock-engineering/skills/tdd` | Red-green-refactor TDD |
| to-issues | `pocock-engineering/skills/to-issues` | Break plans into issues |
| to-prd | `pocock-engineering/skills/to-prd` | Conversation → PRD |
| triage | `pocock-engineering/skills/triage` | State-machine issue triage |
| zoom-out | `pocock-engineering/skills/zoom-out` | Broader context perspective |

## Overlaps with default tree

The following plugin skills duplicate names already in `~/.hermes/skills/software-development/`:

| Plugin Skill | Default Skill | Notes |
|-------------|---------------|-------|
| grill-me | `software-development/grill-me` | May differ in content depth |
| grill-me-agents | `software-development/grill-me-agents` | May differ in content depth |
| grill-me-with-agents | `software-development/grill-me-with-agents` | May differ in content depth |
| grill-with-docs | `software-development/grill-with-docs` | May differ in content depth |
| peer-grill | `software-development/peer-grill` | May differ in content depth |
| peer-grill-with-agents | `software-development/peer-grill-with-agents` | May differ in content depth |

If `skill_view()` loads a different version than expected, check whether the symlink or the default tree version is being resolved first.

## Registration commands

```bash
cd ~/.hermes/skills
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/athenaeum/athenaeum-audit .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/athenaeum/athenaeum-design .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/athenaeum/athenaeum-ratify .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/athenaeum/athenaeum-reconcile .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/athenaeum/skills/smart-graph .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/grill-each-other/skills/agent-show-and-tell .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/grill-each-other/skills/dialectic-vocabulary .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/grill-each-other/skills/fleet-ratify .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/grill-each-other/skills/permutation .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/diagnose .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/setup-matt-pocock-skills .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/tdd .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/to-issues .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/to-prd .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/triage .
ln -s /Users/jack.reis/Documents/pilot-sandbox/plugins/skill-enhancers/pocock-engineering/skills/zoom-out .
```

## Result

`skills_list()` count increased from 144 to 160 skills. All newly-registered skills pass `skill_view()` verification.
