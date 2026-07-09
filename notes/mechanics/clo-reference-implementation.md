# The Collateralised-Loan-Obligation Reference Implementation

The collateralised loan obligation is the canonical, at-scale implementation of the active-cure coverage-test mechanic defined in [`oc-ic-coverage-tests.md`](./oc-ic-coverage-tests.md). It is the structure against which the on-chain protocols surveyed in [`../protocols/loss-tranching-protocols.md`](../protocols/loss-tranching-protocols.md) are found wanting: it computes coverage ratios at multiple levels of the capital structure and re-routes cash flow on breach, the component (④) that no surveyed on-chain protocol provides. This note records its structure and parameters as the prior-art benchmark for the proposed standard.

Figures below are typical ranges drawn from educational and manager sources rather than a single indenture; specific deals vary, and the ranges are indicative.

## Capital structure

A CLO is a managed securitisation of below-investment-grade corporate (leveraged) loans, arranged into a rated capital stack (Guggenheim, *Understanding CLOs*):

```
  AAA senior      ~65% of the stack   lowest coupon, most loss-remote, paid first
  Mezzanine       ~4-12% per tranche  AA / A / BBB / BB, rising coupon and risk
  Equity          ~8-10%              unrated, non-coupon, residual, first-loss
```

Two waterfalls operate in parallel — an interest waterfall and a principal waterfall — each distributing top-down from the senior-most tranche to the equity, which receives only the excess remaining once every debt obligation is met.

## The coverage-test ladder

Unlike the single senior test described in [`oc-ic-coverage-tests.md`](./oc-ic-coverage-tests.md), a CLO computes an overcollateralisation and an interest-coverage test at **each rated level** of the stack, each with its own trigger. The tests use the formulas already recorded in that note, evaluated per tranche ([collateralizedloanobligations.com](https://collateralizedloanobligations.com/mechanics/coverage-tests)):

```
OC ratio (tranche T) = (par value of the collateral pool)
                       / (par value of liabilities senior to and including T)

IC ratio (tranche T) = (interest collected from the collateral)
                       / (interest due on liabilities senior to and including T)
```

Indicative trigger levels by rated tranche:

| Tranche | OC trigger | IC trigger |
|---|---|---|
| AAA | 127–132% | ~120–130% (≈125% typical) |
| AA | 118–122% | — |
| A | 113–117% | — |
| BBB | 108–112% | — |
| BB | 103–107% | — |

Worked example (source): a 500M pool against a 300M AAA tranche gives an AAA OC ratio of 500/300 = 167% against a 130% trigger — a pass. On the income side, a pool earning SOFR + 450 bps against a 300M AAA tranche paying SOFR + 135 bps produces roughly 47.5M of income against 19.05M of expense, an IC ratio near 249% against a 125% trigger.

## The active cure

On breach of either test at a given level, interest and principal that would otherwise flow to the equity and junior-debt tranches are diverted to repay senior principal until the ratio returns above its trigger. During a breach, *"equity and junior debt receive 0"* until the tests cure ([collateralizedloanobligations.com](https://collateralizedloanobligations.com/mechanics/coverage-tests)). This is the same active-cure sequence the proposed standard defines as its normative core.

## The reinvestment period and the second cure path

A CLO has a **reinvestment period**, typically up to five years, during which the manager *"is permitted to actively trade underlying assets … and uses principal cash flow from underlying assets to purchase new assets"* (Guggenheim). This introduces a second, distinct cure path that a purely static structure lacks:

- **After reinvestment** — the cure is senior paydown (divert cash to de-lever the senior claim).
- **During reinvestment** — a higher-threshold OC test (the interest-diversion test) redirects a portion of interest proceeds to **purchase additional collateral** rather than to pay down debt, curing the ratio by rebuilding the numerator instead of shrinking the denominator. (A commonly-cited cap of up to 50% of interest proceeds appears in secondary sources but was not confirmed against a primary indenture here.)

The manager is *"not [a] forced seller … and can actively trade loans to capture value or minimise losses"* (Guggenheim) — an active-management degree of freedom that has no direct analogue in the passive on-chain structures.

## Mapping to the reference architecture

Assessed against [`../protocols/reference-architecture.md`](../protocols/reference-architecture.md):

| Component | CLO provision |
|---|---|
| ① Asset pool / strategy | A managed portfolio of leveraged loans; actively traded during reinvestment |
| ② Net-asset-value and loss accounting | Par-based collateral valuation, with defaulted and low-rated collateral haircut before entering the ratio numerator |
| ③ Waterfall | Separate interest and principal waterfalls, distributed top-down |
| ④ Coverage tests and active cure | **Fully present** — a per-tranche OC/IC ladder with two cure paths; the component absent on-chain |
| ⑤ Settlement | A managed cash-flow structure with defined payment dates, not an epoch vault |

## Lessons for the proposed standard

1. **A coverage-test ladder, not a single test.** Each liability level carries its own OC and IC trigger. A standard that specifies only a single senior test under-describes the reference structure; the engine should admit a test per tranche boundary.
2. **First-loss thickness is the binding constraint on-chain.** CLO equity is approximately 8–10% of the structure, against the 0–3% cover observed on-chain (per [`../protocols/loss-tranching-protocols.md`](../protocols/loss-tranching-protocols.md), Maple). The specification should treat first-loss thickness as an explicit, governable parameter and surface the shortfall.
3. **The re-route step may admit more than one cure action.** The reference structure cures either by de-levering the senior claim or by rebuilding collateral, depending on the phase. A standard that permits only senior paydown captures one of the two.
4. **Triggers are calibration, not constants.** The trigger table varies by rating and by deal; this is consistent with the governing principle in [`heuristics-vs-plumbing.md`](./heuristics-vs-plumbing.md) — specify the ratio computation and the re-route deterministically, and expose every trigger as an explicit, auditable parameter.

## Sources

- [collateralizedloanobligations.com — Coverage Tests](https://collateralizedloanobligations.com/mechanics/coverage-tests) — OC/IC formulas, indicative trigger table, worked example, breach mechanics.
- [Guggenheim Investments — Understanding Collateralized Loan Obligations](https://www.guggenheiminvestments.com/perspectives/portfolio-strategy/understanding-collateralized-loan-obligations-clo/) — capital structure, reinvestment period, waterfall, manager role.
