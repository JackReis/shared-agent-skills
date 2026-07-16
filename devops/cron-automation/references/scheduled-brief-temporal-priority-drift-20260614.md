# Scheduled Brief Temporal-Priority Drift (2026-06-14)

## Class of problem

Scheduled briefs can resurrect stale urgency when they aggregate old task/brief language without first resolving the newest authoritative task state.

This is especially likely when a task has many older "overdue" or "last chance" mentions, then receives a newer defer/postpone/resolution event. A naive brief generator treats all language as if it occurred at once and overweights repeated old urgency.

## Incident seed

A 2026-06-14 morning/midday brief surfaced Amy Thompson / Excel Therapy as a same-day "last decide-day" item. Jack corrected that this item had already been moved out one week. The valid state was deferred/revisit 2026-06-21.

The fix was not merely changing the task date; it required recording the correction in the sources the brief generator reads:

- ContextForge task due date/comment
- Vault task list
- Health/referral plan
- People/provider card
- The stale scheduled brief files themselves, with correction blocks
- A plan for a TLM-inspired temporal-priority auditor

## Durable lesson

Before emitting a scheduled brief top-priority item, resolve the entity's temporal state:

1. Gather dated evidence across tasks, comments, handoffs, scheduled briefs, daily notes, and source plans.
2. Identify the newest authoritative state-change event for the entity.
3. Classify current state: `active`, `due_now`, `deferred`, `future_watch`, `resolved`, or `stale_mention`.
4. If a newer defer/resolution exists, suppress older urgent language rather than repeating it.
5. If the generated brief already shipped stale guidance, patch the stale brief with a correction block so future retrieval sees the supersession.

## TLM-inspired framing

From Duderstadt & Helm's *A Model of the Language Process*: language is a process over time; models that ignore timestamps treat documents as if they all occurred at once and become anachronistic.

Apply this to briefs:

> Treat tasks, comments, handoffs, and briefs as dated language events. Priority is not mention frequency; it is current temporal validity.

## Suggested guardrail for brief jobs

For each proposed top-priority item, require a small temporal audit record:

```json
{
  "entity": "<task/entity>",
  "state": "deferred|active|due_now|resolved|stale_mention",
  "current_priority": true,
  "dominant_evidence": "newest authoritative event + timestamp",
  "suppressed_evidence": ["older conflicting urgency language"],
  "revisit_at": "YYYY-MM-DD or null"
}
```

If `state=deferred` and `revisit_at > now`, the item may appear only under "future watch" or "correction," never as a top action.

## Verification

After correction, verify all relevant sources contain the new state and date. A deterministic check should assert expected strings in all touched files and fail if any stale source remains uncorrected.
