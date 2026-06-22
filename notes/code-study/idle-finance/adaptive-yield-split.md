# Adaptive Yield Split

The Adaptive Yield Split is the yield-distribution rule inside `IdleCDO`, applied at harvest when a period's yield is divided between the senior (AA) and junior (BB) tranches. It is the most advanced feature of the engine. Primary source: [Idle — Adaptive Yield Split fosters PYTs' liquidity scalability](https://medium.com/idle-finance/adaptive-yield-split-foster-pyts-liquidity-scalability-a796fa17ea35) and the [Adaptive Yield Split documentation](https://docs.idle.finance/products/yield-tranches/adaptive-yield-split).

## The prior fixed split and its limitations

Under a fixed yield split, risk and reward were misaligned in four respects, per the source article:

1. The senior tranche was underpaid, receiving "a yield lower than half of the underlying (5% APY) in most cases".
2. Coverage was asymmetric: when junior liquidity was low, the senior tranche surrendered yield for protection that was not yet present.
3. The junior tranche had a scalability ceiling, since a fixed split diluted junior returns below the underlying yield beyond a certain size.
4. Governance-token incentives concentrated in a single senior pool.

## The mechanism

The split is made a function of the coverage ratio — the relative size of the senior and junior tranches. In the engine this is the `trancheAPRSplitRatio`, the share of the pool's yield allocated to the senior tranche, recomputed from the senior-value ratio:

| Pool state | Senior share of yield | Effect |
|---|---|---|
| Low junior coverage (senior predominant) | High, approaching approximately 99% | The senior tranche earns close to the full underlying yield. |
| High junior coverage (large junior tranche) | Reduced, but floored at approximately 50% | The senior yield falls toward a guaranteed floor; the junior tranche captures up to half. |

The relationship is monotonic: the more protection the junior tranche provides, the more yield the senior tranche cedes in exchange, subject to a floor, while the junior tranche is designed always to earn above the underlying yield.

```
Senior yield  ▲
   ≈underlying │ ●───────────────●     senior predominant: senior ≈ full underlying yield
              │                  ╲
        floor │ ●─────────────────●    high junior coverage: senior at floor
              └───────────────────────▶  junior coverage (junior value / total)
```

The source article gives a worked illustration: a 2.5 million USD senior position at a 10% underlying yield, with a 30% governance-incentive allocation, realises approximately 7% when the senior tranche is fully covered by an equal junior balance and up to approximately 12% when the senior tranche predominates.

The approximately 99% cap and approximately 50% floor on `trancheAPRSplitRatio` are properties of the implementation and the Adaptive Yield Split documentation; the article describes the behaviour qualitatively and states no closed-form expression.

## Effect on liquidity and scalability

- The senior tranche earns a competitive organic yield independent of junior deposits, which removes the requirement to accumulate coverage before offering competitive returns.
- The junior tranche accepts arbitrary size, because the split re-prices as the junior tranche grows rather than diluting toward the underlying yield.
- Governance incentives can be distributed across more pools, since organic senior yields are already competitive.

## Risk model

The junior tranche is the loss-absorption layer for the senior tranche, taking loss first. The Adaptive Yield Split prices that protection: when junior coverage is high, the senior tranche accepts a lower, floored yield reflecting the protection then in place. The source identifies the mechanism as most valuable for higher-risk, higher-yield strategies, citing leveraged staked ether, where the senior tranche obtains a base yield while its capital is protected from liquidation risk.

## Classification

The Adaptive Yield Split is a yield-pricing mechanism, not a coverage test and not a waterfall. It adjusts the division of yield as a continuous function of coverage, but it does not redirect principal cash flow and contains no trigger that diverts cash flow to cure a breach; loss absorption remains passive. This classification is the central observation carried into [`mapping-to-standard.md`](./mapping-to-standard.md).
