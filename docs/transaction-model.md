# Transaction Model

How the shared lifecycle platform maintains consistency when mutating fault lifecycle state — from agent-produced outcomes through aggregate materialization to post-commit grouping.

**Audience:** Backend engineers, data engineers, software architects.

**Scope:** Platform consistency guarantees. Agent reasoning and orchestration internals are out of scope except at the materialization handoff boundary.

---

## Purpose

Every lifecycle mutation must leave the platform in a coherent state: fault logs, operational summaries (FS), and validated summaries (FSV) must agree at commit, and grouping must eventually reflect the new topology.

The transaction model defines:

- What work runs atomically inside the primary database transaction
- What work runs before and after that transaction
- What consistency is guaranteed at each phase
- How failures are handled without corrupting aggregate state

The platform treats **FaultLog as source of truth** and FS/FSV as **derived materialized views**. The transaction model exists to keep those views derivable from log state at every commit boundary.

---

## Why Transaction Boundaries Matter

A single lifecycle event can touch multiple documents: one or more fault logs, an FS, an FSV, and potentially a second FS/FSV pair when a log moves between lifecycles. Partial completion — log updated but aggregates stale, or FS updated but FSV out of sync — produces state that cannot be explained from log data alone.

The primary transaction exists to make these mutations **atomic**:

| Without boundaries | With boundaries |
|--------------------|-----------------|
| Log validation persisted but FS not created | Log and aggregates commit together or roll back together |
| Log moved to new FS while old FS still counts it | Membership change and both sides' aggregates reconcile in one unit |
| FSV reflects old loss while log carries new loss | FS and FSV synchronized before commit |

Work that cannot be made atomic without unacceptable cost — cross-lifecycle grouping, external metadata lookups — is deliberately deferred to post-commit with explicit eventual-consistency semantics.

---

## High-Level Flow

```text
┌─────────────────────────────────────────────────────────────────────────┐
│  PRE-TRANSACTION                                                        │
│  Load log, identity, current FS/FSV, target FSV candidate, date stats   │
│  Determine route                                                        │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PRIMARY TRANSACTION                                                    │
│                                                                         │
│  1. Persist validation fields on fault log                              │
│  2. Execute route-specific FS/FSV mutations (or noop)                   │
│  3. Canonical aggregate recompute (conditional — same-status noop)    │
│  4. Synchronize FSV from FS                                             │
│                                                                         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ commit
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  POST-COMMIT                                                            │
│  Build grouping context → reconcile old FSV → reconcile target FSV        │
└─────────────────────────────────────────────────────────────────────────┘
```

When an outer orchestrator participates in the transaction (for example, operational log mutation before materialization), agent-owned log writes and platform materialization share the same session so reads within the transaction see uncommitted log changes.

---

## Pre-Transaction Phase

Before opening the primary transaction, the platform performs read-only (or occasionally corrective) preparation:

| Read | Purpose |
|------|---------|
| **Fault log** | Current membership, operational fields, validation state |
| **Equipment identity** | Resolve hierarchy for lifecycle matching (string → combiner → MPPT → inverter, or plant-level) |
| **Current FS** | Lifecycle the log is attached to via membership reference |
| **Paired FSV** | Validation status and grouping state for the current FS |
| **Target FSV by identity** | Candidate lifecycle for join when outcome status matches an existing open lifecycle |
| **Sibling date statistics** | Min/max fault dates among logs on old (and target) FS, excluding the current log where appropriate |
| **Prior parent link** | Grouping parent reference before mutation, for post-commit parent reevaluation |

**Route determination uses only pre-transaction state.** The route is fixed before the transaction opens so the transaction body executes a known plan without re-deriving topology mid-write.

A limited pre-transaction correction may normalize FS projected-loss fields when stored values are inconsistent with past-loss-derived expectations. This runs outside the primary transaction boundary and is conservative — failures do not abort the lifecycle flow.

---

## Route Determination

Routing is driven by the **proposed validation outcome** and **existing lifecycle topology**, not by which agent produced the outcome.

Evaluation proceeds in order:

1. **Unsupported or inconclusive outcomes** → noop (validation metadata may still be persisted; no FS/FSV topology change)
2. **Same status as paired FSV** (already valid or already invalid) → noop
3. **Joinable target FSV exists** for the resolved identity and proposed status → join (log moves to target lifecycle)
4. **No sibling logs remain on current FS** (excluding current log) → convert in place (FS gains or updates FSV without creating a new FS)
5. **Otherwise** → create new (new FS and FSV; log moves; old FS handled after removal)

The join path includes a validity guard: for valid outcomes, a target FSV whose fixing date precedes the log's fault date is not considered joinable.

