# Transaction Flow

Phases of a lifecycle mutation and the consistency guarantees at each boundary.

```mermaid
flowchart TD
    PRE["Pre-transaction<br/><i>reads & route determination</i>"]

    PRE --> TXN["Primary transaction"]

    subgraph TXN["Primary transaction"]
        direction TB
        LOG["Persist validation fields"]
        BODY["Route body or noop"]
        RECOMP["Canonical recompute<br/><i>conditional</i>"]
        SYNC["FS ↔ FSV sync"]

        LOG --> BODY
        BODY --> RECOMP
        RECOMP --> SYNC
    end

    TXN --> COMMIT["Commit"]
    COMMIT --> POST["Post-commit grouping"]

    style PRE fill:#f5f5f5
    style COMMIT fill:#e8f5e9
    style POST fill:#fff3e0
```

## Phase guarantees

| Phase | Consistency |
|-------|-------------|
| **Pre-transaction** | Read-only preparation; route fixed before any write |
| **Primary transaction** | Log, FS, and FSV mutate atomically |
| **Commit** | Per-lifecycle state is durable and internally consistent |
| **Post-commit grouping** | Eventual; failures do not roll back committed mutation |

## Related documents

- [`docs/transaction-model.md`](../docs/transaction-model.md)
- [`docs/design-principles.md`](../docs/design-principles.md)
