# Codex Automations → Hermes Cron Migration (2026-06-23)

## Context

User had two automations configured in Codex (OpenAI's app) and wanted to bring them into Hermes cron. The automations were discovered by OCR-ing screenshots of the Codex Automations UI (`20260623-130115-Codex@2x.png` and `20260623-130128-Codex@2x.png`).

## Source Automations

### 1. Daily Fleet Context Sync
- **Schedule**: Daily at 5:00 AM
- **Model**: GPT-5.4 High
- **Project**: =notes
- **Prompt summary**: Verify Hermes health, create/append daily note from template, sync context across OBn/OB1/ContextForge, populate project context for Sea Ranch AI / Fleet Infra / PAI, evidence block, fleet notification on failures.

### 2. Weekly notes digest
- **Schedule**: Fridays at 9:00 AM
- **Model**: GPT-5.4 Medium
- **Project**: =notes
- **Prompt summary**: Draft weekly digest from repo git history (last 7 days), prefer substantive commits over routine noise, never invent PR/MR markers, concise output.

## Migration Steps

1. **OCR the screenshots** using macOS Vision framework (`scripts/ocr_screenshot.swift`) — `vision_analyze` was down (401 auth error), so this was the fallback.

2. **Check for existing Hermes cron jobs** — `cronjob(action='list')` confirmed no existing "daily fleet context sync" or "weekly notes digest" jobs.

3. **Adapt health probes**:
   - Codex: `http://127.0.0.1:8642/health`, `http://127.0.0.1:8644/health`, `launchctl status for ai.hermes.gateway`
   - Hermes: `hermes doctor`, `hermes status --all`, `curl localhost:8642/health`, `launchctl list | grep ai.hermes`, `fleetctl status`

4. **Create Hermes cron jobs**:
   - Daily Fleet Context Sync: schedule `0 5 * * *`, deliver `origin`, toolsets `["terminal","file","web","skills"]`, workdir `=notes`
   - Weekly notes digest: schedule `0 9 * * 5`, deliver `origin`, toolsets `["terminal","file","web"]`, workdir `=notes`

5. **Model decision**: Left null to inherit session default (glm-5.2 via ollama-cloud). User was informed they could pin a specific model if desired.

## Key Adaptations

| Codex concept | Hermes equivalent |
|---|---|
| "Run now" button | `cronjob(action='run', job_id=...)` |
| Project binding | `workdir` parameter |
| Model (GPT-5.4 High/Medium) | `model` override (left null = inherit default) |
| Health probes (Codex-specific URLs) | `hermes doctor` + `hermes status --all` + native endpoints |
| Delivery (Codex UI) | `deliver` parameter (`origin` = current chat) |
| `pnpm volt tunnel 3141` | `npx --yes @voltagent/cli tunnel 3141` (see VoltAgent skill) |

## Outcome

Both jobs created successfully:
- Daily Fleet Context Sync: `4318b2425a20`, next run 2026-06-24 05:00 CT
- Weekly notes digest: `cb889d8d4af7`, next run 2026-06-26 09:00 CT