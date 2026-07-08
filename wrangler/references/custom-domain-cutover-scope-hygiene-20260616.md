# Custom-domain cutover scope hygiene — Sally Workers lesson (2026-06-16)

## Context

During a Cloudflare Workers custom-domain cutover plan, the legacy plan carried five hostnames:

- `sally.oursearanchhome.com`
- `guest-resources.oursearanchhome.com`
- `bill.oursearanchhome.com`
- `ask.oursearanchhome.com`
- `sally.jack.digital`

The user corrected the plan: "We don't need bill. Or ask." The real production goal was to remove the laptop SPOF for the canonical guest surfaces, not preserve every historical alias.

## Durable lesson

For Workers custom-domain migrations, do not blindly carry every legacy hostname as a required cutover target. Reconfirm the canonical production surfaces and turn stale aliases into optional redirect cleanup. This prevents the plan from spending dashboard/DNS attention on domains that no longer matter.

## Pattern

1. Separate hostnames into:
   - **Canonical production targets** — required for Definition of Done.
   - **Legacy aliases** — optional redirect/deprecation cleanup.
   - **Split-account / special-routing domains** — explicitly out of the core cutover unless required.
2. Rewrite success criteria around the canonical pair/surface only.
3. Rewrite rollback around only the required cutover targets.
4. Keep optional aliases in a "future cleanup" section, not in the critical path.
5. Persist the scope decision in the plan/runbook and memory so future agents do not resurrect stale aliases as blockers.

## Verification phrases to search after editing

Search the plan for stale scope language such as:

- `all five`
- `five hostnames`
- `secondaries first`
- `Task 4.13`
- `total rollback = all five`
- deprecated alias names that should no longer be in success criteria

If any remain, either remove them or explicitly mark them as optional cleanup.

## Sally-specific outcome

The plan was narrowed to:

- Required: `sally.oursearanchhome.com`, `guest-resources.oursearanchhome.com`
- Not required: `bill.oursearanchhome.com`, `ask.oursearanchhome.com`
- Optional cleanup: `sally.jack.digital`

A remaining separate gate was model-provider drift: the preview Worker was healthy but used OpenAI while the original plan said Ollama-only. That is an architecture-contract decision, not a hostname-scope blocker.