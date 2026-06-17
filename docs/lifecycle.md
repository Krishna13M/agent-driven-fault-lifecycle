# Fault Lifecycle and State Transitions

Describes the lifecycle states of a fault detection from initial creation through resolution, and the routing logic that governs FS/FSV transitions.

→ Companion to the [README](../README.md) and [`architecture.md`](architecture.md).

---

## Fault Log States

A FaultLog may exist at any of the following stages:

| Stage | Description |
|-------|-------------|
| **Pending** | Created but not yet validated |
| **Agent-validated** | Outcome assigned by automated analysis |
| **Human-overridden** | Validation metadata set by a human operator |
| **Investigator-revised** | Operational fields mutated by the Investigator Agent |
| **Protected** | Explicitly flagged as immutable to automated correction |

Human validation metadata does not confer immutability. A log may be validated, then later revised by the Investigator Agent, without triggering re-validation.

---

## Materialization Routing

The shared lifecycle platform selects a route for each materialization call based on the validation status transition and existing lifecycle topology:

| Route | Condition |
|-------|-----------|
| **Create** | No existing FS lifecycle for this equipment identity; validation outcome is valid |
| **Join** | An existing FS lifecycle is open; the new log attaches to it |
| **Convert** | A pending FS becomes validated after outcome assignment |
| **Noop** | Validation status is unchanged; aggregate re-aggregation is skipped |

When noop is returned but operational fields have changed, canonical recomputation must be explicitly invoked within the same transaction.

---

## FS/FSV Lifecycle States

| State | Description |
|-------|-------------|
| **Active** | Fault is open and unresolved |
| **Resolved** | Underlying condition cleared; lifecycle closed |
| **Grouped** | FSV is a child of a parent group FSV |
| **Parent** | FSV is a parent group aggregating sibling FSVs |

Grouping operates at the FSV layer only. FS records reflect operational aggregates; FSV records carry grouping topology and validation-aware state.

---

## Empty Lifecycle Cleanup

When the last sibling log is removed from a lifecycle, FS and FSV records are deleted. This is expected behavior, not an integrity failure. Recomputation primitives support this cleanup path. Disabling it creates orphan aggregates that no log set can resolve.

---

## Identity Resolution

Lifecycle routing identity is resolved deterministically from the most specific non-empty field in the equipment hierarchy. When no equipment field is set, site-level identity is used. This fixed precedence ensures that concurrent materialization on the same equipment identity is always serializable.
