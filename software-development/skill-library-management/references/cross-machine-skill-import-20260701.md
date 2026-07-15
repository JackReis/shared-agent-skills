# Cross-Machine Skill Import (Aegis ← Talaris, 2026-07-01)

Session-specific reproduction of importing skills from a fleet peer host
and adapting them for the target machine's paths and conventions.

## Context

Aegis (Mac Mini, user `hermes`) and Talaris (MacBook Pro, user `jack.reis`)
are fleet peers. Talaris had 9 skills Aegis didn't. 7 were imported, 2
skipped as duplicates. 3 required path adaptation.

## Workflow

### 1. Compare skill inventories

```bash
# Local skills (Aegis)
find ~/.hermes/skills/ -maxdepth 3 -name "SKILL.md" \
  | sed "s|$HOME/.hermes/skills/||" | sed 's|/SKILL.md||' | sort

# Remote skills (Talaris via SSH)
ssh jack.reis@Talaris 'find ~/.hermes/skills/ -maxdepth 3 -name "SKILL.md" \
  | sed "s|$HOME/.hermes/skills/||" | sed "s|/SKILL.md||" | sort'
```

### 2. Compute the diff

```bash
comm -23 <(echo "$remote_skills") <(echo "$local_skills")  # only on remote
comm -12 <(echo "$remote_skills") <(echo "$local_skills")  # on both
```

### 3. Evaluate each candidate

For each skill only on the remote host:
- Read the SKILL.md to assess relevance to the target host
- Check for duplicates under different names or category paths on the
  target (e.g., `productivity/fleet-knowledge-quiz` on Talaris vs
  `fleet/fleet-knowledge-quiz` on Aegis — same skill, different category)
- Skip if a functional equivalent already exists

### 4. Copy skill directories

```bash
# Create target directories
mkdir -p ~/.hermes/skills/<category>/<skill-name>

# SCP each skill directory from the remote host
scp -rq "jack.reis@Talaris:~/.hermes/skills/<category>/<skill-name>/"* \
  "$HOME/.hermes/skills/<category>/<skill-name>/"
```

### 5. Adapt paths for the target host

Identify all host-specific references in SKILL.md files:
- User home paths: `/Users/jack.reis/` → `/Users/hermes/`
- Vault paths: `~/Documents/=notes` → `~/=notes` (Aegis local mirror)
- Binary locations: `bd` at `~/bin/bd` (not `~/.local/bin/bd`)
- Sync mechanisms: git pull of `=notes` → `bd dolt pull` (Dolt sync)
- Service architecture differences: launchd labels, port numbers

Use `grep -rn` to find all instances:
```bash
grep -rn "jack.reis\|/Users/jack\|~/Documents/=notes" \
  ~/.hermes/skills/<imported-skill>/
```

Patch each SKILL.md with the `patch` tool. Leave references to the
remote host in explanatory text (e.g., "the canonical vault is on
Talaris at ~/Documents/=notes") — those provide useful context.

### 6. Verify all imported skills

```bash
for skill in <list>; do
  f="$HOME/.hermes/skills/$skill/SKILL.md"
  if [ -f "$f" ]; then
    name=$(grep "^name:" "$f" | head -1)
    echo "OK: $skill — $name"
  else
    echo "MISSING: $skill"
  fi
done
```

Also verify no unadapted host-specific paths remain in executable
commands or path tables. References to the remote host in explanatory
text are fine.

### 7. Check for cross-category duplicates

Skills may exist under different category paths on different hosts.
Always compare by frontmatter `name` and by functional description,
not just by category/skill path. Example: `productivity/fleet-knowledge-quiz`
(Talaris) vs `fleet/fleet-knowledge-quiz` (Aegis) — same `name` field,
different directory structure.

## Path Mapping Table (Aegis ← Talaris)

| Talaris path | Aegis path | Notes |
|---|---|---|
| `/Users/jack.reis/Documents/=notes` | `/Users/hermes/=notes` | Vault mirror |
| `~/Documents/=notes` | `~/=notes` | Vault mirror (relative) |
| `~/Documents/=notes/.beads/` | `~/Documents/.beads/` | Beads DB location |
| `cd ~/Documents/=notes && git pull` | `~/bin/bd dolt pull` | Beads sync mechanism |
| `~/.local/bin/bd` | `~/bin/bd` | Beads binary (old one broken) |

## Skills imported this session

| Skill | Adapted? | Notes |
|---|---|---|
| apple/macos-launchd-troubleshooting | No | Generic, references Aegis already |
| autonomous-ai-agents/hindsight | No | Already references Aegis paths |
| note-taking/okf-knowledge-graph | No | Generic OKF structure |
| productivity/beads | Yes | Beads dir, sync mechanism, binary path |
| productivity/daily-briefing-assembly | Yes | =notes path, Obsidian Vault refs removed |
| productivity/email-campaign-coordination | No | Generic workflow |
| productivity/fleet-context-sync | Yes | =notes path throughout |

## Skills skipped (duplicates)

| Talaris skill | Aegis equivalent | Match basis |
|---|---|---|
| productivity/fleet-knowledge-quiz | fleet/fleet-knowledge-quiz | Same frontmatter name |
| software-development/nightly-forge | software-development/nightly-personal-forge | Same function, different name |