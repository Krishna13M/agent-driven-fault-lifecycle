# Validation Agent

## Purpose

The Validation Agent is an embeddable library that reviews automated fault detections for solar plants. It takes fault logs produced by upstream detection systems, gathers the telemetry and metadata needed to judge each alert, invokes structured reasoning (an LLM reasoning layer or deterministic rule engines), and returns a validation verdict with supporting rationale.

The agent is deliberately not a standalone service. It is a pipeline that host applications—dashboard APIs, batch workers, or CLI tools—invoke when they want expert-style review of open faults. Its contract is: **given fault logs, produce validation decisions and debug artifacts; leave authoritative state management to the caller.**

```
Fault Logs
     │
     ▼
Validation Agent
     │
     ├── Case Construction
     ├── Evidence Assembly
     ├── Materialization
     └── Decision Generation
     │
     ▼
Validation Outcome
     │
     ▼
Host Platform
(Reconciliation & State Ownership)
```

## Problem It Solves

Solar monitoring platforms generate large volumes of hierarchical faults—plant shutdowns, grid curtailment, inverter events, string disconnections—across plants operating in different timezones. Each alert may be genuine, a symptom of a parent condition, or a false positive driven by missing data, low irradiance, or communication gaps.

Manual review does not scale. Fully automated acceptance is unsafe without evidence. The Validation Agent sits in the middle: it operationalizes domain expertise (encoded in a knowledge base and fault-specific plugins) so that every fault can be reviewed against telemetry, plots, and historical context before a platform commits to a final disposition.

The core engineering problem is not "call an LLM on a fault." It is **orchestrating heterogeneous evidence acquisition, dependency-aware ordering across a fault graph, bounded parallelism over shared infrastructure, and reproducible decision artifacts**—while keeping the boundary clear between *generating* a decision and *owning* the platform's aggregate fault state.

## Core Responsibilities

The Validation Agent owns the full path from fault intake to validation conclusion:

- **Discovery and filtering** — Load fault logs for plants and date ranges from the platform's operational store, optionally excluding auto-validated faults or restricting to named fault types. Accept caller-supplied fault logs when the host application has already filtered or enriched them.

- **Case construction** — Transform each fault log into a structured `Case`: hierarchy placement, input identifiers, occurrence windows, a declarative data plan (`data_needed`), and versioning metadata. Fault-specific plugins encode what evidence is required for each fault type.

- **Context and evidence assembly** — Execute the data plan: fetch timeseries and metadata from platform APIs, resolve comparison inputs, generate plots, assemble historical fault context, and build agent-ready prompts. Evidence gathering and prompt construction live here rather than in a separate analysis stage.

- **Asset materialization** — Upload images, CSV attachments, and case payloads to object storage so the reasoning layer consumes stable references instead of in-memory blobs.

- **Validation execution** — Route to deterministic rule engines where outcomes are fully derivable (e.g., communication issues, missing data), apply orchestrator-level prechecks (e.g., parent-shutdown invalidation for disconnected strings), or invoke the LLM reasoning layer with strict response validation.

- **Artifact persistence** — Write complete case bundles to cloud storage (case JSON, context, analysis metadata, raw fault log, errors) for replay, audit, and debugging.

- **Dependency-aware scheduling** — Classify faults into hierarchy stages, attach same-day dependency edges (plant before grid before lineage children; chronological ordering for selected fault types), and run ready work in parallel within configured limits.

- **Operational protection** — Per-service throttles, bounded retries, and structured logging with run-scoped context (run ID, plant, day, case ID, fault ID).

- **Return enriched results** — Emit JSON-serializable fault-log dicts augmented with `validation_status`, `validated_by`, and nested `agent_results` for the host to persist or reconcile.

## Validation Workflow

At a high level, every validation run follows the same lifecycle:

```
Host invokes runner
    → Initialize config, clients, throttles
    → Resolve plant timezones; group plants for day-window queries
    → For each calendar day / plant batch:
        → Fetch fault logs (read-only)
        → Build scheduler nodes with dependency edges
        → Process ready batches in parallel
            → Per fault:
                1. Case build (plugin)
                2. Context build (plugin + data fetch + plots)
                3. Materialize assets to cloud storage
                4. Deterministic precheck or reasoning-layer invocation
                5. Merge conclusion; persist case bundle
                6. Return enriched fault-log dict
    → Yield or collect batch results to caller
```

**Entry points** expose this workflow at different granularities:

| Surface | Role |
|--------|------|
| `run_validation_in_batches` | Async generator; caller handles each scheduler batch as it completes |
| `run_validation` | Full run with optional per-batch callback; returns flattened list |
| `run_validation_sync` | Synchronous wrapper for CLI and blocking hosts |
| `fault-validator` CLI | Operational entry for plant/date-range runs |
| Prefetched `FaultLog` list | Skips discovery; host controls intake |

