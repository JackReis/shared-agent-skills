# Skill Evolution and Cross-Harness Stamping

Use this reference when a request combines skill authoring, measurable self-improvement, and distribution across several agent harnesses.

## Layer boundaries

Treat these as separate systems with separate completion criteria:

| Layer | Responsibility | Completion evidence |
|---|---|---|
| Authoring | Produce a valid first `SKILL.md` from sources or a demonstrated workflow | Schema validation and human-readable review |
| Evolution | Compare baseline and candidates against task-specific evidence | Held-out improvement with constraints and regressions passing |
| Canon | Preserve authoritative bytes, versions, lineage, and rollback | Git-tracked package and provenance receipt |
| Distribution | Materialize the approved package for intended harnesses | Target-by-target install and visibility checks |
| Projection | Aggregate installed state into dashboards or registries | Regenerated output from scanner; never hand-edited generated data |

A prompt that creates a skill is not automatically an optimization engine. A scanner-generated registry is not automatically a publisher. An installer is not automatically the canonical source.

## Audit before design

1. Inspect the authority host or canonical repository first. Do not treat a convenient edge machine as canonical merely because it is available.
2. Search for the proposed skill under alternate names in canonical, profile, plugin, and cross-harness roots.
3. Read executable source and tests. Use READMEs and plans as intent evidence, not proof of implementation.
4. Separate working kernel/substrate status from product completion. A green runtime does not prove its packaging, handoff, promotion, or distribution work is complete.
5. Build an evidence table: implemented, partially implemented, planned, absent.
6. If another session is already implementing the same design, keep one lane read-only as an independent reviewer rather than creating competing artifacts.

## Evolution pipeline

For meaningful optimization, preserve this sequence:

1. **Baseline:** retain the current skill and establish its score.
2. **Dataset:** combine real failures/corrections, curated goldens, and synthetic cases. Keep training, validation, and untouched holdout splits distinct.
3. **Candidates:** use a deterministic optimizer where available; otherwise use independent model candidates and judges without pretending that review alone is statistical optimization.
4. **Constraints:** validate schema, size, semantic intent, platform accuracy, tool framing, secret hygiene, and any task-specific tests.
5. **Regression gates:** reject targeted improvements that damage broader behavior.
6. **Promotion:** require an explicit human approval boundary before canonical merge or public publication.
7. **Rollback:** retain the baseline, candidate lineage, metrics, and a reversible version reference.

Keep orchestration and state transitions outside the model when possible. The model proposes or judges content; deterministic code owns run state, checkpoints, receipts, validation, and promotion rules.

## Stamping pipeline

1. Produce one portable, versioned skill package as canon.
2. Adapt or install that package for each intended harness; do not maintain unrelated hand-written copies.
3. Verify both physical presence and runtime visibility after any required reload/restart.
4. Record installed, updated, declined, or incompatible outcomes per target.
5. Regenerate cross-harness registries from their scanners. Do not patch generated `registry.json` or dashboard payloads directly.
6. Report a host × harness matrix, including explicit verified `N/A` entries.

If the distribution system primarily pairs skills with generated CLIs, do not force pure procedural skills into a CLI-only schema. Add a generic skill-package route or a distinct pure-skill catalog type.

## Multi-host continuity

Canonical authority and execution availability are different concerns. A battery-backed edge can evaluate candidates or resume from exported checkpoints while a boss machine remains the promotion authority.

Do not place one single-writer SQLite database under concurrent mutation from multiple hosts. Exchange immutable checkpoint/evidence bundles or candidate branches; let the authority host accept and promote results.

## Verification checklist

- [ ] Canonical host/repository inspected before mirrors or edges
- [ ] Source and tests distinguished from README/plan claims
- [ ] Baseline retained and holdout data untouched during candidate generation
- [ ] Human promotion gate preserved
- [ ] Canonical package separated from harness adapters
- [ ] Every intended target verified for install and runtime visibility
- [ ] Generated registries rebuilt from scanners rather than edited
- [ ] Concurrent sessions assigned non-overlapping writer/reviewer roles
