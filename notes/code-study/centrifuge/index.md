# Centrifuge — Code Study

An implementation-level review of Centrifuge, the on-chain real-world-asset securitisation protocol, across its two generations. Centrifuge is the design closest to the proposed standard, because it is the only surveyed protocol that enforces a subordination constraint in explicit on-chain logic.

The study is organised so that each concern is treated in isolation and mapped, in the final note, onto the clauses of [`../../../spec/erc-waterfall-tranche.md`](../../../spec/erc-waterfall-tranche.md).

- [`overview.md`](./overview.md) — The two generations (Tinlake and Liquidity Pools v3), the repositories, and where to study which concern
- [`tinlake-architecture.md`](./tinlake-architecture.md) — The Tinlake lender module: contract roles, the inheritance/dependency base, and the per-pool deployment model
- [`assessor-and-coordinator.md`](./assessor-and-coordinator.md) — The valuation/coverage core and the epoch solver (the centerpiece)
- [`liquidity-pools-v3.md`](./liquidity-pools-v3.md) — The current ERC-7540 layer, and the consequence of moving the coverage math off-chain
- [`contract-map.md`](./contract-map.md) — Which contracts are relevant to tranching, what to ignore, and a reading order
- [`mapping-to-standard.md`](./mapping-to-standard.md) — How each mechanism maps onto, and differs from, the proposed standard

## Summary judgement

Tinlake implements components ②, ③, and ⑤ of the [reference architecture](../../protocols/reference-architecture.md) with the most explicit on-chain coverage logic in the survey: the **Assessor** computes a senior ratio and enforces `minSeniorRatio`/`maxSeniorRatio` bounds, and the **Coordinator** executes epoch orders subject to those bounds in a fixed priority. This is coverage expressed as a *constraint on order execution*, not the active *diversion of cash flow to cure a breach* that component ④ specifies; that distinction remains the increment the proposed standard introduces. The successor (Liquidity Pools v3) reissues tranches as ERC-7540 vaults but relocates the coverage and pricing math off-chain, which is the opposite of the verifiable-on-chain direction the standard pursues.

Related: [`../idle-finance/`](../idle-finance/index.md) (the other primary code study) and the conceptual survey in [`../../protocols/loss-tranching-protocols.md`](../../protocols/loss-tranching-protocols.md).
