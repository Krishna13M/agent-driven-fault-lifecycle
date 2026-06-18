# Design Principles

Architectural principles that shaped the fault lifecycle platform. This document explains *why* the system is structured as it is — not how to execute individual workflows or implement specific components.

**Source material:** validation agent boundaries, shared lifecycle platform model, transaction model.

---

## Source of Truth vs Derived State

### Problem it solves

Fault management spans two levels of abstraction: individual detections (logs) and operational summaries (aggregates). Without a declared authority, writers compete — agents update summaries directly, aggregates drift from log data, and replay or revert become impossible because no single layer owns the facts.

### Tradeoff

Treating logs as authoritative means aggregate views are always one step behind the write path. Every correction must flow log → aggregate rather than patching summaries in place. The system accepts this latency within a transaction in exchange for explainability: any FS or FSV state must be derivable from its sibling logs.

### How it appears in the architecture

FaultLog holds operational and validation facts at the detection level. FaultStatus and FaultStatusValidated are materialized views — reduced, synchronized summaries of log sets linked by lifecycle membership. Recomputation, replay, and revert operate by restoring or mutating log state and re-deriving aggregates. Membership is by reference, not embedded lists on aggregate documents.

---

## Decision Generation vs State Ownership

### Problem it solves

Automated agents must analyze, propose, and decide — but allowing every agent to own persistent lifecycle state produces incompatible topologies. Validation might create an FS one way; investigation another. The platform needs a clean line between *what an agent concludes* and *what the system commits*.

### Tradeoff

Agents are limited to producing outcomes at the log level. They cannot directly shape aggregate topology, routing, or grouping. This constrains agent autonomy but eliminates duplicate lifecycle logic. Decision quality remains agent-specific; state shape remains platform-uniform.

### How it appears in the architecture

Agents are **producers**: they assign validation outcomes, apply operational corrections, or reconcile proposals into log-level mutations. The platform is the **owner** of FS/FSV lifecycle, routing, aggregation, synchronization, and grouping. Reconciliation decisions (matching a proposal to create, update, or attach) are computed before mutation but do not themselves write aggregate state — execution happens only through materialization.

---

## Materialization Boundaries

### Problem it solves

Multiple entry points — validation runs, human overrides, operational corrections — must produce identical downstream state for identical inputs. Scattered write paths inevitably diverge in routing rules, aggregate math, and sync behavior.

### Tradeoff

All paths converge on a single per-log materialization entry point. Agents cannot short-circuit or duplicate aggregate construction. The cost is that every mutation, regardless of origin, pays the full materialization cost — including routing evaluation even when the result is a noop.

### How it appears in the architecture

The materialization boundary separates agent domain from platform domain. Above the boundary: log-level outcomes and optional operational field writes. Below the boundary: validation persistence, route selection, FS/FSV mutation, conditional canonical recomputation, and FS-to-FSV synchronization. Aggregate state is not authoritative until materialization commits. Human overrides and agent-produced outcomes follow the same path; the platform does not branch on actor identity.

---

## Deterministic Recomputation

### Problem it solves

