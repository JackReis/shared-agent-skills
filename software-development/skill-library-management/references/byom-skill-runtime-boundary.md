# BYOM Skill and Runtime Boundary

Use this pattern when a portable workflow skill must run across multiple model providers or agent harnesses.

## Stable separation

- **Portable skill:** owns workflow intent, inputs, outputs, evidence requirements, and a versioned result contract. It names no preferred provider and stores no credentials.
- **Harness adapter:** maps abstract workflow operations to the tools available in one runtime. Keep it thin and replaceable.
- **Execution runtime:** owns engine/model selection, authentication, subprocess or API lifecycle, cancellation, retries, token accounting, and model attribution.
- **Registry or OKF projection:** records provenance and readiness. It is not the runtime and must not become a second routing authority.

A skill may accept an externally selected engine/model as provenance, but it must not maintain a competing model-routing table when the execution runtime already owns routing.

## Versioned context packet

For evidence-gathering or context-refresh workflows, exchange a versioned packet rather than concatenating untracked prose. At minimum record:

- schema version and snapshot timestamp;
- scope and source identifiers;
- source observation times and content hashes;
- claims linked to evidence IDs;
- contradictions and unresolved uncertainty;
- rendered briefing or context text;
- freshness or expiry policy;
- runtime-attested engine, model, and run identity;
- packet hash and failure disposition.

The packet is an artifact, not an authority store. Never place task queues, durable facts, secrets, or provider credentials inside it.

## Quick-to-ideal dual track

When a user wants a fast path now and a cleaner architecture later, spike both from one frozen contract:

1. Freeze the packet schema and shared fixture vectors before implementation fan-out.
2. Create isolated worktrees from the same proven baseline.
3. Give one lane the **quick adapter**: fit the current CLI/process boundary with minimal core refactoring.
4. Give the other lane the **ideal runtime**: introduce a provider protocol or middleware while preserving the current adapter as compatibility behavior.
5. Keep contract files under one controller-owned path. Lanes may vendor an exact generated copy but must not independently redesign it.
6. Require producer-to-consumer tests using the same serialized fixtures; separate green unit suites are insufficient.
7. Retain the quick path as the default/fallback until the ideal path passes the full compatibility, containment, cancellation, raw-log, attribution, and integration suites.
8. Switch through an explicit config/feature gate. Do not perform a flag-day replacement.
9. Record a verdict for each spike: validated, partial, or invalidated, with executable evidence.

## Approval-gated setup

Do not bundle tracker creation, child-task fan-out, worktree creation, and other writes into one approval-gated command. Perform and verify them in small stages: parent record first, children second, worktrees third. If approval times out, stop and wait for fresh user consent; do not retry or route around the gate.

## Promotion gate

Promote the ideal path only when all are true:

- the shared contract is unchanged or intentionally version-bumped;
- old CLI/provider behavior remains covered by compatibility tests;
- exact quick-packet fixtures are accepted by the ideal consumer;
- runtime identity is harness-attested rather than model self-declared;
- no second queue, fact store, credential store, or routing authority was introduced;
- rollback to the quick path is one documented configuration change;
- canonical skill, adapters, registry projections, and runtime readback all agree on version and hash.
