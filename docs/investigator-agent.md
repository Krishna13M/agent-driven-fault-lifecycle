# Investigator Agent

Full reconciliation decision model, mutation lifecycle, revert design, and audit trail for the Investigator Agent.

→ Companion to the [README](../README.md) and [`reconciliation.md`](reconciliation.md).

---

## Goals

- Analyze telemetry and existing fault state to produce operational correction proposals
- Reconcile each proposal to the correct lifecycle action before any mutation occurs
- Apply mutations transactionally with a complete audit trail
- Support revert of applied actions via snapshot restoration
- Operate at batch granularity (per-asset, per-operational-day), not per-row triggers

---

## Inputs

- Structured operational correction proposals: occurrences, loss, dates, fault metadata, validation status
- Candidate fault logs from repository queries: identity, operational date, hierarchy compatibility
- Attachable open lifecycles when no same-day candidate exists
- Asset validation metadata and hierarchy context

---

## Outputs

- Mutated or created fault logs
- Materialized FS and FSV state via the shared lifecycle platform
- Canonical aggregate recomputation when validation status is unchanged but operational fields changed
- Investigator action records with pre-mutation and post-mutation snapshots
- Post-commit grouped reconciliation side effects

---

## Boundaries

| Inside | Outside |
|--------|---------|
| Proposal generation (external package) | FS/FSV routing, delta aggregation, date recomputation |
| Reconciliation decisioning (pure, deterministic) | Grouped reconciliation implementation |
| Fault log mutation and fault metadata derivation | Validation LLM prompting |
| Lifecycle transaction orchestration | Duplicate materialization paths |
| Snapshot capture and revert | Dashboard presentation of FS/FSV topology |

---

## Reconciliation Actions

Before any mutation, pure deterministic logic classifies each proposal:

| Action | Condition |
|--------|-----------|
| `UPDATE_FAULT` | Exactly one hierarchy-compatible same-day candidate exists and is not protected |
| `ATTACH_TO_EXISTING` | No same-day candidate; an attachable open or valid lifecycle exists |
| `CREATE_FAULT` | No same-day candidate and no attachable lifecycle |
| `SKIP / REJECT` | Protected log, ambiguous match, hierarchy mismatch, or invalid proposal |

---

## Revert Design

Investigator revert restores prior log state via snapshot and recomputes aggregates. It does not reconstruct historical FS/FSV graph shapes — it removes the effect of the mutation from the present system state. This is sufficient because aggregates are always re-derivable from current log state.

Revert is supported only for investigator mutations, not for validation outcomes or human overrides.

---

## Run Model

Investigator runs require **persisted identity and action records**, including revert linkage. This is a fundamentally different model from the ephemeral validation run. Each investigator run operates at per-asset, per-operational-day batch granularity, producing action records for every applied mutation that can be individually reverted.

---

## Audit Trail

Every applied investigator action records:

- Pre-mutation snapshot of the affected fault log
- Post-mutation outcome
- Reconciliation action taken (UPDATE / ATTACH / CREATE)
- Revert linkage for rollback eligibility

Post-commit audit persistence failures are logged and visible — never silently swallowed. The committed mutation stands; audit gaps are repaired by retry.