| Route | Topology effect | Aggregate work |
|-------|-----------------|----------------|
| **Noop** | None | Skipped (validation fields only) |
| **Convert in place** | FSV created or updated on existing FS | Route body recomputes and syncs |
| **Create new** | New FS + FSV; log membership changes; old FS cleaned up | Route body handles both sides |
| **Join** | Log moves to existing FS; old FS cleaned up | Route body patches target and source |

---

## Primary Transaction

All materialization mutations execute inside a single database transaction when a replica set supports multi-document transactions. On standalone deployments, the same callback runs without session isolation — callers accept weaker atomicity guarantees in that environment.

### Log Mutation

The transaction always begins by persisting validation outcome fields on the fault log: status, validated-by attribution, timestamps, and human-override metadata when applicable.

For noop routes, this is the only write. The transaction commits with updated log validation state but unchanged FS/FSV aggregates.

When an outer orchestrator has already mutated operational log fields (occurrences, loss, dates) in the same transaction, materialization sees those writes through the shared session.

### Materialization

After log validation persistence, the platform executes the route-specific transaction body:

- **Convert in place** — create FSV if absent, or update FSV validation fields; synchronize operational fields from FS to FSV
- **Create new** — create new FS and FSV from log and prior FS context; reassign log membership; recompute or remove the vacated old FS
- **Join** — reassign log membership to target FS; incrementally update target FS aggregates; clean up old FS after log departure

Route bodies use transition-aware aggregate helpers for addition and removal. These are appropriate when validation status changes drive topology change. They are not used on noop routes.

### Aggregate Recompute

When routing returns noop but operational log fields changed under an unchanged validation status, route-body aggregate helpers do not run. The platform invokes **canonical recomputation** within the same transaction:

1. Enumerate all sibling logs for the lifecycle (deterministic ordering)
2. Derive FS aggregates from the full sibling set — loss, detection date, fault date, actionable
3. Update FS fields
4. Synchronize FSV from the recomputed FS

Canonical recomputation is skipped when:

- The route is non-noop (the route body already handled aggregates)
- The lifecycle is a grouped parent record
- The mutation action does not require operational field propagation (lifecycle-creation paths)

When the sibling set is empty after recomputation, the platform may delete orphaned FS and FSV records. This is expected lifecycle cleanup, not a consistency error.

Canonical recomputation is **idempotent**: given the same sibling log set, the derived aggregates are always identical.

### FS/FSV Synchronization

After route-body work or canonical recomputation, the platform ensures FSV operational fields mirror the paired FS. Synchronization copies shared lifecycle fields — dates, losses, hierarchy, resolution — from FS into FSV while preserving FSV-owned validation and grouping fields.

Synchronization runs inside the primary transaction so FS and FSV are pairwise consistent at commit.

---

## Commit Guarantees

When the primary transaction commits successfully:

| Guarantee | Description |
|-----------|-------------|
| **Log durability** | Validation outcome fields are persisted |
| **Membership correctness** | Log `fault_status_id` reflects the lifecycle chosen by the route |
| **Aggregate accuracy** | FS fields reflect either route-body computation or canonical sibling reduction |
| **FS/FSV parity** | Paired FS and FSV agree on shared operational fields |
| **Atomic scope** | All primary-transaction writes succeed or none do |

What commit does **not** guarantee:

- Updated grouping parent-child links
- Parent aggregate dates across grouped children
- Audit record persistence for external orchestrators

Those are post-commit concerns with separate failure semantics.

---

## Post-Commit Grouping

Grouped fault reconciliation runs **after** the primary transaction commits, with no database session bound to the prior transaction.

### Execution model

1. **Reload FSV state** for the old FS and, when applicable, the target FS after membership change
2. **Build grouping context once** — plant validation metadata, equipment input counts, per-fault thresholds — to avoid repeated external store access
3. **Reconcile the old FSV** — detach, reattach, promote to parent, or dissolve group membership as eligibility changes
4. **Reconcile the target FSV** when it differs from the old FSV — same logic on the post-mutation lifecycle

When the old FSV no longer exists after a noop or topology change but a prior parent link was recorded, the platform may reevaluate that parent directly.

### Grouping inputs

Grouping evaluates FSV-level state only: validation status, actionable flag, grouped flag, linked-to reference, feedback presence, resolution status. It does not read per-log occurrence data.

Certain fault types are excluded from grouping entirely. Eligible faults use plant-configured thresholds based on equipment counts at the resolved hierarchy level.

### Failure behavior

Grouping failures are caught and logged. They do not roll back the committed primary transaction. A subsequent materialization on any affected log will retry grouping.

---

## Failure Semantics

