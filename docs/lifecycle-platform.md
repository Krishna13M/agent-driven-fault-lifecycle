# Shared Fault Lifecycle Platform

Agents produce **outcomes** — validation judgments or operational corrections — at the fault-log level. The shared lifecycle platform is what turns those outcomes into durable **aggregate platform state**: FaultStatus (FS) and FaultStatusValidated (FSV). Every agent converges here; none of them owns aggregate topology.

```text
Agent outcome (validation or operational)
        │
        ▼
┌───────────────────────────────────┐
│  Shared Lifecycle Platform        │
│  materialize → sync → (recompute) │
└───────────────┬───────────────────┘
                │ commit
                ▼
        post-commit grouping
                │
                ▼
   FS / FSV aggregate state
```

## Why This Platform Exists

Without a shared lifecycle boundary, each agent would need to independently decide lifecycle ownership, compute aggregates, synchronize operational and validated views, and maintain grouping state. That duplication creates drift: the same log-level outcome could produce different platform state depending on who wrote it.

The platform exists to prevent that drift. It centralizes lifecycle ownership behind one materialization boundary so all outcomes follow the same rules for routing, aggregation, synchronization, and post-commit grouping. The result is consistent aggregate state, predictable transitions, and a clear separation between agent-produced outcomes and platform-owned lifecycle state.

---

## Entity ownership

The model is deliberately asymmetric: one writable source, two derived views.

**FaultLog** is the source of truth. Agents and operators write here — validation status, operational fields (occurrences, loss, dates), equipment identity, audit metadata. Membership in a lifecycle is expressed only by reference (`fault_status_id`), not by embedded lists on FS/FSV.

**FaultStatus (FS)** is platform-owned. It is the operational summary for a lifecycle: aggregated loss, detection and fault dates, equipment identity, resolution state, severity. FS reflects the deterministic reduction of all sibling logs sharing that lifecycle.

**FaultStatusValidated (FSV)** is also platform-owned. It mirrors FS for shared operational fields and adds validation-aware and grouping-aware state: `validated_status`, `actionable`, `linked_to`, `grouped`, `feedback`. Consumers that need "is this fault credible and how does it cluster?" read FSV.

```text
FaultLog (many) ──fault_status_id──▶ FaultStatus (one)
                                         │
                                         ▼ sync
                                  FaultStatusValidated (one)
```

Agents never independently create, route, aggregate, or synchronize FS/FSV. Their only lifecycle obligation is to produce a well-formed log-level outcome and hand it to the platform.

Field ownership is further split between FS and FSV at the aggregate layer:

| Layer | Owns |
|-------|------|
| **FS** | Operational lifecycle: dates, losses, hierarchy, resolution, severity |
| **FSV** | Validation and grouping: validated status, actionable, linked_to, grouped, feedback |

`actionable` is treated as FSV-owned in practice, even though it originates from log-level classification during materialization.

---

## Lifecycle materialization

Materialization is the per-log entry point. Given a fault log and an outcome, the platform:

1. **Persists** validation fields on the log
2. **Routes** based on the validation transition and existing topology
3. **Mutates** FS/FSV according to the selected route
4. **Synchronizes** FSV from FS so paired records agree at commit

Routing is driven by **validation-state transitions**, not by which agent invoked materialization. The platform resolves equipment identity (string → combiner → MPPT → inverter, or plant-level when none apply) and inspects whether a paired FSV already exists for the log's lifecycle.

| Route | Meaning |
|-------|---------|
| **Create** | New lifecycle — new FS and FSV |
| **Join** | Log attaches to an existing FSV for the same identity and status |
| **Convert** | Validation status changed — FSV converted in place or lifecycle reshaped |
| **Noop** | Proposed status equals current FSV status — validation metadata persisted, aggregate work skipped |

Human overrides follow the same path as agent validation. Operational corrections that arrive with a validation status already set also flow through materialization; the platform does not branch on actor.

