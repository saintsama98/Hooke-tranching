# Research Notes — Index

This directory contains the directional research underpinning the proposed **Waterfall Tranching Standard**. The material is organised by topic and intended to be read in sequence; conclusions are distilled into [`../spec/`](../spec/) as they mature.

Conventions are described in [`README.md`](./README.md): one concept per file, kebab-case filenames, liberal cross-linking, and inline citation of primary sources. All theoretical research is maintained within `notes/`.

## 0. Orientation

- [`00-thesis-and-direction.md`](./00-thesis-and-direction.md) — A summary of the gap the standard addresses and the principle that governs its design. Recommended starting point.

## 1. Standards landscape — `standards/`

A survey of existing standards: what each specifies, and the point at which each stops short of tranching or waterfall logic.

- [`standards/index.md`](./standards/index.md) — Section overview
- [`standards/timeline-and-reading-order.md`](./standards/timeline-and-reading-order.md) — Chronological sequence and dependency-ordered reading path
- [`standards/token-substrate-layer.md`](./standards/token-substrate-layer.md) — ERC-20, ERC-1155, ERC-6909, ERC-3475
- [`standards/vault-and-yield-layer.md`](./standards/vault-and-yield-layer.md) — ERC-4626, ERC-7540, ERC-7575, EIP-5115, EIP-5095
- [`standards/storage-substrate.md`](./standards/storage-substrate.md) — ERC-7201 namespaced storage
- [`standards/security-token-branch.md`](./standards/security-token-branch.md) — ERC-1400/1410, ERC-3643, ERC-7518

## 2. Financial mechanics — `mechanics/`

The behaviour that existing standards do not specify.

- [`mechanics/index.md`](./mechanics/index.md) — Section overview
- [`mechanics/waterfall-tranching-primer.md`](./mechanics/waterfall-tranching-primer.md) — Subordination, priority of payments, loss allocation
- [`mechanics/oc-ic-coverage-tests.md`](./mechanics/oc-ic-coverage-tests.md) — Overcollateralisation and interest-coverage tests
- [`mechanics/heuristics-vs-plumbing.md`](./mechanics/heuristics-vs-plumbing.md) — Deterministic logic versus parameterised calibration

## 3. On-chain prior art — `protocols/`

Each surveyed protocol implements a subset of the reference architecture.

- [`protocols/index.md`](./protocols/index.md) — Section overview
- [`protocols/reference-architecture.md`](./protocols/reference-architecture.md) — The abstract reference architecture
- [`protocols/loss-tranching-protocols.md`](./protocols/loss-tranching-protocols.md) — Idle, BarnBridge, Centrifuge, Goldfinch, Maple, Saffron, Waterfall DeFi
- [`protocols/cashflow-tranching-protocols.md`](./protocols/cashflow-tranching-protocols.md) — Pendle, Spectra
- [`protocols/rwa-securitization-entrants.md`](./protocols/rwa-securitization-entrants.md) — Untangled, Galaxy CLO, Tradable
- [`protocols/standards-adoption-matrix.md`](./protocols/standards-adoption-matrix.md) — Standards adoption across surveyed protocols

## 4. Code study — `code-study/`

Implementation-level reviews of individual protocol codebases, each mapped onto the proposed standard.

- [`code-study/index.md`](./code-study/index.md) — Section overview
- [`code-study/idle-finance/`](./code-study/idle-finance/index.md) — Idle Finance: the `IdleCDO` engine, the Adaptive Yield Split, and the Ethena (sUSDe) integration
- [`code-study/centrifuge/`](./code-study/centrifuge/index.md) — Centrifuge: the Tinlake lender module (Assessor + Coordinator) and the v3 ERC-7540 layer

## 5. Synthesis — `synthesis/`

- [`synthesis/index.md`](./synthesis/index.md) — Section overview
- [`synthesis/substrate-fork.md`](./synthesis/substrate-fork.md) — Token-substrate selection: ERC-3475 versus ERC-6909 with ERC-7201
- [`synthesis/open-questions-and-next.md`](./synthesis/open-questions-and-next.md) — Outstanding research questions

## 6. Harness design — `hooke-centrifuge/`

The generic credit harness (Centrifuge-style use case) into which the waterfall engine plugs; the concrete substrate the spec interface is distilled from.

- [`hooke-centrifuge/README.md`](./hooke-centrifuge/README.md) — Section overview
- [`hooke-centrifuge/harness-architecture.md`](./hooke-centrifuge/harness-architecture.md) — Architecture document, with validated Mermaid diagrams in [`hooke-centrifuge/diagrams/`](./hooke-centrifuge/diagrams/)

---

*Status: directional research, in progress. Last consolidated 2026-06-19.*