The orchestrator wires data clients, policy resolution, artifact storage, and the reasoning client. The scheduler ensures that when multiple faults coexist on the same plant and day, higher-level conditions are validated before dependent child faults—so downstream conclusions can account for upstream context without blocking unrelated work when parents are absent.

## Decision Model

Every processed fault produces a **decision** and a **delivery shape**:

### AgentConclusion

An `AgentConclusion` captures the verdict:

| `validated_status` | Meaning |
|-------------------|---------|
| `valid` | Fault confirmed against evidence |
| `invalid` | Fault rejected or explained by another cause |
| `inconclusive` | Insufficient evidence, build failure, or ambiguous signals |

Supporting fields include confidence score, engineering rationale, cited data points, visual observations, and optional extended metadata (detection method, secondary candidates, recommended action).

### Case lifecycle

Cases transition through a minimal internal state machine:

- **`to be reviewed`** — Case and context built; awaiting or undergoing validation.
- **`reviewed`** — A conclusion is attached (from the reasoning layer, deterministic engine, precheck, or fail-closed build error).

Build failures do not abort the pipeline silently. They produce a reviewed case with `inconclusive` status so the run remains auditable. Reasoning-layer transport or parse failures leave the case without a conclusion; the returned fault log carries `validation_status = null` while error artifacts in the case bundle record the failure.

### Caller-facing envelope

The host receives the original fault-log document enriched with:

- `validation_status` — Top-level disposition (or `null` on reasoning failure)
- `validated_by` — `"validation_agent"` when a conclusion exists
- `agent_results` — Confidence, rationale payload, KB and validator versions

The library does **not** automatically write these fields back to the platform's authoritative store. A separate helper exists for hosts that choose a legacy attach API write path.

## Architectural Boundaries

The system is organized around five concerns that are easy to conflate but serve different roles:

### Validation

**Scope:** Judging whether a fault log is correct.

Validation encompasses case construction (what to look at), context assembly (what was observed), and conclusion generation (the verdict). It ends at the `AgentConclusion` and the enriched return dict. Validation does not decide how the platform's dashboards, workflows, or operational indexes reflect that verdict.

### Materialization

**Scope:** Turning in-memory evidence into durable, referenceable artifacts before reasoning.

Plots, CSV extracts, and serialized case JSON are uploaded to object storage; stable URLs and refs are injected into prompts. Materialization is a **transport boundary**: it keeps reasoning requests bounded, releases large byte buffers before long-running calls, and creates an immutable snapshot of what the validator saw. It is not validation logic and not aggregate state management.

### Aggregation (evidence)

**Scope:** Combining raw telemetry into comparable signals within a single validation run.

Aggregation appears in fault-specific builders: time-bucketed telemetry queries, grouping series by input ID, orientation-group comparisons for grid curtailment, sibling grouping for ground-fault plots. This is **analytic aggregation for one case**, not platform-wide rollups of validation status across plants or time.

### Grouping (orchestration)

**Scope:** Ordering and batching work for correctness and throughput.

The scheduler groups faults by execution stage (plant → grid → inverter → MPPT → combiner → string), attaches dependency edges within a day, and groups chronological faults (e.g., disconnected strings) so older events on the same input lineage complete before newer ones. Plants are grouped by timezone for accurate day-window queries. Grouping governs **when** conclusions are produced relative to sibling faults; it does not merge conclusions into a single aggregate record.

### Reconciliation

**Scope:** Aligning validation decisions with the platform's authoritative fault-log state.

Reconciliation is **explicitly outside** the Validation Agent. The host application (or a downstream service) maps returned dicts to its persistence model, handles idempotency of reruns, resolves conflicts between multiple case runs for the same fault, and updates operational views. Optional platform API adapters are convenience hooks, not part of the core pipeline.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Host application                             │
│  intake → invoke validator → reconcile → persist fault state    │
└────────────────────────────┬────────────────────────────────────┘
                             │ FaultLog[] in, enriched dict[] out
┌────────────────────────────▼────────────────────────────────────┐
│                   Validation Agent (library)                     │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ Grouping │→ │ Case/Context │→ │Materialization│→│ Decision │ │
│  │(scheduler)│  │  (plugins)   │  │ (obj. store) │  │(rules /  │ │
│  └──────────┘  └──────────────┘  └──────────────┘  │  LLM)    │ │
│                                                     └────┬─────┘ │
│  ┌──────────────────────────────────────────────────────▼─────┐ │
│  │ Case bundles in object storage (owned artifacts)           │ │
│  └────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────┘
                             │ read-only