Materialization runs inside the **primary transaction** alongside any log writes. FS and FSV are consistent with each other when the transaction commits.

---

## State transitions

Two transition types matter, and they interact differently with routing.

**Validation transitions** change credibility (`valid` / `invalid` / `inconclusive`). These drive routing: a pending log becoming valid may create a lifecycle; a valid log becoming invalid may convert or detach. This is the primary path FS/FSV topology changes.

**Same-status transitions** keep validation status unchanged. Materialization intentionally routes to **noop** — re-running validation on an already-settled log should not redundantly re-aggregate. Noop persists validation metadata but skips loss summation, date recomputation, and FS-to-FSV sync.

Same-status noop creates a gap when **operational fields change under unchanged validation status** — for example, revised occurrences or loss without re-validation. Routing alone will not propagate those changes to FS/FSV. The platform closes this with **canonical recomputation**: enumerate all sibling logs for the lifecycle, re-derive FS aggregates from the full set, sync FSV. This runs inside the same transaction, independent of routing. It is idempotent — unchanged logs yield unchanged aggregates.

When the last sibling log leaves a lifecycle, the platform may delete orphaned FS and FSV. That is expected cleanup, not an integrity failure.

---

## Lifecycle Scenarios

The following scenarios illustrate how materialization, routing, and recomputation interact in common cases.

### New valid fault

A fault log arrives with no existing lifecycle membership and a validation outcome of **valid**.

Materialization routes to **create**: a new FS and paired FSV are materialized from the log. Aggregates reflect that single log's operational fields. After commit, grouped reconciliation evaluates whether this FSV should join or form a parent group with other active, valid faults of the same type.

Whether the outcome originated from validation or from an operational correction with status already set, the platform path is the same — parity is preserved.

### Valid → invalid transition

A log currently on a **valid** lifecycle receives a validation outcome of **invalid**.

Materialization routes to **convert**: the FSV and FS topology reshapes to reflect the status change. Aggregate fields may be recalculated as part of the route body because the validation transition itself triggers aggregate work — this is not a noop.

After commit, grouped reconciliation runs for both the prior and target FSV. If the fault was grouped, parent membership and parent aggregates may be updated or the child may be detached.

### Same-status operational update

A log's validation status remains **valid**, but operational fields change — revised occurrences, updated loss, or shifted dates.

Materialization routes to **noop** because the proposed status matches the current FSV status. Validation metadata is persisted, but aggregate fields on FS and FSV are not updated by routing alone.

Canonical recomputation runs within the same transaction: all sibling logs for the lifecycle are enumerated, FS aggregates are re-derived from the full set, and FSV is synchronized from the recomputed FS. After commit, grouped reconciliation proceeds as usual.

---

## Aggregate consistency

Consistency is defined relative to FaultLog as source of truth.

At **commit**, the platform guarantees:

- Every log's `fault_status_id` reflects its lifecycle membership
- FS aggregates equal the deterministic reduction of attached sibling logs
- FSV operational fields mirror the paired FS
- Log and aggregate state for the affected lifecycle are **atomically** consistent

The platform does **not** guarantee immediate grouping consistency. Grouping runs post-commit and may lag briefly.

Consistency breaks in two known ways:

1. **Operational change + noop without recompute** — log updated, aggregates stale. Addressed by canonical recomputation within the transaction.
2. **Post-commit grouping failure** — log and FS/FSV are correct, but parent-child links may be stale until the next materialization retries grouping.

Incremental delta patching of aggregates is avoided for general use because it is order-sensitive and non-idempotent. Full sibling enumeration is the safe model.

**Why canonical recomputation over delta patching:** Delta patching applies a change relative to the previous aggregate state — add this log's loss, subtract that log's contribution, adjust dates by comparison. That approach assumes a known prior state and a single orderly sequence of changes. When multiple logs mutate, when the same lifecycle is updated from different entry points, or when a correction revises fields that were already aggregated, deltas compound errors and become sensitive to processing order. Canonical recomputation discards the assumption of incremental history: it reads the current sibling log set and derives aggregates from scratch every time. Given the same log state, the result is always the same — making the operation safe to retry, safe to run after noop, and safe to use when reverting or replaying mutations.

