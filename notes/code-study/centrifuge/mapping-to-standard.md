# Mapping to the Proposed Standard

This note maps Centrifuge's mechanisms onto the [reference architecture](../../protocols/reference-architecture.md) and the clauses of [`../../../spec/erc-waterfall-tranche.md`](../../../spec/erc-waterfall-tranche.md), and states what the standard takes and what it adds.

## Component mapping (Tinlake)

| Component | Tinlake mechanism | Standard clause |
|---|---|---|
| ① Strategy / asset source | Borrower loans + NAV feed | Composed source, outside the interface |
| ② NAV / loss accounting | `assessor` valuation; pool value = NAV + reserve | `totalAssets()`, `trancheAssets()` |
| ③ Waterfall | `coordinator` priority ordering (senior redeem > junior redeem > junior supply > senior supply) | `distribute()`, `allocateLoss()` |
| ④ Coverage tests + cure | `assessor` senior ratio + `min/maxSeniorRatio`; `coordinator.validate()` enforces them — **as a constraint, not an active cure** | `coverageRatio()`, `isBreached()`, `Diversion` |
| ⑤ Settlement | Epochs (`closeEpoch`); v3 reissues as ERC-7540 vaults | Composition with ERC-4626 / ERC-7540 |
| Tranche tokens | DROP / TIN restricted ERC-20 (v3: `Tranche.sol`) | `trancheToken()` |

## The two precise distinctions

**Coverage is a constraint, not an active cure.** Tinlake's Coordinator rejects any epoch solution that would push the senior ratio outside `[minSeniorRatio, maxSeniorRatio]`, which throttles junior redemptions to keep the senior tranche covered. It does not take cash that would reach the junior tranche and apply it to repay senior principal to *cure* a breach already in progress. The standard's `distribute()` rule 3 and the `Diversion` event specify exactly that active redirection, which Tinlake does not perform. This is the single clearest "closest, but not the same" finding in the survey.

**The modern layer pushed the math off-chain.** Tinlake's coverage logic is on-chain and auditable. Centrifuge v3 relocated the NAV, pricing, and epoch-solving to the off-chain Centrifuge chain, leaving the EVM contracts to hold tokens and apply delivered prices. The proposed standard's premise is the opposite: keep the coverage engine on-chain and verifiable.

## The deployment fork

Centrifuge and Idle take opposite layouts for the same senior/junior concept:

- **Centrifuge (Tinlake):** one contract *per tranche* (two `tranche.sol` instances) plus a shared `coordinator` and `assessor` above them; tranches are first-class contracts, deployed per pool by a factory layer.
- **Idle:** one engine (`IdleCDO`) holding both tranches as two token addresses; tranches are plain ERC-20s indexed by the engine.

This is a real interface decision for the standard: does a tranche expose its own contract (Centrifuge-style — `trancheToken()` could be a full vault), or is it an `id`/token the engine indexes (Idle-style)? It connects directly to [`../../synthesis/substrate-fork.md`](../../synthesis/substrate-fork.md).

## What the standard should adopt from Centrifuge

1. **The explicit senior-ratio model with governable bounds** (`seniorRatio` against `minSeniorRatio`/`maxSeniorRatio`) — the cleanest on-chain expression of subordination, and a direct model for `coverageRatio()`/`isBreached()`.
2. **The fixed-priority order of execution** (senior redeem first, senior supply last) — a concrete payment-priority ordering.
3. **Senior debt accruing a per-second rate** (`dripSeniorDebt`) — a workable form of a protected senior return.
4. **Junior price as the residual** — a clean realisation of junior-first loss.
5. **ERC-7540 composition** for the settlement layer, as v3 demonstrates the async model requires.

## What the standard adds beyond Centrifuge

1. **Active cure**, not just constraint: divert cash flow to de-lever the senior tranche on a breach, rather than only rejecting order combinations that would breach.
2. **An arbitrary number of ordered tranches**, where Tinlake fixes two (DROP/TIN).
3. **On-chain, verifiable coverage at the modern layer** — the part v3 moved off-chain.
4. **An interest-coverage test** alongside overcollateralisation; Tinlake's constraint is overcollateralisation-style (a ratio of assets), with no separate income-coverage test.
