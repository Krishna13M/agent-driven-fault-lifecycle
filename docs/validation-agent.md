# Validation Agent

Architecture of the validation agent — decision generation, validation workflow, ownership boundaries, and materialization handoff.

The validation agent answers one question: *Is this fault detection credible?* It assigns a validation outcome to individual fault logs and hands off to the shared lifecycle platform for all aggregate state.

---

## Role in the system

The validation agent is a **producer**, not an owner of lifecycle topology. It classifies detections and persists validation metadata on fault logs. FS and FSV materialization, routing, aggregation, synchronization, and grouping belong to the shared lifecycle platform.

```text
Fault logs in scope
        │
        ▼
Validation Agent  ──outcome──▶  Shared Lifecycle Platform  ──▶  FS / FSV
```

Human operators follow the same materialization path when overriding validation — the platform does not branch on actor identity.

---

## Goals

- Classify fault detections as valid, invalid, or inconclusive
- Persist validation outcomes to fault logs with attribution
- Trigger materialization so validation state propagates to aggregate views
- Support scoped batch runs across plants, date ranges, and fault types
- Allow human override without breaking downstream consistency

---

## Inputs

- Fault logs matching run scope (plants, date range, optional fault-name filter, optional log subset)
- Plant and company context for metadata lookups
- Configuration for auto-validated fault exclusion and persistence behavior

---

## Outputs

- Updated fault log validation fields (status, validated_by, validated_at, agent results, comments)
- Materialized FS and FSV state via the shared lifecycle platform
- Post-commit grouped reconciliation side effects

Progress reporting is ephemeral in the current model — the durable audit trail lives on the fault log itself.

---

## Ownership boundaries

| Inside validation agent | Outside validation agent |
|-------------------------|--------------------------|
| Validation outcome assignment | FS/FSV routing and aggregation |
| Validation metadata on fault logs | Grouping engine |
| Run orchestration and progress reporting | Operational field mutation (occurrences, loss revision) |
| Standardized outcome handoff to materialization | Investigator reconciliation and lifecycle creation |

---

## Materialization handoff

Every validation outcome crosses a single boundary: the per-log materialization entry point. The handoff contract is a fault log plus a validation outcome — status, attribution, and optional comments.

The platform then:

1. Persists validation fields on the log
2. Selects a route based on validation transition and existing topology
3. Mutates FS/FSV within the primary transaction
4. Runs grouping after commit

The validation agent does not choose routes, recompute aggregates, or invoke grouping. Re-running validation on an already-settled log may route to noop when status is unchanged — aggregate work is skipped unless operational fields also changed (handled by the platform, not the agent).

---

## Relationship to investigation

Validation and investigation are orthogonal concerns on the same log:

- A log can be validated and later revised operationally without re-validation
- Both agents converge on the same materialization path
- Equivalent log fields and validation outcomes must produce equivalent downstream state (**agent parity**)

---

## Related documents

- [Lifecycle Platform](internal/lifecycle-platform.md) — shared materialization platform
- [Transaction Model](transaction-model.md) — consistency guarantees at the handoff boundary
- [Design Principles](design-principles.md) — agent parity and materialization boundaries
- [System Brain](internal/system-brain.md) — full architecture reference
