# fault-lifecycle-platform

An architecture case study of a production fault lifecycle platform that coordinates multiple autonomous agents — a **Validation Agent** and an **Investigator Agent** — through a shared materialization, reconciliation, and lifecycle framework.

This repository documents the architecture and design tradeoffs behind a non-trivial event-driven system built for operational fault management at scale. It is aimed at software engineers, data engineers, backend engineers, and technical reviewers.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Why Validation Alone Is Insufficient](#2-why-validation-alone-is-insufficient)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core Entities](#4-core-entities)
5. [Validation Agent](#5-validation-agent)
6. [Investigator Agent](#6-investigator-agent)
7. [Architectural Principles](#7-architectural-principles)
8. [Repository Structure](#8-repository-structure)
9. [Author's Role](#9-authors-role)

---

## 1. Problem Statement

Systems that continuously monitor complex equipment hierarchies produce a high volume of automated fault detections — potential operational issues such as underperformance, disconnection, or anomalies, each with associated loss estimates, severity levels, and timestamps.

**Raw detections are inherently noisy.** They overlap, recur, and vary in credibility. Downstream consumers — dashboards, reporting pipelines, work-order systems, analytics — need a curated, consistent, and explainable view of what is real, what is in progress, and what requires attention.

A fault detection is not a single static record. Over time, detections are:

- **Validated** — confirmed, rejected, or marked inconclusive by automated or human review
- **Aggregated** — rolled up from individual events into operational summaries
- **Grouped** — clustered with related faults to reduce alert noise
- **Resolved** — closed when the underlying condition clears
- **Revised** — updated when new telemetry changes occurrence data, loss estimates, or date windows

Without lifecycle management, downstream consumers see a stream of disconnected events with no operational meaning.

---

## 2. Why Validation Alone Is Insufficient

Validation answers one question: *Is this detection credible?* It assigns an outcome — valid, invalid, or inconclusive — to individual fault events. That is necessary but not sufficient.

Validation alone does not:

- **Correct operational facts.** Occurrence windows and loss totals may need revision even when validation status is unchanged.
- **Create missing lifecycle records.** A credible detection may require a new fault record where none existed.
- **Attach detections to existing lifecycles.** A new detection may belong to a fault already in progress.
- **Maintain aggregate consistency.** Changing one detection's data must propagate to parent summaries and sibling records.
- **Maintain grouping coherence.** Related faults must be clustered or unclustered as conditions evolve.

These responsibilities belong to a dedicated Investigation capability. Both agents converge on a **shared materialization platform** that owns aggregate state. Neither constructs or manages lifecycle aggregates independently.

---

## 3. High-Level Architecture

```
                      ┌──────────────────────┐
                      │   Raw Fault           │
                      │   Detections          │
                      └──────────┬────────────┘
                                 │
             ┌───────────────────┴───────────────────┐
             │                                       │
             ▼                                       ▼
 ┌───────────────────────┐             ┌───────────────────────┐
 │   Validation Agent    │             │   Investigator Agent   │
 │  - Classify credibility│            │  - Propose corrections │
 │  - Persist outcomes   │             │  - Reconcile to action │
 │  - Handoff to platform│             │  - Transactional apply │
 └───────────┬───────────┘             └───────────┬───────────┘
             │                                     │
             └─────────────────┬───────────────────┘
                               │
                               ▼
               ┌───────────────────────────────┐
               │    Shared Lifecycle Platform   │
               │  - Single materialization path │
               │  - Routing (create/join/noop)  │
               │  - Canonical recomputation     │
               │  - FS ↔ FSV synchronization    │
               │  - Grouped reconciliation      │
               └───────────────┬───────────────┘
                               │
             ┌─────────────────┴─────────────────┐
             │                                   │
             ▼                                   ▼
 ┌───────────────────────┐         ┌──────────────────────────┐
 │   FaultStatus (FS)    │◀─ sync ─▶  FaultStatusValidated    │
 │   Operational summary │         │  (FSV) — Validated view  │
 └───────────────────────┘         └──────────────────────────┘
```

**Agents are thin producers.** They write to fault logs and hand off to the platform. The platform is the single authority for aggregate state.

---

## 4. Core Entities

**FaultLog** is the source of truth for a single fault detection or revision. All aggregate state is derived from the current set of logs linked to a lifecycle. Recomputation and revert operate by restoring log state and re-deriving aggregates from it. Both the Validation Agent (validation fields) and the Investigator Agent (operational fields) write to fault logs — they are orthogonal concerns.

**FaultStatus (FS)** is the materialized operational summary for a lifecycle. It aggregates loss totals, date ranges, equipment identity, resolution state, and severity across all sibling logs in the lifecycle. The shared lifecycle platform exclusively constructs and mutates FS — no agent writes to it directly.

**FaultStatusValidated (FSV)** mirrors FS structure but carries validation-specific and grouping-specific state: validation status, actionable flag, grouping links, and human-validation timestamps. Consumers that need validation-aware state read FSV. Grouping operates at the FSV layer.

```
FaultLog (many) ──membership──▶ FaultStatus (one)
                                      │
                                      ▼ mirror / sync
                               FaultStatusValidated (one)
```

---

## 5. Validation Agent

The Validation Agent classifies fault detections as valid, invalid, or inconclusive using automated analysis. It is a **producer** of validation outcomes — it does not own lifecycle topology or aggregate state.

It persists outcomes to fault log fields, triggers materialization so validation state propagates to FS and FSV, and supports scoped batch runs across configurable dimensions (asset identity, date ranges, fault types, or explicit log subsets). Human overrides are supported without breaking downstream consistency.

Validation runs are ephemeral and request-scoped. The agent explicitly does not own FS/FSV routing, the grouping engine, or operational field mutation.

→ See [`docs/validation-agent.md`](docs/validation-agent.md) for the full input/output contract and run model.

---

## 6. Investigator Agent

The Investigator Agent analyzes existing fault state and telemetry to produce structured correction proposals. It reconciles each proposal to a lifecycle action, applies mutations transactionally, and records a full audit trail with revert capability. It is a **producer** of fault log mutations — it delegates all aggregate materialization to the shared platform.

Before any mutation, pure deterministic reconciliation resolves each proposal to one of four actions:

| Action | Condition |
|--------|-----------|
| `UPDATE_FAULT` | One hierarchy-compatible same-day candidate exists and is not protected |
| `ATTACH_TO_EXISTING` | No same-day candidate; an attachable open lifecycle exists |
| `CREATE_FAULT` | No same-day candidate and no attachable lifecycle |
| `SKIP / REJECT` | Protected log, ambiguous match, hierarchy mismatch, or invalid proposal |

Investigator runs require persisted identity, action records, and revert linkage — a meaningfully different run model from the ephemeral validation agent.

→ See [`docs/investigator-agent.md`](docs/investigator-agent.md) for the full reconciliation decision model and revert design.

---

## 7. Architectural Principles

**One materialization path.** Every agent mutation — validation outcome, human override, investigator create/update/attach, revert — converges on the same materialization entry point. Duplicate implementations are prohibited by design.

**Separation of decision from mutation.** Reconciliation produces decisions; lifecycle transactions perform mutations. Mixing discovery, decision, and write in one step makes testing, replay, and audit impractical.

**Idempotency.** Canonical recomputation from unchanged logs produces unchanged aggregates. Re-running materialization on a settled lifecycle noops. Revert restores prior log state and recomputes — it does not replay historical topology.

**Deterministic processing.** Reconciliation decisions are pure functions of proposal, candidates, and context. Non-determinism is confined to agent analysis layers. All downstream materialization is deterministic given agent outputs.

**Consistency guarantees.** Fault log, FS, and FSV are atomic within the primary transaction. Grouping is eventual — post-commit grouped reconciliation failures do not roll back the committed mutation. Concurrent materialization on the same equipment identity is serialized.

**Validation and investigation are orthogonal.** A log can be validated and later have its occurrences revised without re-validation. Human validation metadata does not block subsequent automated correction.

→ See [`docs/architecture.md`](docs/architecture.md) for the full principles reference, consistency table, and transaction model.

---

## 8. Repository Structure

```
fault-lifecycle-platform/
├── README.md
└── docs/
    ├── architecture.md          # Principles, consistency guarantees, transaction model
    ├── validation-agent.md      # Validation Agent: inputs, outputs, run model, boundaries
    ├── investigator-agent.md    # Investigator Agent: reconciliation, revert, audit
    ├── lifecycle.md             # Fault lifecycle state transitions and routing
    └── reconciliation.md        # Reconciliation model, recomputation, grouping
```

---

## 9. Author's Role

This repository is based on production systems I designed, implemented, and maintained while building fault management infrastructure for large-scale industrial monitoring platforms.

Areas of contribution include:

- Lifecycle orchestration and shared platform design
- Validation workflow architecture and run model
- Investigator reconciliation design (proposal → decision → transactional execution)
- Canonical aggregate recomputation strategy
- Investigator platform integration and revert capability
- Operational automation workflows and telemetry processing pipelines
- Audit trail design with pre/post snapshot storage

This repository intentionally generalizes implementation details while preserving the architectural concepts and engineering tradeoffs from production.