| Failure point | Behavior |
|---------------|----------|
| Pre-transaction read failure | No writes; error propagated to caller |
| Primary transaction failure | Full rollback of log, FS, and FSV changes within the transaction |
| Canonical recompute failure inside transaction | Same rollback as any transaction failure |
| Post-commit grouping failure | Committed mutation stands; grouping state may be stale |
| Grouping context build failure | Grouping skipped; per-lifecycle state remains correct |
| Post-commit audit/event failure | Committed mutation stands; audit gap logged |

There are **no compensating transactions** for post-commit failures. Eventual consistency for grouping relies on retry via subsequent lifecycle events.

Concurrent materialization on the same equipment identity is serialized through a per-identity lock in the asynchronous reconcile dispatch path, preventing race conditions on shared FS/FSV documents.

---

## Idempotency and Retries

| Mechanism | Idempotency property |
|-----------|---------------------|
| **Noop routing** | Re-applying the same validation outcome when status is already settled skips aggregate work |
| **Canonical recomputation** | Full sibling enumeration produces the same FS/FSV state for the same log set |
| **Route-body aggregation** | Intended for single transition events; not safe as a general-purpose retry primitive |
| **Grouping reconciliation** | Re-running on the same FSV state converges membership to the correct cluster |

Safe retry strategy:

- **Inside transaction:** retry the full primary transaction; partial retries of individual steps are unsafe
- **After commit:** retry post-commit grouping by re-invoking materialization post-commit phase or a dedicated grouping repair
- **Operational corrections with unchanged status:** must include canonical recomputation in the transaction; retrying materialization alone on noop is insufficient if recompute was skipped

Re-running materialization with identical inputs on an already-settled lifecycle typically noops at the routing layer, making blind retries low-risk for validation re-runs but insufficient for operational field corrections unless recompute is explicitly triggered.

---

## Design Tradeoffs

### Grouping outside the primary transaction

Grouping requires external metadata (plant configuration, equipment counts), performs multiple independent writes across parent and child records, and may touch FSVs unrelated to the triggering log. Including it in the primary transaction would:

- Extend transaction duration and lock contention
- Couple lifecycle correctness to external store availability
- Risk rolling back valid per-lifecycle mutations because an unrelated parent's grouping step failed

The tradeoff is **strong per-lifecycle consistency at commit** versus **eventual grouping consistency** afterward. The platform chooses the former.

### Pre-transaction reads with fixed routing

Routing decisions use snapshot state from before the transaction. This prevents mid-transaction route changes that could cause write ordering bugs. The tradeoff is that extreme concurrency on the same identity is handled by serialization rather than optimistic re-routing.

### Standalone database fallback

When multi-document transactions are unavailable, the platform executes the same logical callback without session isolation. Correctness depends on single-writer assumptions and per-identity serialization. Production deployments with consistency requirements should use replica-set transactions.

### Noop routing versus operational updates

Noop routing optimizes validation re-runs by skipping redundant aggregate work when status is unchanged. The tradeoff is an explicit second path — canonical recomputation — for operational corrections that do not change validation status. This is simpler than making noop routing always re-aggregate, which would penalize every validation re-run.

### Projected-loss normalization before transaction

A pre-transaction FS correction for projected-loss consistency may write outside the primary transaction. The tradeoff accepts a narrow window where FS projected loss is normalized before log validation commits, in exchange for consistent downstream aggregate behavior.

---

## Key Takeaways

1. **One primary transaction** owns log validation persistence, FS/FSV topology changes, canonical recomputation, and FS-to-FSV synchronization.
2. **Pre-transaction reads** fix the route before any mutation, making the transaction body predictable.
3. **Routing is transition-driven** — validation status change selects create, join, convert, or noop; actor identity is irrelevant.
4. **Noop is intentional but incomplete** — operational field changes under unchanged status require canonical recomputation inside the same transaction.
5. **FS and FSV are consistent at commit** — synchronization is always part of the primary transaction's successful path.
6. **Grouping is eventual** — post-commit, non-blocking, retriable; failures do not undo committed lifecycle state.
7. **Canonical recomputation is the safe aggregate primitive** — full sibling enumeration is idempotent; incremental delta patching is reserved for route-body transition events.
8. **Per-identity serialization** prevents concurrent writers from corrupting shared lifecycle documents.
9. **Agent producers stop at the materialization boundary** — everything that shapes aggregate topology and grouping executes under platform rules, regardless of outcome origin.

---

## Related documents

- [`docs/internal/lifecycle-platform.md`](internal/lifecycle-platform.md) — conceptual model of the shared lifecycle platform
- [`docs/internal/system-brain.md`](internal/system-brain.md) — canonical architecture reference
- [`docs/post_validation_pipeline.md`](post_validation_pipeline.md) — pipeline entry points and membership rules
