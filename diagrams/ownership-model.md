# Ownership Model

Who writes what — agents own log-level outcomes; the platform owns aggregate lifecycle state.

```mermaid
flowchart TD
    subgraph agents["Agents & operators"]
        VA["Validation Agent"]
        INV["Investigation"]
        HO["Human Override"]
    end

    FL["FaultLog"]

    subgraph platform["Shared lifecycle platform"]
        FS["FaultStatus (FS)"]
        FSV["FaultStatusValidated (FSV)"]
    end

    VA --> FL
    INV --> FL
    HO --> FL

    FL -.->|"membership reference"| FS
    FS --> FSV

    MAT["Materialization boundary"] -.-> platform
```

## Ownership summary

| Owner | Domain |
|-------|--------|
| **Agents & operators** | FaultLog — validation fields, operational fields, audit metadata |
| **Platform** | FS — operational lifecycle aggregates |
| **Platform** | FSV — validation status, actionable, grouping links, feedback |

Agents do not independently create, route, aggregate, synchronize, or group. The platform does not perform agent analysis or proposal generation.

Within aggregates, FS owns operational fields (dates, losses, hierarchy, resolution). FSV owns validation and grouping fields. Shared operational fields on FSV are synchronized from FS at commit.

## Related documents

- [`docs/design-principles.md`](../docs/design-principles.md)
- [`docs/internal/lifecycle-platform.md`](../docs/internal/lifecycle-platform.md)
