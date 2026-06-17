# Validation Agent

Full input/output contract, run model, and boundary definitions for the Validation Agent.

→ Companion to the [README](../README.md).

---

## Goals

- Classify fault detections as valid, invalid, or inconclusive using automated analysis
- Persist validation outcomes to fault logs
- Trigger materialization so validation state propagates to FS and FSV
- Support scoped batch runs across configurable dimensions
- Allow human override without breaking downstream consistency

---

## Inputs

- Fault logs matching run scope (asset identity, inclusive start date, exclusive end date, optional fault-type filter, optional log-id subset)
- Asset and site context for metadata lookups
- Configuration for auto-validation exclusion and persistence behavior

---

## Outputs

- Updated fault log validation fields (status, attribution, timestamp, agent results, comments)
- Materialized FS and FSV state via the shared lifecycle platform
- Post-commit grouped reconciliation side effects
- Ephemeral run progress (no persisted run record in the current model)

---

## Boundaries

| Inside | Outside |
|--------|---------|
| Validation outcome assignment | FS/FSV routing and aggregation logic |
| Fault log field persistence (validation metadata) | Grouping engine |
| Run orchestration and progress reporting | Operational field mutation (occurrence revision, loss revision) |
| Handoff to materialization via standardized outcome objects | Investigator reconciliation and lifecycle creation |

---

## Run Model

Validation runs are **ephemeral and request-scoped**. There is no persisted run record; progress is observable during execution and terminal outcomes are reflected in updated fault logs and aggregates.

This differs meaningfully from the Investigator Agent run model, which requires persisted identity, batch semantics (per-asset, per-operational-day), action records, and revert linkage. Mirroring the validation shell pattern does not mean copying its persistence model.

---

## Human Override

Human overrides set validation metadata but do not mark logs as immutable. Investigator correction remains eligible after a human override. Protection is reserved for explicitly flagged records — for example, externally sourced or locked fault records. This ensures human validation does not unintentionally block subsequent automated correction.
