# Daily HTML Output Publishing Pattern (2026-06-20)

## Context

A daily Nightly Forge / dashboard workflow needed to do more than write HTML locally: Jack asked to "publish the html using surge.sh or otherwise present the outputs each day."

The working class-level pattern is a **script-only Hermes cron job** that publishes only artifacts that pass a conservative privacy scan, and otherwise delivers local paths and caveats.

A follow-up approval-gated Mission Control pass showed the companion pattern for operational dashboards: keep them local/private by default, but when Jack explicitly replies **"Approve all"** to a scoped action gate, capture that approval durably, run verification, publish a Surge preview, verify by HTTP readback, then commit/push the approval + regenerated artifact evidence.

## Pattern

1. Keep the content-generating job separate from the publishing/presentation job.
   - Example: the 2 AM Forge builds artifacts.
   - A later no-agent cron job publishes/presents the latest output.
2. Use `no_agent=True` / `--no-agent` for the publisher so the script stdout is the exact user-facing daily message.
3. Discover the latest artifact from a stable index or pointer.
   - Prefer an index such as `nightly-forge/index.json` with `latest` and per-entry `path`, `readme`, `handoff` fields.
4. Before any public publish, run a conservative text scan over the HTML.
   - Block obvious tokens, `Authorization`/`Bearer`, known secret env names, session-token headers, private calendar/email markers, email addresses, and private-key markers.
5. If the scan passes and the static host CLI is authenticated, publish and verify the public URL with HTTP readback.
6. If publish fails or scan blocks, **do not stay silent**. Deliver the local artifact path, README/handoff paths, and the reason public publishing was skipped.
7. Treat sensitive operational dashboards separately from sanitized daily artifacts.
   - A dashboard containing fleet topology, queues, internal hostnames, or dispatch packets should be rebuilt/presented locally unless an operator explicitly approves public publishing.

## Approval-Gated Operational Dashboard Publish Pattern

Use this when the artifact is an operational dashboard or queue/agent control surface rather than a sanitized daily briefing.

1. **Ask/receive scoped approval by action group.** The gate should spell out categories such as read-only verification, local regeneration, dry-run helpers, and mutation/publish/commit actions.
2. **When the user replies "Approve all," treat it as approval for all named action groups only.** Do not expand the approval beyond the prompt's scope.
3. **Capture the approval durably before acting.** Write a handoff/reference file that includes:
   - source surface (for example Telegram DM)
   - exact source message (for example `Approve all`)
   - timestamp
   - thread/session ID
   - each approved action group in full
   - remaining guardrails (verification still required)
4. **Run verification before public exposure.** Minimum for this class:
   - generator `py_compile`
   - focused tests
   - local regeneration command
   - static secret scan over the publish directory
   - snapshot/readback of readiness claims
5. **Publish as a Surge preview first** when the dashboard contains operational/private context.
   - Example shape: `surge public/byom-mission-control-local byom-mission-control-local.surge.sh --preview`
   - Surge prints a timestamped preview host such as `https://1781991589215-byom-mission-control-local.surge.sh/`.
6. **HTTP-read the preview and check semantic markers.** Do not claim publish success from the Surge CLI line alone.
7. **Retry HTTP readback once or a few times for preview propagation.** A first request may transiently return `504 Gateway Time-out`; if the Surge command succeeded, retry with a short delay before declaring publish failure.
8. **Commit/push only the scoped evidence and regenerated artifacts.** Avoid staging unrelated dirty files in the shared vault.

## Example user-facing stdout

```text
🌅 Daily HTML outputs — 2026-06-20 08:52 CDT
Latest Forge: Weekend Stewardship Console (2026-06-20)
Self-contained local dashboard for protecting a family-heavy Saturday...
Local HTML: /Users/jack.reis/Documents/=notes/nightly-forge/2026-06-20/artifact/index.html
Published: https://nightly-forge-talaris.surge.sh/
README: /Users/jack.reis/Documents/=notes/nightly-forge/2026-06-20/README.md
Handoff: /Users/jack.reis/Documents/=notes/nightly-forge/2026-06-20/handoff.md
BYOM Mission Control: rebuilt local static app → /Users/jack.reis/Documents/=notes/public/byom-mission-control-local/index.html
BYOM public Surge publish remains operator-gated because that snapshot can include private fleet/queue topology.
```

## Verification

Minimum fresh verification before claiming success for the daily sanitized artifact:

```bash
python3 ~/.hermes/scripts/daily-html-output-presenter.py
python3 - <<'PY'
import urllib.request
url='https://nightly-forge-talaris.surge.sh/'
with urllib.request.urlopen(url, timeout=20) as r:
    body=r.read().decode('utf-8', errors='replace')
print('status=200')
for needle in ['Weekend Stewardship Console', 'Nightly Forge', 'Campaign Guardrail', 'Monday Launch']:
    print(f'{needle}={needle in body}')
PY
```

Minimum fresh verification before claiming success for an approval-gated operational dashboard preview:

```bash
python3 -m py_compile scripts/mission_control/build_surge_app.py
python3 -m pytest tests/scripts/test_build_surge_app.py -q
python3 scripts/mission_control/build_surge_app.py \
  --out public/byom-mission-control-local \
  --registry okf/fleet/tools/openskills/registry.json \
  --beads-dir .beads \
  --surge-domain byom-mission-control-local.surge.sh
rg "__HERMES_SESSION_TOKEN__|X-Hermes-Session-Token|Hermes-Session|session token|LINEAR_API_KEY|sk-[A-Za-z0-9]" public/byom-mission-control-local
surge public/byom-mission-control-local byom-mission-control-local.surge.sh --preview
# Then curl the exact preview host that Surge prints and check expected markers.
```

Also verify the cron entry exists with the intended schedule and `no_agent=True` for unattended daily publishing.

## Pitfalls

- Do not publish private operational dashboards just because the latest artifact is HTML. Classify the artifact first.
- Do not let a failed public publish become silence; the requirement is "publish **or otherwise present** outputs."
- Do not rely on a host CLI success message alone. Fetch the public URL and check expected page markers.
- Do not treat approval as verification. Approval removes the human gate; tests, secret scans, and readback still decide whether it is safe to report completion.
- Do not panic on a single post-Surge `504` from a timestamped preview. Retry briefly; preview propagation can lag the CLI success line.
- Do not combine content generation, publication, and sensitive-dashboard export in one opaque LLM cron prompt; script-only stdout is easier to audit and cheaper to run.
