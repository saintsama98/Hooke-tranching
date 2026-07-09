# hooke-centrifuge/

Design of the **generic credit harness** — the smallest scaffold for a tranched private-credit pool (Centrifuge-style use case), into which a waterfall engine is plugged. The harness is specified here; the waterfall is an external seam, specified separately.

But this approach has just started to mature, complete development will be added later and consider this as a singleton starter for the  architecture for final interface.

- [`harness-architecture.md`](./harness-architecture.md) — the architecture document
- [`diagrams/`](./diagrams/) — Mermaid sources (`.mmd`) and rendered SVGs, validated with mermaid-cli:

  - `01-seam-boundaries` — the NAV and waterfall seams and what crosses each
  - `02-protocol-synthesis` — how surveyed protocols map onto the generic slots

This is the concrete substrate from which the [spec interface](../../spec/erc-waterfall-tranche.md) is to be distilled.