Concurrent materialization on the same equipment identity is serialized so two writers cannot race on the same FS/FSV.

---

## Grouping semantics

Grouping is a **separate phase** from materialization. It runs **after commit**, outside the primary transaction, for both the prior and target FSV affected by a materialization event.

**Purpose:** cluster related valid, active faults of the same type under a parent record to reduce alert noise and surface systemic plant issues.

**Structure:** a parent FS + parent FSV, with child FSVs referencing the parent via `linked_to`. A valid parent has both FS and FSV; an unvalidated parent (FS only) is invalid — children linked to it are detached during reconciliation.

**Eligibility:** grouping reads FSV state only — `validated_status`, `actionable`, `grouped`, `linked_to`, `feedback`, `is_resolved`. It does not use per-log occurrences or validation timestamps. A child must be valid and active (unresolved, no feedback).

**Threshold logic:** group creation and survival depend on plant configuration and equipment counts at the resolved hierarchy level. A minimum of two members is always required. Certain fault types are excluded entirely.

**Failure semantics:** grouping failures are logged, not compensated. The committed mutation stands. Any subsequent materialization on an affected log will retry grouping. Consumers must tolerate brief lag between correct per-lifecycle state and updated group membership.

---

## How agents interact (and where they stop)

Both agents are **producers**. Their interaction with the lifecycle is narrow:

| Agent contribution | Handoff to platform |
|--------------------|---------------------|
| **Validation** | Assigns validation outcome → standardized outcome object → materialization |
| **Human override** | Same handoff with human attribution |
| **Investigation** | Writes operational and/or validation fields on the log → materialization (and canonical recompute when status is unchanged but economics changed) |

Neither agent chooses routing, performs FS aggregation, syncs FSV, or runs grouped reconciliation. Investigation may create or attach logs before invoking the platform, but lifecycle topology is always resolved by materialization, not by the agent.

The design requirement is **parity**: a log processed through validation materialization and an equivalent log arriving from investigation should produce the same downstream FS/FSV state, given the same log fields and validation outcome.

---

## Design Principles

The rules below are implicit throughout this document. They govern how the platform behaves and how agents must interact with it.

- **Log as source of truth** — FaultLog state is authoritative; FS and FSV are derived and always re-derivable from sibling logs.
- **Single materialization path** — All agents converge on one per-log entry point; no parallel aggregate construction.
- **Platform owns aggregates** — Agents write log-level outcomes; routing, aggregation, synchronization, and grouping belong to the platform.
- **Transition-driven routing** — FS/FSV topology changes are driven by validation-state transitions, not by which agent invoked materialization.
- **Noop is intentional, not sufficient alone** — Same-status routing skips redundant aggregate work; operational changes under unchanged status require canonical recomputation.
- **Atomic log–aggregate consistency at commit** — FS and FSV agree with each other and with attached logs when the primary transaction commits.
- **Grouping is eventual** — Post-commit grouping may lag; failures do not roll back committed mutations.
- **Deterministic downstream processing** — Given the same log state and outcome, materialization and recomputation produce the same aggregates.
- **Agent parity** — Equivalent log fields and validation outcomes yield equivalent downstream state regardless of agent origin.

---

## Transaction phases (summary)

```text
PRIMARY TRANSACTION
  log writes (agent)
  → materialize (route or noop)
  → canonical recompute? (same-status operational change)
  → FS ↔ FSV sync
COMMIT

POST-COMMIT
  → grouped reconciliation (old + target FSV)
```

The boundary between agents and platform state is the materialization call. Everything above that line is agent domain; everything below is platform domain. Aggregate state is never authoritative until materialization commits.
