# Architecture Overview

A map of how the documentation fits together — what each layer owns, how data flows between them, and where to read next.

This repository documents an **agent-driven fault lifecycle** for renewable-energy plants: automated agents and human operators produce outcomes at the fault-log level; a shared platform materializes those outcomes into operational and validated aggregate state.

The documentation is organized in four layers. Read them in order.

```text
Validation Agent
        ↓
Lifecycle Platform
        ↓
Transaction Model
        ↓
Design Principles
```

---

## The problem in one paragraph

Solar plants generate continuous fault detections across hierarchical equipment. Those detections are noisy and numerous. Operators need to know which events are credible, how they roll up into plant-level state, and how related faults cluster into actionable groups. The system separates **decision-making** (what an agent or operator concludes about a detection) from **state ownership** (how lifecycle records are created, aggregated, and kept consistent). That separation is the spine of the architecture.

---

## Layer 1 — Validation Agent

**Document:** [validation-agent.md](validation-agent.md)

The validation agent answers one question: *Is this fault detection credible?* It classifies logs as valid, invalid, or inconclusive and persists validation metadata — status, attribution, timestamps, comments.

It is a **producer**, not an owner of lifecycle topology. It does not create FS or FSV records, choose routes, aggregate loss, synchronize views, or run grouping. Its sole handoff to the platform is a fault log plus a standardized validation outcome.

Human overrides follow the same path. The platform does not branch on who produced the outcome.

**Read this doc to understand:** what the validation agent owns, what it deliberately does not own, and the materialization handoff boundary.

---

## Layer 2 — Lifecycle Platform

**Document:** [lifecycle-platform.md](lifecycle-platform.md)

The shared lifecycle platform turns log-level outcomes into durable aggregate state: FaultStatus (FS) for operational summaries and FaultStatusValidated (FSV) for validated, grouping-aware summaries.

**Materialization** is the single per-log entry point. Given a log and an outcome, the platform persists validation fields, selects a route (create, join, convert in place, or noop), mutates FS/FSV, and synchronizes the pair. Routing is driven by validation-state transitions, not by which agent invoked materialization.

Three entities define the model:

- **FaultLog** — source of truth
- **FS** — derived operational aggregate
- **FSV** — derived validated aggregate, synchronized from FS

When validation status is unchanged but operational fields change, routing may noop and skip aggregate work. The platform closes that gap with **canonical recomputation** — re-deriving FS from the full sibling log set.

**Grouping** runs after commit, outside the primary transaction. Per-lifecycle state is correct at commit; parent-child cluster links may lag briefly.

**Read this doc to understand:** ownership between log and aggregates, routing behavior, lifecycle scenarios, and grouping semantics.

---

## Layer 3 — Transaction Model

**Document:** [transaction-model.md](transaction-model.md)

The transaction model explains how the platform maintains consistency while mutating lifecycle state.

Every materialization passes through distinct phases:

1. **Pre-transaction** — load log, identity, current and target FS/FSV, date statistics; fix the route before any write
2. **Primary transaction** — persist validation fields, execute route body, run conditional canonical recompute, synchronize FS ↔ FSV
3. **Commit** — log, FS, and FSV are atomically consistent for the affected lifecycle
4. **Post-commit grouping** — eventual; failures do not roll back the committed mutation

The primary transaction guarantees that partial writes — log updated but aggregates stale, or FS updated but FSV out of sync — do not survive commit. Work that cannot be made atomic without unacceptable cost (cross-lifecycle grouping, external metadata lookups) is deliberately deferred.

**Read this doc to understand:** phase boundaries, commit guarantees, failure semantics, idempotency, and why grouping sits outside the transaction.

---

## Layer 4 — Design Principles

**Document:** [design-principles.md](design-principles.md)

Design principles explain *why* the architecture is shaped this way — the problems each rule solves, the tradeoffs accepted, and how the rule appears across the layers above.

The principles are not independent ideas. They reinforce each other:

- **Log as source of truth** makes **canonical recomputation** safe and **revert** possible
- **Materialization boundaries** enforce **agent parity** across producers
- **Transactional consistency at commit** pairs with **eventual grouping** as a deliberate split
- **Explicit ownership** prevents agents from duplicating platform logic

**Read this doc to understand:** the reasoning behind decisions that appear in the other three documents.

---

## How the layers connect

| Layer | Question it answers |
|-------|---------------------|
| Validation Agent | Who produces validation outcomes, and where does agent responsibility end? |
| Lifecycle Platform | How do outcomes become FS/FSV, and what happens on each route? |
| Transaction Model | What is atomic at commit, what is eventual, and what happens on failure? |
| Design Principles | Why was the system designed this way? |

```text
Agent outcome (validation or operational)
        │
        ▼
   Materialization  ──primary transaction──▶  FS / FSV consistent at commit
        │
        ▼
   Grouping (post-commit, eventual)
```

Investigation and human override are additional producers that converge on the same materialization path. The validation agent doc describes the first producer in detail; the lifecycle platform doc describes the path all producers share.

---

## Where to start

| If you want to… | Start with |
|-----------------|------------|
| Orient yourself quickly | [README](../README.md) |
| See how the docs connect | This page |
| Understand agent boundaries | [Validation Agent](validation-agent.md) |
| Understand materialization and grouping | [Lifecycle Platform](lifecycle-platform.md) |
| Understand consistency guarantees | [Transaction Model](transaction-model.md) |
| Understand the reasoning | [Design Principles](design-principles.md) |

The README embeds the **Ownership Model** and **Transaction Flow** diagrams. Use those for a visual summary; use the four documents above for depth.
