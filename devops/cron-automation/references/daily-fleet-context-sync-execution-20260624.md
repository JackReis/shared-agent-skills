# Daily Fleet Context Sync — Execution Patterns (2026-06-24)

## Context

The "Daily Fleet Context Sync" cron job (origin: Codex Automation, migrated to Hermes cron
2026-06-23 as job `4318b2425a20`, schedule `0 5 * * *`) runs health probes, creates/appends
the daily note, syncs context across agents, and populates project links.

This reference captures the *execution* patterns and pitfalls discovered during the first
successful Hermes-cron run on 2026-06-24 10:05 CDT.

## Daily Note Path Discovery

**Correct path:** `/Users/jack.reis/Documents/=notes/daily/YYYY-MM-DD.md`

The template (`templates/daily-note-template-latest.md`) uses Templater syntax (`<% ... %>`)
and `YYYYMMDD.md` naming, but the actual notes live in the `daily/` subdirectory with
hyphenated ISO dates (`YYYY-MM-DD.md`). Prior sessions wasted time searching `calendar/day/`
and the vault root before discovering the real path.

**Discovery command:**
```bash
find /Users/jack.reis/Documents/=notes -maxdepth 1 -name "daily" -type d
ls /Users/jack.reis/Documents/=notes/daily/ | sort -r | head -5
```

## Append-Only Injection Pattern

The daily note is an append-only ledger. When it already exists (which is common —
other cron jobs or sessions may have created it earlier), use `patch(mode='replace')`
to find a unique anchor string and append after it. **Never use `write_file` on an existing
daily note** — it will destroy all prior entries (see pitfall #45 in SKILL.md).

**Anchor pattern:**
```python
# Find the last line of the existing note (typically a footer comment)
# and use it as the old_string in patch(mode='replace')
old_string = "*Cron: Claude settings audit runs every 4 hours..."
new_string = old_string + "\n\n---\n\n## 🔄 Daily Fleet Context Sync — ..."
```

Also update the frontmatter `touch_history` array using a second `patch` call to add
the cron run timestamp.

## ContextForge Quota Pre-Check

ContextForge has a free-tier limit of 500 queries/month (resets on the 1st). When exhausted,
`mcp_contextforge_memory_query` returns `quota_exceeded_queries` (HTTP 403). Do NOT retry —
it is not retryable.

**Pre-check pattern:**
1. Call `mcp_contextforge_memory_current_project` (does NOT count against quota)
2. Attempt one query. If it returns quota error:
   - Note the reset date (typically the 1st of next month)
   - Skip all L4 ContextForge operations
   - Fall back to OB1, OBn, Hindsight for knowledge graph queries
3. `mcp_contextforge_memory_list_items` and `mcp_contextforge_tasks_list` also fail — do NOT loop on them

**Fallback knowledge graph sources (when ContextForge is down):**

| Source | Tool | What it provides |
|--------|------|------------------|
| OB1 (Open Brain 1) | `mcp_open_brain_search_thoughts` | Semantic search of 25K+ thoughts/observations |
| OBn (ChromaDB) | `mcp_fleet_memory_fleet_memory_obn_search` or `mcp_fleet_memory_fleet_memory_status` | 58K+ vault docs indexed across 12 pods |
| Hindsight PostgreSQL | `mcp_fleet_memory_fleet_memory_status` | Durable facts with trust scores |
| Holographic (L1) | `mcp_fleet_memory_fleet_memory_fact_search` | Turn-sync entries with agent/session metadata |
| Fleet Memory MCP | `mcp_fleet_memory_fleet_memory_status` | Unified status for OB1/OBn/Hindsight in one call |

## Health Probe Sequence

Run all probes in parallel (they are independent):

```bash
hermes doctor
hermes status --all
curl -sf http://127.0.0.1:8642/health
launchctl list | grep ai.hermes
~/.hermes/bin/fleetctl status
```

If `fleetctl` is missing, skip it — the first 4 probes are sufficient for core health.
If `hermes doctor` or `hermes status` fail, do NOT proceed with context sync — focus on
recovery instead.

## Fleet-Directives File

The fleet directives file (`fleet-directives.md`) was not found in expected locations:
- `claude/mcp-coordination/state/fleet-directives.md` — does not exist
- `claude/coordination/fleet-directives.md` — does not exist

**Fallback:** Use `claude/coordination/inbox-fleet.md` (the fleet inbox) and
`LATEST_HANDOFF.md` (symlink to the latest handoff) as directive proxies. The inbox
contains dated entries from fleet agents with policy updates and status reports.

## Chronological Fleet Brief

Run the chronological brief script for current fleet activity:
```bash
python3 ~/.hermes/scripts/chronological_brief.py --table-only
```

This queries the Holographic fact_store for recent [TURN] entries and outputs a markdown
table with agent, harness, model, session, and last-turn snippet. See the
`turn-sync` skill and `chronological_brief.py` for details.

## Project Context Population

Three projects need context links populated in the daily note:

1. **Sea Ranch AI** — source: OB1 search, active-projects-index, fleet-memory turn-sync facts
2. **Fleet Infrastructure** — source: OB1 search, fleet inbox, git log, fleet-memory facts
3. **Personal AI Workspace (PAI)** — source: OB1 search, MEMORY.md, memory-sop skill

For each project, include: status, latest update, key documents (wikilinks), next action,
and a sample artifact.

## OB1 Session Capture

After completing the daily note update, capture a summary thought to OB1:
```python
mcp_open_brain_capture_thought(
    content="[DONE] Daily fleet context sync YYYY-MM-DD ...",
    source="hermes-cron",
    task_id="daily-fleet-context-sync-YYYYMMDD"
)
```

Verify the capture by searching for it:
```python
mcp_open_brain_search_thoughts(
    query="daily fleet context sync YYYY-MM-DD",
    task_id="daily-fleet-context-sync-YYYYMMDD",
    limit=1
)
```

## Evidence Block

The final output must include a concise evidence block with:
- Daily note path (and line number of the sync section)
- Health probe results (all 5 probes)
- Knowledge graph sources used (with item counts)
- Project links added (3 projects with doc links)
- Verifier output (grep, wc, OB1 readback)
- Caveats (what was skipped, what was missing, what was degraded)