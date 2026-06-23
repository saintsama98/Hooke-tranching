# hooke-centrifuge/

Design of the **generic credit harness** — the smallest scaffold for a tranched private-credit pool (Centrifuge-style use case), into which a waterfall engine is plugged. The harness is specified here; the waterfall is an external seam, specified separately.

- [`harness-architecture.md`](./harness-architecture.md) — the architecture document
- [`diagrams/`](./diagrams/) — Mermaid sources (`.mmd`) and rendered SVGs, validated with mermaid-cli:
  - `00-overview-generic` — the complete harness, all components and both seams
  - `01-component-architecture` — contracts and interfaces
  - `02-capital-flow` — deposit / redeem
  - `03-loan-lifecycle` — fund / repay / default, and the route to the waterfall seam
  - `04-seam-boundaries` — the NAV and waterfall seams and what crosses each
  - `05-protocol-synthesis` — how surveyed protocols map onto the generic slots

This is the concrete substrate from which the [spec interface](../../spec/erc-waterfall-tranche.md) is to be distilled.
