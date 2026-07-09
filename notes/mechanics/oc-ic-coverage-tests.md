# Overcollateralisation and Interest-Coverage Tests

The two coverage tests of collateralised-loan-obligation and securitisation practice convert a waterfall from a passive distribution into an active one. This mechanic is absent from the surveyed on-chain protocols.

## Overcollateralisation (OC) test

A test of whether sufficient collateral principal supports the senior claim.

```
OC ratio = (par value of the collateral pool) / (par value of senior + this tranche)
Example:  100M / 80M = 1.25, above a 1.10 trigger
```

This is a principal, or asset-value, test.

## Interest-coverage (IC) test

A test of whether the pool generates sufficient income to service the senior coupon.

```
IC ratio = (interest collected) / (interest due to the senior claim)
Example:  8M / 4M = 2.0, above a 1.20 trigger
```

This is an income test.

## The active-cure mechanic

On breach of either test, the waterfall re-routes cash flow: distributions that would otherwise reach the junior or equity layer are redirected to repay senior principal until the ratio returns above its trigger, which cures the test.

```
Compliant:  senior coupon → junior remainder
Breached:   senior coupon → de-lever senior (junior distributions suspended) → cure
```

In stress, the senior layer therefore receives a smaller distribution while the junior layer absorbs the shortfall first — the protection that tranching is intended to provide.

## On-chain status

| Behaviour | Present on-chain |
|---|---|
| Passive loss absorption (junior net asset value declines) | Yes, generally |
| Suspension of new senior issuance on breach | Centrifuge only |
| Periodic OC/IC computation with cash-flow diversion to cure | None |
| Excess-spread capture (residual trapped before equity) | Approximated only by Goldfinch's fixed 20% allocation |
| Three-or-more-tranche sequential waterfall | Attempted by Waterfall DeFi (now defunct) |

Evidence is collected in [`../protocols/loss-tranching-protocols.md`](../protocols/loss-tranching-protocols.md). The normative core of the proposed standard is the sequence: compute the coverage ratios each epoch, evaluate them against governable triggers, and re-route the waterfall on breach. The triggers are calibration parameters rather than constants; see [`heuristics-vs-plumbing.md`](./heuristics-vs-plumbing.md).

The at-scale prior-art benchmark for this mechanic — the per-tranche coverage ladder, the reinvestment-period interest-diversion cure, and the calibration levels — is recorded in [`clo-reference-implementation.md`](./clo-reference-implementation.md).
