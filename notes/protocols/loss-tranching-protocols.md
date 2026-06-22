# Loss-Tranching Protocols (Risk Axis)

Protocols implementing senior-and-junior subordination, the axis the proposed standard primarily addresses. Each is assessed against the [reference architecture](./reference-architecture.md), where ② denotes net-asset-value and loss accounting, ③ the waterfall, ④ the active coverage tests, and ⑤ the settlement layer.

## Idle Perpetual Yield Tranches

```
strategy → IdleCDO (tracks virtual price) → split by trancheAPRSplitRatio
        → AA token (ERC-20, senior)   BB token (ERC-20, junior, first-loss)
```

The BB tranche absorbs loss first; the AA tranche is affected only after the BB tranche's total value is exhausted, giving strict subordination. The `trancheAPRSplitRatio` adjusts the yield division as a function of the senior share of total value, which is a pricing mechanism rather than a coverage test. Provides ②, ③ (passive), and ⑤ (perpetual, ERC-4626-compliant); does not provide ④. Sources: [README](https://github.com/Idle-Labs/idle-tranches/blob/master/README.md), [Adaptive Yield Split](https://docs.idle.finance/products/yield-tranches/adaptive-yield-split). A full implementation-level review is in [`../code-study/idle-finance/`](../code-study/idle-finance/index.md).

## BarnBridge SMART Yield (discontinued)

The senior claim was issued as sBONDs (ERC-721; fixed, dated, and guaranteed) and the junior claim as jTokens (ERC-20; residual, first-loss), with a moving-average yield oracle setting the senior rate. The structure is notable for representing seniority as a non-fungible, dated instrument and juniority as a fungible, perpetual one. The protocol was discontinued in July 2023 and was the subject of a [Securities and Exchange Commission settlement](https://www.sec.gov/newsroom/press-releases/2023-258) in December 2023 — a regulatory outcome rather than a technical failure. Source: [SPEC.md](https://github.com/BarnBridge/BarnBridge-SmartYieldBonds/blob/master/SPEC.md).

## Centrifuge (DROP / TIN)

DROP (ERC-20, senior) and TIN (ERC-20, junior, first-loss) are issued against a pool valued as off-chain real-world-asset net asset value plus an on-chain reserve. A minimum-subordination-ratio test halts new DROP issuance when the TIN share falls below a floor, until the share is restored. This is the closest on-chain analogue to component ④, but the response is passive (issuance is suspended) rather than active (cash flow is diverted to de-lever the senior claim); that distinction is the gap the proposed standard addresses. Version 3 reissues the tranches as ERC-7540 vaults, a standard Centrifuge co-authored. Provides ②, ③, partial ④, and ⑤. Sources: [A Tale of Two Tokens](https://medium.com/centrifuge/a-tale-of-two-tokens-introducing-tin-drop-our-two-investment-tokens-d4c7342c799a), [RWA token standards](https://centrifuge.io/blog/rwa-token-standards). A full implementation-level review is in [`../code-study/centrifuge/`](../code-study/centrifuge/index.md).

## Goldfinch

The Senior Pool issues FIDU (ERC-20); Backers occupy the junior, first-loss position and hold PoolTokens (ERC-721). A leverage model sizes the senior position as a multiple of the junior, and a fixed 20% of senior nominal interest is reallocated to the junior position. This reallocation is the closest on-chain analogue to traditional excess-spread capture. Provides ② and ③; does not provide ④. Source: [Backers documentation](https://docs.goldfinch.finance/goldfinch/goldfinch-v1/protocol-mechanics/backers).

## Maple

Lender positions are ERC-4626 shares. The loss order is writedown, then borrower collateral, then liquidation of the cover (junior) position ahead of lenders, then recoveries. Cover is thin — on the order of 0–3% and often unused — which demonstrates that on-chain first-loss capital is structurally smaller than the 8–12% equity typical of traditional structures. This is a constraint for the specification rather than a mechanic. Source: [defaults and impairments](https://docs.maple.finance/maple-for-lenders/defaults-and-impairments).

## Saffron and Waterfall DeFi (dormant / defunct)

- **Saffron.** AA, A, and an auto-balancing S tranche, on 14-day epochs, with the junior position staking SFI as insurance at approximately ten-times leverage. Now dormant. Source: [introduction](https://medium.com/saffron-finance/introduction-to-saffron-9a46f2693612).
- **Waterfall DeFi.** A three-tranche structure (Senior, Mezzanine, Junior) with cash flow distributed top-down and loss absorbed bottom-up, on 7-day epochs. Now defunct (CertiK score 2.1/10). It demonstrates that a three-or-more-tranche sequential waterfall is implementable but was not sustained. Source: [what is tranching](https://waterfall-defi.gitbook.io/waterfall-defi/introduction/what-is-tranching).

## Assessment

The surveyed protocols provide non-decreasing or checkpointed net-asset-value accounting, strict junior-first loss exhaustion, dynamic coverage pricing, and subordination gating (Centrifuge). They do not provide active coverage-test cures, excess-spread capture beyond a fixed allocation, sustained three-or-more-tranche waterfalls, or adequately thick first-loss capital. See [`../mechanics/oc-ic-coverage-tests.md`](../mechanics/oc-ic-coverage-tests.md).
