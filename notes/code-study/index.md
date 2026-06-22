# Code Study

Implementation-level studies of specific protocol codebases. This section is distinct from [`../protocols/`](../protocols/index.md): the protocols section is a conceptual survey assessing each design against the [reference architecture](../protocols/reference-architecture.md), whereas this section examines the source code of a single protocol in depth and maps its concrete mechanisms onto the clauses of the proposed standard.

Each protocol studied is given its own subdirectory, so that the section scales as further codebases are reviewed.

## Modules

- [`idle-finance/`](./idle-finance/index.md) — Idle Finance: the `IdleCDO` engine, the Adaptive Yield Split, and the Ethena (sUSDe) integration.
- [`centrifuge/`](./centrifuge/index.md) — Centrifuge: the Tinlake lender module (Assessor + Coordinator subordination logic) and the v3 ERC-7540 layer.

## Planned

- Pendle (`pendle-core-v2-public`) — the SY wrapper and the principal/yield split engine.