Aggregate fields — loss totals, detection and fault dates, actionable classification — must reflect the current log set. Incremental patching (add this log's loss, subtract that contribution) assumes orderly, single-threaded history. Concurrent mutations, out-of-order corrections, and revised fields that were already aggregated compound errors and produce order-dependent results.

### Tradeoff

Canonical recomputation reads the full sibling log set and derives aggregates from scratch every time. This is more work per correction than a delta but is safe to retry, safe after noop routing, and safe during revert. Transition-driven route bodies still use incremental helpers for topology changes (join, create, convert) where the event boundary is clear; general-purpose aggregate repair uses full enumeration.

Grouped parent lifecycles are excluded from canonical recomputation — their aggregates follow grouping rules, not sibling-log reduction.

### How it appears in the architecture

When validation status is unchanged but operational fields change, routing returns noop and skips aggregate work. Canonical recomputation closes the gap inside the same transaction. Given the same sibling log set, the derived FS and synchronized FSV are always identical. Empty sibling sets trigger lifecycle cleanup (orphaned FS/FSV removal), which is expected behavior rather than an integrity failure.

---

## Agent Parity

### Problem it solves

Validation and investigation address different questions — credibility versus operational correctness — but both mutate fault logs and require consistent platform state. If equivalent log fields produce different FS/FSV depending on which agent wrote them, downstream consumers cannot trust aggregates.

### Tradeoff

Parity requires the platform to ignore agent origin during materialization. Agents may differ in scope, batching, and persistence models, but the handoff contract is uniform: well-formed log state plus a validation outcome enters materialization; routing and aggregation rules are identical. The validation agent does not gain privileged routing; investigation does not bypass materialization.

### How it appears in the architecture

A log validated to **valid** with given operational fields and a log corrected to the same fields and status through investigation must produce the same FS/FSV after materialization. The validation agent's boundary stops at outcome assignment; investigation's boundary stops at log mutation and handoff. Neither chooses routes, performs aggregation, or runs grouping.

---

## Transactional Consistency

### Problem it solves

A single lifecycle event can touch a fault log, one or two FS/FSV pairs, and membership references. Partial completion — log updated but aggregates stale, FS updated but FSV out of sync — yields state that cannot be explained from log data.

### Tradeoff

The primary transaction bundles log validation persistence, route-specific FS/FSV mutations, conditional canonical recomputation, and FS-to-FSV synchronization. Work that cannot be included without unacceptable cost — cross-lifecycle grouping, external metadata lookups, multi-parent writes — is deferred post-commit. The system chooses **strong per-lifecycle consistency at commit** over **immediate cross-lifecycle consistency**.

Pre-transaction reads fix the route before any write, trading optimistic re-routing for predictable transaction bodies. Concurrent writers on the same equipment identity are serialized rather than resolved through mid-transaction route changes.

### How it appears in the architecture

At commit: log validation fields are durable, membership references are correct, FS aggregates reflect route-body or canonical computation, and paired FS/FSV agree on shared operational fields. All primary-transaction writes succeed or none do. What commit does not guarantee — grouping links, parent aggregates, external audit records — has separate, weaker semantics.

---

## Eventual Consistency

### Problem it solves

Grouping clusters related faults across lifecycles, depends on plant configuration and equipment counts from external stores, and may touch parent and child records unrelated to the triggering log. Including this work in the primary transaction would extend lock duration, couple lifecycle correctness to external availability, and risk rolling back valid per-lifecycle mutations because an unrelated grouping step failed.

### Tradeoff

Grouping runs after commit with explicit eventual-consistency semantics. Per-lifecycle state is correct at commit; group membership may lag briefly. Failures are logged, not compensated — no rollback of the committed mutation. Subsequent materialization on any affected log retries grouping.

Consumers — dashboards, alerting, reporting — must tolerate a window where individual lifecycles are accurate but parent-child links are stale.

### How it appears in the architecture

Post-commit grouping reconciles the old and target validated summaries affected by a materialization event. It evaluates FSV-level eligibility (validation status, actionable, resolution, feedback, linked-to references) against plant thresholds. Certain fault types are excluded entirely. Grouping context is built once per run to avoid redundant external access. Invalid memberships — for example, children linked to unvalidated parents — are detached during reconciliation.

---

## Idempotency

### Problem it solves

Agents retry, validation re-runs on settled logs, and post-commit grouping may be invoked multiple times. Without idempotent primitives, retries amplify errors — double-counting loss, duplicating topology changes, or diverging aggregates from repeated partial application.

### Tradeoff

Idempotency is designed into specific layers, not assumed everywhere. Noop routing makes validation re-runs cheap but insufficient for operational corrections under unchanged status. Canonical recomputation is fully idempotent; transition-route incremental helpers are not general-purpose retry primitives. Safe retry means retrying the full primary transaction or re-invoking post-commit grouping — not replaying individual steps.

### How it appears in the architecture

Re-applying the same validation outcome when status is already settled routes to noop and skips aggregate work. Canonical recomputation from an unchanged sibling set produces unchanged aggregates. Grouping reconciliation on stable FSV state converges to the correct cluster. Operational corrections that change log fields without changing validation status must explicitly trigger recomputation within the transaction — blind materialization retry on noop is insufficient.

---

## Auditability

### Problem it solves

Automated and human actors mutate production fault state. Without traceability, operators cannot determine what changed, who initiated it, whether a correction can be reversed, or whether a failure left partial side effects.

### Tradeoff

Auditability is layered. The platform guarantees durable log and aggregate state at commit with clear attribution on validation fields. Investigator mutations add snapshot-based revert capability — restoring prior log state and recomputing aggregates rather than replaying historical topology. Post-commit failures (grouping, audit record writes) are visible in logs but do not undo committed mutations. The system favors **recoverable, attributable state** over **all-or-nothing audit atomicity**.

Validation runs in the current model produce ephemeral progress signals rather than persisted run records; audit trail for validation lives on the fault log itself.

### How it appears in the architecture

Validation outcomes carry validated-by attribution, timestamps, and optional comments. Human overrides are distinguished from agent attribution. Investigator actions record pre-mutation snapshots and post-mutation outcomes, enabling revert by restoring snapshot state and recomputing — not by reconstructing historical FS/FSV graph shapes. Post-commit failures are logged explicitly; grouping and audit gaps are repaired by subsequent runs rather than compensating transactions.

---

## Explicit Ownership Boundaries

### Problem it solves

Ambiguous ownership leads to duplicated logic, conflicting writes, and untestable coupling. When two components both "own" aggregation or routing, neither can be changed safely and parity breaks silently.

### Tradeoff

Ownership is partitioned strictly, even where fields appear on multiple layers. Agents own log-level facts; the platform owns aggregate lifecycle. Within aggregates, FS owns operational lifecycle fields; FSV owns validation and grouping fields. Actionable classification originates at the log level but is treated as FSV-owned in aggregate behavior. The split adds coordination cost at the materialization boundary but eliminates write conflicts.

### How it appears in the architecture

| Domain | Owner |
|--------|-------|
| Fault log operational fields | Investigation / operational correction path |
| Fault log validation fields | Validation agent, human operators |
| FS lifecycle (dates, losses, hierarchy, resolution) | Shared lifecycle platform |
| FSV validation and grouping state | Shared lifecycle platform |
| Routing and topology | Shared lifecycle platform |
| Grouped reconciliation | Shared lifecycle platform (post-commit) |
| Proposal-to-action matching | Investigation (pre-mutation, no aggregate writes) |

Agents never independently create, route, aggregate, synchronize, or group. The platform never performs agent analysis or proposal generation. The materialization call is the contract surface between the two domains.

---

## Related documents

- [`docs/internal/lifecycle-platform.md`](internal/lifecycle-platform.md) — shared lifecycle platform model
- [`docs/transaction-model.md`](transaction-model.md) — transaction phases and consistency guarantees
- [`docs/internal/system-brain.md`](internal/system-brain.md) — canonical architecture reference
