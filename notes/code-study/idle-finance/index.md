# Idle Finance — Code Study

An implementation-level review of Idle Finance's Perpetual Yield Tranches, the closest production analogue to the engine the proposed standard specifies. Repository: [`Idle-Labs/idle-tranches`](https://github.com/Idle-Labs/idle-tranches).

The study is organised so that each concern is treated in isolation and mapped, in the final note, onto the clauses of [`../../../spec/erc-waterfall-tranche.md`](../../../spec/erc-waterfall-tranche.md).

- [`overview.md`](./overview.md) — The protocol, its current scale, the Pareto transition, and the repository structure
- [`idle-cdo-engine.md`](./idle-cdo-engine.md) — The `IdleCDO` engine: tranche tokens, net-asset-value accounting, deposit and withdrawal, and loss allocation
- [`adaptive-yield-split.md`](./adaptive-yield-split.md) — The Adaptive Yield Split: a coverage-priced dynamic yield division
- [`ethena-integration.md`](./ethena-integration.md) — The sUSDe integration: native staking, the redemption cooldown, and the per-request clone pattern
- [`mapping-to-standard.md`](./mapping-to-standard.md) — How each mechanism maps onto, and differs from, the proposed standard

## Summary judgement

Idle implements components ②, ③, and ⑤ of the [reference architecture](../../protocols/reference-architecture.md): net-asset-value accounting by a checkpointed virtual price, a senior-and-junior split with strict junior-first loss exhaustion, and perpetual or epoch-based settlement. Its most advanced feature, the Adaptive Yield Split, prices the protection that the junior tranche extends to the senior tranche, but it does so on the yield-distribution axis only. It contains no coverage test and no active diversion of cash flow (component ④), which remains the increment the proposed standard introduces.
