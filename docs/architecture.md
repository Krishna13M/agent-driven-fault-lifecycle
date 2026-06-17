# Architecture Reference

Detailed component architecture, consistency guarantees, and transaction model for the fault lifecycle platform.

→ This document is a deep-dive companion to the [README](../README.md).

---

## Shared Lifecycle Platform

Two agents — Validation and Investigator — must produce consistent downstream state. Without a shared platform, each would reimplement routing, aggregation, synchronization, and grouping independently, with inevitable semantic drift.

The shared lifecycle platform provides:

| Capability | Description |
|------------|-------------|
| **Materialization** | Per-log entry point: persist validation fields, route FS/FSV mutations, synchronize aggregates |
| **Routing** | Transition-driven path selection: create, join, convert, or noop based on validation status and existing topology |
| **Canonical recomputation** | Re-derive FS aggregates from the full sibling log set, independent of routing |
| **Grouped reconciliation** | Post-commit clustering and parent group maintenance |
| **Transactional execution** | Atomic mutation + materialization within a single database transaction |
| **Identity resolution** | Deterministic equipment hierarchy traversal for lifecycle matching |
| **Actionable classification** | Persisted as part of materialization, not delegated to agents |

The platform does not distinguish agent origin during materialization. Parity between agents is a design requirement, not an implementation coincidence.

---

## Ownership Boundaries

| Owner | Domain |
|-------|--------|
| Fault logs — operational fields | Investigator Agent |
| Fault logs — validation fields | Validation Agent and human operators |
| FS / FSV lifecycle | Shared lifecycle platform |
| Grouping | Shared lifecycle platform (post-commit) |
| Reconciliation decisions | Investigator Agent (pure, pre-mutation) |
| Run orchestration | Per-agent worker layer |

---

## Transaction Model

All lifecycle mutations follow a phased execution model:

```
┌─────────────────────────────────────────────────────┐
│  PRIMARY TRANSACTION                                │
│    mutation → materialize → recompute? → sync       │
└────────────────────────┬────────────────────────────┘
                         │ commit
                         ▼
┌─────────────────────────────────────────────────────┐
│  POST-COMMIT                                        │
│    grouped reconciliation → audit → events          │
└─────────────────────────────────────────────────────┘
```

At commit: fault log state is durable; FS and FSV are consistent with each other for the affected lifecycle. Rollback before commit reverses all mutation and materialization work. Post-commit failures are logged and visible in audit records — never silently swallowed.

Post-commit grouped reconciliation performs multiple independent writes and depends on external metadata. Keeping it outside the primary transaction avoids long-held locks and partial rollback complexity. Consumers must tolerate brief grouping lag.

---

## Consistency Guarantees

| Guarantee | Scope |
|-----------|-------|
| **Atomic** | Fault log + FS + FSV within the primary transaction |
| **Read-your-writes** | Materialization sees its own in-transaction log mutations |
| **FS/FSV sync at commit** | Paired records reflect each other after every commit |
| **Eventual (grouping)** | Parent-child group links may lag if post-commit grouping fails |
| **Serialized per identity** | Concurrent materialization on the same equipment identity is serialized |

The system does not guarantee immediate grouping consistency or cross-lifecycle atomicity. It guarantees that committed log and aggregate state are always internally consistent.

---

## Architectural Principles (Extended)

### Idempotency

Canonical recomputation from unchanged logs produces unchanged aggregates. Re-running materialization with the same outcome on an already-settled lifecycle noops. Investigator revert restores prior log state and recomputes — it does not replay historical topology.

Idempotency is a property of the recomputation primitive and routing semantics, not an accident of caching.

### Deterministic Processing

- Reconciliation decisions are pure functions of proposal, candidates, and context
- Identity resolution follows fixed hierarchy precedence
- Aggregate recomputation reads the full sibling set and applies fixed reduction rules
- Grouping thresholds are computed from asset configuration and resolved equipment counts

Non-determinism is confined to agent analysis layers (LLM output, telemetry interpretation). All downstream materialization is deterministic given those outputs.

### Empty Lifecycle Cleanup

When the last sibling log is removed from a lifecycle, FS and FSV deletion is expected behavior, not an integrity failure. Recomputation primitives must support this cleanup path; disabling it for convenience creates orphan aggregates.

### External Packages Stop at Proposals

Both the validator and investigator packages are external analysis producers. The application layer owns orchestration, reconciliation, lifecycle transactions, and ecosystem integration. Keeping package boundaries at proposal generation prevents ecosystem coupling from leaking into analysis code.
