# Mapping to the Proposed Standard

This note maps Idle's mechanisms onto the components of the [reference architecture](../../protocols/reference-architecture.md) and the clauses of [`../../../spec/erc-waterfall-tranche.md`](../../../spec/erc-waterfall-tranche.md), and states precisely what the standard adds.

## Component mapping

| Component | Idle mechanism | Standard clause |
|---|---|---|
| ① Strategy | `IIdleCDOStrategy` adapter and `IdleCDO<Strategy>Variant` | Composed yield source, outside the interface |
| ② Net-asset-value and loss accounting | Per-tranche virtual price, updated on harvest | `totalAssets()`, `trancheAssets()` |
| ③ Waterfall | Two-tranche split; strict junior-first loss exhaustion | `distribute()`, `allocateLoss()` |
| ④ Coverage tests and active cure | None | `coverageRatio()`, `isBreached()`, `Diversion` |
| ⑤ Settlement | Perpetual, or epoch-based in the Epoch and Ethena variants | Composition with ERC-4626 and ERC-7540 |
| Tranche tokens | `IdleCDOTranche` (one ERC-20 per tranche) | `trancheToken()` |

## Two precise distinctions

**The Adaptive Yield Split is a pricing mechanism, not a coverage test.** It adjusts `trancheAPRSplitRatio`, the division of yield, as a continuous function of coverage. It does not measure a ratio against a trigger, and it does not redirect principal cash flow on a breach. In the standard's terms it has no analogue of `isBreached()` or the `Diversion` event; it operates on the yield-distribution axis only. The standard's coverage tests are orthogonal to it, and the two could coexist: a deployment could price yield by an Adaptive-Yield-Split-style rule while also enforcing coverage tests that divert cash flow.

**Loss absorption is passive.** A loss reduces the junior virtual price directly. There is no point at which cash flow that would reach the junior tranche is instead applied to de-lever the senior tranche. This is the increment the standard introduces in clause `distribute()` rule 3 and the `Diversion` event.

## What the standard adds beyond Idle

1. **An ordered, n-tranche model.** Idle implements one senior-junior boundary. The standard specifies an arbitrary number of tranches under a strictly ordered seniority index.
2. **Coverage tests with active cure.** The overcollateralisation and interest-coverage tests, and the diversion of cash flow on breach, which Idle does not implement.
3. **Governable, auditable calibration.** Triggers exposed as parameters with events, per [`../../mechanics/heuristics-vs-plumbing.md`](../../mechanics/heuristics-vs-plumbing.md).

## What the standard should adopt from Idle

1. **The checkpointed virtual price** as a concrete, workable form of component ②.
2. **The strict junior-first exhaustion rule**, which the standard generalises to reverse-seniority loss allocation.
3. **The engine-and-adapter separation**, which keeps the engine asset-agnostic and is the basis for the single-asset denomination confirmed in [`ethena-integration.md`](./ethena-integration.md).
4. **The evidence for asynchronous settlement.** The Ethena cooldown demonstrates that redemption latency is a real and recurring property of yield sources, supporting composition with ERC-7540 for the settlement layer.
