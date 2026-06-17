# Reconciliation Model

Deep dive into proposal reconciliation, canonical aggregate recomputation, and grouped reconciliation.

→ Companion to the [README](../README.md) and [`architecture.md`](architecture.md).

---

## Why Reconciliation Exists

Not every proposed change maps trivially to a database update. The system must answer:

- Does this proposal update an existing log or create a new one?
- Should it join an in-flight lifecycle or start a new one?
- Is the target protected from automated mutation?
- After mutation, are aggregates and groups still consistent?

Reconciliation separates **decision** from **execution**. Decisions are pure and deterministic given the same inputs; execution is transactional.

---

## Proposal Reconciliation

The Investigator Agent performs pure, deterministic decision logic before any mutation. Each proposal resolves to one of four actions:

| Action | Condition |
|--------|-----------|
| `UPDATE_FAULT` | Exactly one hierarchy-compatible same-day candidate exists and is not protected |
| `ATTACH_TO_EXISTING` | No same-day candidate; an attachable open or valid lifecycle exists |
| `CREATE_FAULT` | No same-day candidate and no attachable lifecycle |
| `SKIP / REJECT` | Protected log, ambiguous match, hierarchy mismatch, or invalid proposal |

No database writes occur during decision resolution. The decision phase is a pure function of proposal, candidate log set, and available open lifecycles.

---

## Aggregate Recomputation

When validation status does not change between runs, materialization may route to a **noop** path — intentionally skipping redundant re-aggregation. However, operational field changes (occurrence windows, loss totals, date revisions) under the same validation status require **explicit canonical recomputation**:

1. Enumerate all sibling logs for the lifecycle
2. Deterministically recompute FS fields from the full sibling set using fixed reduction rules
3. Synchronize FSV from the recomputed FS
4. Operate independently of validation routing

Canonical recomputation is idempotent: the same log state always produces the same aggregate state.

**Delta-based aggregate patching is explicitly avoided.** Incrementally updating aggregates rather than recomputing from source is order-sensitive and non-idempotent at scale. Delta helpers are not safe general-purpose recomputation APIs.

---

## Same-Status Transitions

A same-status transition occurs when the proposed validation outcome equals the current FSV validation status. Materialization intentionally noops aggregate work for these transitions — re-runs should not redundantly re-aggregate.

Investigator UPDATE flows that change operational fields without changing validation status must invoke canonical recomputation within the same transaction after materialization returns noop. Grouped reconciliation still runs post-commit regardless.

---

## Grouped Reconciliation

Grouped reconciliation is a separate post-commit concern. It runs after transaction commit and:

- Evaluates whether active, valid FSVs should form or join a parent group
- Applies threshold logic based on asset configuration and resolved equipment counts
- Detaches invalid group memberships (children linked to unvalidated or closed parents)
- Updates parent FS and FSV summaries from children

**Failures in grouped reconciliation do not roll back the committed mutation.** A subsequent materialization on any affected log will retry grouping. Grouping consistency is eventual, not atomic.

---

## State Consistency Definition

FaultLog is the source of truth. FS and FSV are materialized views. Consistency means:

- Every log's lifecycle membership reflects its current attachment
- FS aggregates equal the deterministic reduction of sibling logs
- FSV operational fields mirror the paired FS after synchronization
- Grouping links reference valid parent records

Inconsistency arises when log fields change but aggregates are not recomputed, or when post-commit grouping fails after a successful transaction. The architecture addresses the first case explicitly; the second is eventual-consistency with retry on subsequent materialization.
