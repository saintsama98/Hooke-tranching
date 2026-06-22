# The Assessor and Coordinator

These two contracts are the centerpiece of Tinlake and the closest existing on-chain expression of the mechanism the proposed standard targets. The Assessor values the tranches and defines the subordination constraint; the Coordinator executes each epoch's orders subject to that constraint.

## Assessor (`src/lender/assessor.sol`)

State (with `Fixed27` ray-scaled ratios, `1e27 = 100%`):

- `Fixed27 public seniorRatio` — the senior tranche's share of pool value, from the last executed epoch.
- `Fixed27 public minSeniorRatio` / `maxSeniorRatio` — the bounds the senior ratio must stay within (the subordination floor and cap).
- `uint256 public seniorDebt_` and `seniorBalance_` — together the total senior asset value; the debt portion accrues interest, the balance does not.
- `Fixed27 public seniorInterestRate` — the per-second rate at which senior debt accrues (DROP's protected return).

Behaviour:

- **`calcSeniorRatio`** is inherited from `Definitions`, not declared in this file. Its formula is `rdiv(seniorAsset, calcAssets(nav, reserve))`, where `calcAssets(nav, reserve) = nav + reserve`. In words: **seniorRatio = seniorAsset ÷ (NAV + reserve)** — the senior coverage ratio.
- **`dripSeniorDebt()`** accrues `seniorDebt_` forward at `seniorInterestRate` (via the inherited `Interest` math), so the senior tranche earns a fixed, time-based return.
- **`calcSeniorTokenPrice(nav, reserve)`** and **`calcJuniorTokenPrice(nav, reserve)`** derive the per-token prices; the junior price is the **residual** after the senior claim, so the junior tranche absorbs loss first.
- **`reBalance` / `changeSeniorAsset`** recompute `seniorRatio` whenever the senior asset value or pool value changes; `seniorRatioBounds()` exposes the min/max bounds for the Coordinator to read.

In standard terms, the Assessor is the on-chain form of component ② (valuation) plus the *measurement* half of component ④ (the coverage ratio and its bounds).

## Coordinator (`src/lender/coordinator.sol`)

The Coordinator runs the pool in epochs and decides which pending orders to execute.

- **`closeEpoch()`** — after the minimum epoch duration, it snapshots NAV, reserve, senior asset, and token prices, and reads the pending supply/redeem orders from both tranches. If the full set of orders satisfies all constraints, it executes immediately; otherwise a solution-submission window opens.
- **`submitSolution()`** — during the window, any party may propose fulfilment amounts. Solutions are scored by a weighted objective with a fixed priority: **senior redeem > junior redeem > junior supply > senior supply**. The best valid solution wins after a challenge period. This priority ordering is, in effect, the payment-priority of the waterfall.
- **`validate()`** — checks a candidate solution in layers:
  - *core constraints*: sufficient currency available, and each fulfilment within its order amount;
  - *pool constraints*: the reserve must not exceed `assessor.maxReserve()`, and the resulting senior asset must keep the senior ratio within `[minSeniorRatio, maxSeniorRatio]` (for example `seniorAsset < rmul(assets, minSeniorRatio)` is rejected);
  - a closing guard: when the junior token price reaches zero (junior exhausted), only redemptions are permitted.
- **Execution** calls `epochUpdate(...)` on each tranche with the fulfilment percentages and prices, and `changeSeniorAsset(...)` on the Assessor to record the new senior position.

In standard terms, the Coordinator is the on-chain form of component ③ (the waterfall) plus the *enforcement* half of component ④ — but enforced as a **constraint on which orders may execute**, not as a diversion of cash flow to cure a breach.

## The precise limit, relative to the standard

Tinlake enforces subordination by **gating order execution**: a solution that would push the senior ratio out of bounds is simply rejected, so junior redemptions are throttled to keep the senior tranche covered. It does **not** take cash that would have flowed to the junior tranche and actively apply it to de-lever the senior tranche to *cure* an existing breach. That active-cure behaviour is the increment the proposed standard adds; see [`mapping-to-standard.md`](./mapping-to-standard.md) and [`../../mechanics/oc-ic-coverage-tests.md`](../../mechanics/oc-ic-coverage-tests.md).