┌────────────────────────────▼────────────────────────────────────┐
│ Platform data plane (fault logs, telemetry APIs, knowledge base)  │
└─────────────────────────────────────────────────────────────────┘
```

## State Ownership Model

Clear ownership prevents accidental coupling and makes reruns safe.

| State | Owner | Validator's relationship |
|-------|-------|---------------------------|
| Fault log documents (detection metadata, occurrences, hierarchy IDs) | Upstream platform | **Read-only** consumer |
| Auto-validated exclusion index | Platform | **Read-only** join |
| Per-plant validation policies (e.g., inverter shutdown irradiance thresholds) | Platform config | **Read-only**; merged into immutable policy objects at run start |
| Plant topology, timeseries, metadata | Platform data services | **Read-only** via data clients |
| Case identity (`case_id`) and case JSON | Validation Agent | **Creates** a new case per run |
| Debug bundles (context, analysis, errors, assets) | Validation Agent | **Writes** to object storage |
| Validation verdict on fault log (`validation_status`, `agent_results`) | Host application | **Produces** in memory; **does not persist** |
| Operational case index (pending / reviewed / failed) | Planned case registry | **Not implemented**; object storage is the current index of record |
| Knowledge base (fault criteria taxonomy) | Packaged with validator | **Read-only** at runtime |

Each validation run mints a fresh `case_id`. Multiple runs against the same fault are expected during reruns or prompt experiments. The validator does not enforce "latest wins" on the platform side—that is reconciliation policy for the host.

## Interaction With Aggregate State

"Aggregate state" in the platform sense—what operators see as the current disposition of faults across plants—is **not produced inside the Validation Agent**. The agent produces **atomic validation outcomes** per fault log per run.

How aggregate state emerges in a complete system:

1. **Detection** writes fault logs (open / unvalidated).
2. **Validation Agent** returns enriched dicts with `validation_status` and evidence payloads.
3. **Host reconciliation** writes conclusions to the authoritative store (platform APIs, legacy attach endpoints, or custom persistence).
4. **Downstream aggregation** (dashboards, reporting, SLA metrics) rolls up reconciled fault-log fields—the validator does not participate in this rollup.

Within a single run, the scheduler creates a *logical* ordering graph so that when plant shutdown and string faults coexist, plant-level validation completes first. That ordering informs parallel execution and can inject context (e.g., parent-shutdown prechecks) but does not mutate a shared in-run aggregate. Each fault still yields its own conclusion and case bundle.

Batch callbacks (`on_batch_complete`, `run_validation_in_batches`) exist so hosts can incrementally reconcile partial results—updating aggregate views as batches finish rather than waiting for an entire multi-day run.

## Relationship To Materialization

Materialization is the handoff between **evidence production** and **decision consumption**.

Context builders may generate plots and tabular attachments in memory. Before the reasoning layer runs, the orchestrator uploads these to object storage, replaces raw bytes with URLs in the prompt payload, and stores a case-input snapshot. After the run, the full case bundle—including final case JSON with conclusion, context, analysis metadata, and optional error records—is written under a hierarchical key keyed by plant, date, fault name, fault log ID, and case ID.

This separation yields three benefits:

- **Reasoning requests stay within size and latency limits** while still referencing rich evidence.
- **Runs are reproducible**—reviewers can inspect exactly what was sent and returned without re-fetching live telemetry.
- **Memory pressure is bounded** under parallel fault processing by releasing materialized blobs before long-running calls.

Materialization failures are retried with the same throttle policy as other storage operations; validation may still complete with partial artifact persistence logged.

## Relationship To Reconciliation

Reconciliation is the host's job of making validation outcomes **durable and consistent** with the rest of the platform.

The Validation Agent intentionally stops at returning enriched dicts because:

- **Different hosts have different commit semantics**—optimistic updates, workflow gates, human review queues, or event sourcing.
- **Reruns must not implicitly overwrite production state**—a failed or experimental run should not mutate the fault-log store just because a debug bundle was persisted.
- **Idempotency keys and conflict resolution belong at the persistence layer**—e.g., whether a second `invalid` overrides a prior `valid`, or whether inconclusive triggers re-queue.

Embedding applications wire their own reconciliation: map `validation_status` to domain enums, attach `case_id` for traceability, emit domain events, or stage results for human approval.

A planned operational case registry would provide an index—status, artifact pointers, run metadata—for "what needs review" queries. That registry is specified but not shipped; today, object-storage paths and structured logs serve debugging and audit, while aggregate fault state remains entirely on the host side.

## Design Principles

1. **Library, not service** — No HTTP server in-repo; configuration and credentials load in the host process. The agent integrates via Python APIs or CLI subprocess.

2. **Plugin extensibility** — New fault types add a case builder and context builder registered by name. Defaults handle unknown faults conservatively.

3. **Fail closed, stay auditable** — Build errors become inconclusive reviewed cases. Reasoning failures persist error artifacts and return partial envelopes rather than crashing the batch.

4. **Async-first with isolated blocking I/O** — Network calls are async; synchronous datastore reads run in thread pools so the event loop stays responsive.

5. **Bounded parallelism with backpressure** — Scheduler maximizes ready work; per-service throttles cap datastore, telemetry API, storage, and reasoning-layer concurrency independently.

6. **Dependency-aware, not dependency-blocking** — Child faults wait for parents only when those parents are present in the same batch. Absent parents do not stall unrelated work.

7. **Determinism where possible** — Rule engines and prechecks handle faults with closed-world evidence; the LLM layer handles interpretive cases with schema-validated outputs.

8. **Separation of decision and commit** — Conclusions are values returned upward; persistence of platform state is always an explicit downstream act.

9. **Evidence traceability** — Every conclusion links to materialized artifacts and version stamps (`kb_version`, `validator_version`).

10. **Spec-driven evolution** — Internal design specs describe target architecture (e.g., case analyzer stage, case registry); implementation may consolidate stages when simplicity wins.

## What Makes This Different

Many validation systems stop at producing a verdict.

This architecture treats validation as a producer of decisions rather than the owner of operational state.

The Validation Agent generates conclusions and evidence artifacts, while lifecycle state, reconciliation, aggregate ownership, and downstream operational views remain responsibilities of the broader platform.

This separation allows validation logic to evolve independently from persistence, reporting, and workflow concerns.

## Key Architectural Decisions

### Fold analysis into context building

An originally planned `CaseAnalyzer` stage was not implemented as a separate module. Telemetry fetch, signal computation, best-input selection, plot generation, and prompt assembly live in fault-specific context builders (with shared helpers). This reduces pipeline hops and keeps evidence and prompts co-located, at the cost of larger plugins.

### Return dicts instead of writing back to the fault-log store

The pipeline reads fault logs but never writes validation fields back. Hosts control transaction boundaries, retries, and authorization for state mutation. Object storage holds validator-owned artifacts; the platform store remains the source of truth for fault disposition only after reconciliation.

### Object storage as the case store of record (for now)

Full case bundles land in a hierarchical object-store layout. An operational case registry is designed for query workloads but deferred. Storage-first artifact persistence keeps the validator stateless with respect to operational indexes.

### Hierarchy-staged scheduler with opportunistic parallelism

Static stage barriers (all plant, then all grid) would under-utilize capacity. The engine builds each batch from the current *ready set*—nodes whose dependencies are satisfied—up to a parallelism cap, spreading work across parent inputs when possible. Chronological faults add per-lineage ordering for faults like disconnected strings.

### Dual validation paths: deterministic engines and LLM reasoning

Communication issues and missing data route to rule engines when attachments suffice. Disconnected strings can short-circuit on parent-shutdown prechecks. Remaining faults use the LLM reasoning layer with centralized response validation, retries on transport and schema errors, and refusal detection. This reduces cost and variance where the decision space is closed.

### Policy materialization at run boundary

Inverter-shutdown and similar behaviors depend on hierarchical config (defaults → company → plant) merged once per run into immutable policy objects. Prompts consume typed policies rather than raw config documents, isolating runtime behavior from storage shape.

### Batch-oriented public API

`run_validation_in_batches` acknowledges that validation runs are long-lived. Hosts reconcile or notify per batch instead of holding all results in memory or blocking user flows until multi-plant date ranges complete.

## Lessons Learned

**Validation is a pure function.** Coupling reasoning output directly to the fault-log store complicated reruns and failure recovery. Returning enriched dicts and persisting debug bundles separately made the pipeline idempotent at the library layer and pushed commit semantics to the host.

**Ordering is correctness.** Plant and grid faults can invalidate child alerts; validating strings before confirming plant shutdown wastes reasoning calls and produces misleading conclusions. Dependency-aware scheduling aligns verdicts with physical causality.

**Materialize before reasoning.** Inline binary payloads do not scale under parallel load. Upload-then-reference patterns stabilized latency and created an audit trail reviewers actually use.

**Not every fault deserves an LLM.** Closed-form rules for communication and missing-data patterns improved consistency and cost. Both deterministic engines and the LLM layer share the same conclusion contract so downstream reconciliation stays uniform.

---

*This document describes the architecture inferred from the Validation Agent codebase. It is intended as portfolio-grade systems documentation—focused on responsibilities, boundaries, and lifecycle—not as API or module reference material.*
