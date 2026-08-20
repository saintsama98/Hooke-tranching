# Sources

An annotated bibliography. Every link was checked at the time of writing. Where a piece
is old enough to describe an earlier state of the market, the date says so, which is
often the reason to read it.

Nothing here is endorsed. Several of these are written by parties with a position in what
they describe, and that is noted where it applies. Sourcing conventions are set out in
[`METHOD.md`](./METHOD.md).

## Protocol documentation

The primary sources. Read these before any commentary about the same protocol.

- [Royco](https://docs.royco.org/) — Senior, junior and senior liquidity provider legs.
  The only three leg structure covered here.
- [Strata](https://docs.strata.markets/) — The Dynamic Yield Split, coverage bands and
  the exit ladder. The parameter set is published per market. The
  [Dynamic Yield Split page](https://docs.strata.markets/protocol-mechanism/dynamic-yield-split)
  carries the formulas and the published values of x, y and k.
- [Re](https://docs.re.xyz/) — Reinsurance capital, a three layer stack, reUSD and
  reUSDe. The [full documentation corpus](https://docs.re.xyz/llms-full.txt) is the
  quickest route to the capital structure and the stated loss probabilities.
- [Idle yield tranches](https://docs.idle.finance/products/yield-tranches/overview) — The
  engine that became Pareto, and the
  [Adaptive Yield Split](https://docs.idle.finance/products/yield-tranches/adaptive-yield-split),
  which remains the clearest published argument for pricing subordination by coverage.
- [Pendle](https://docs.pendle.finance/pendle-v2/Introduction) — Principal and yield
  splitting, and the standardised yield wrapper underneath it.
- [Centrifuge](https://docs.centrifuge.io/) — Real world asset pools. The
  [multi tranche system](https://docs.centrifuge.io/learn/multi-tranche-system/) page is
  the one that matters for subordination, and
  [deRWA tokens](https://centrifuge.io/derwa-tokens) for the liquidity approach.
- [Pareto](https://docs.pareto.credit/product/usp) — The protocol formerly called Idle.
  The USP page is the entry point to the current credit vault products.
- [Maple](https://docs.maple.finance/maple-for-lenders/defaults-and-impairments) — How
  defaults, impairments and the cover position are handled.
- [Untangled](https://docs.untangled.finance/docs/credio/Onchain-Private-Credit/Intro-Untangled-Pool/)
  — Senior and junior obligation tokens with a credit oracle.

## Source code

- [Royco Dawn](https://github.com/roycoprotocol/royco-dawn/) — Factory, kernel,
  accountant, two tranche vaults, three yield distribution models.
- [Royco IAM](https://github.com/roycoprotocol/royco-iam) — The earlier incentive market
  product, kept here because the name causes confusion.
- [Idle tranches](https://github.com/Idle-Labs/idle-tranches) — The engine most often
  read as a reference.
- [Centrifuge Tinlake](https://github.com/centrifuge/tinlake) — The subordination logic,
  and the codebase the implementation in this repository is derived from.
- [Spectra](https://github.com/perspectivefi/spectra-core) — Principal and yield
  splitting over interest bearing tokens.
- [Re smart contract addresses](https://docs.re.xyz/protocol/smart-contract-addresses) —
  Deployments across Ethereum, Avalanche and Arbitrum.

## Independent reviews and audits

The most useful category and the thinnest. An outside party with no position, looking at
a live deployment, is worth more than any amount of protocol documentation.

- [Yearn risk report on srRoyUSDC](https://github.com/yearn/risk-score/blob/master/reports/report/royco-srroyusdc.md)
  — A full assessment of a live Royco senior vault, scored elevated risk. Covers holder
  concentration, custody, coverage thickness and the source of the valuation. Discussed
  in [`10-study-royco`](./10-study-royco/README.md#an-outside-review-reached-an-unflattering-conclusion).
- [Hexens audit of the Royco risk tranching protocol, January 2026](https://hexens.io/audit-reports/royco-perpetual-risk-tranching-protocol-jan-2026)
  — One of several. The full list is on the [Royco security page](https://www.royco.org/security).
- [Royco bug bounty on Immunefi](https://immunefi.com/bug-bounty/royco/information/) —
  Scope and severity definitions, which are often more informative about a protocol's own
  view of its risks than its documentation.
- [Hacken audit of Re](https://hacken.io/audits/re-protocol/sca-re-re-defi-aug2024/) —
  Notable for recording branch coverage of 42.11 percent, which is the kind of figure
  worth reading before the conclusion.
- [Strata risks and mitigations](https://docs.strata.markets/protocol-mechanism/risks-and-mitigations)
  — A protocol's own risk statement, useful mainly for what it chooses to enumerate.
- [Hindenrank on Pareto Credit](https://hindenrank.com/blog/how-does-pareto-credit-work) —
  An outside risk analysis grading the protocol C plus, with a clear statement of the
  maturity mismatch between a demand liability and a term asset.

## Optimizations and the frontier

The material behind [`06-optimizations`](./06-optimizations/README.md). This is where the
unsettled work is, and most of it is a forum thread or a preprint rather than
documentation.

- [Building index tracking assets on top of options instead of debt](https://ethresear.ch/t/building-index-tracking-assets-on-top-of-options-instead-of-debt/25036),
  Ethereum Research, 2026 — Vitalik Buterin's proposal, and more importantly the replies.
  Norswap on whether the construction tracks anything, Equivrel on theta bleed, Czar102 on
  short dated rolls and oracleless physical settlement, Xatarrer on a perpetual variant,
  The Risk Protocol on barriers, and a live physically settled implementation on Base. Read
  the whole thread rather than the summary coverage.
- [Institutionalizing risk curation in decentralized credit](https://arxiv.org/pdf/2512.11976),
  Zbandut and Goldstein, arXiv 2512.11976, December 2025 — Curators posting first loss
  capital, and the contagion channels that creates. The only peer style academic treatment
  of the junior demand problem found in this research.
- [Symbiotic slashing insurance vaults](https://resources.symbiotic.fi/) — Tranching a risk
  that settles observably on chain, which sidesteps the valuation problem entirely.
  Premiums set from expected excess loss.
- [Single tranche CDO](https://en.wikipedia.org/wiki/Single-tranche_CDO) and
  [Consistent valuation of bespoke CDO tranches](https://arxiv.org/pdf/1004.1758) — What a
  correlation market looks like when one exists, and what happened to the last one.
- [CDO tranche sensitivities in the Gaussian copula model](https://www.math.lsu.edu/~sengupta/papers/MengSenguptaTranche08.pdf)
  — The formal treatment behind the identity that a tranche is a spread on the loss
  distribution.
- [Pendle principal token looping on Morpho](https://cryptobriefing.com/pendle-finance-pt-usdai-pt-susdd-morpho-27-apy/),
  August 2026 — Looping at industrial scale against collateral with a known redemption
  value on a known date.
- [Keyrock on onchain structured product strategies](https://keyrock.com/knowledge-hub/onchain-structured-product-strategies-a-guide/)
  — Survey of the payoff engineering side, including principal protected structures.

## Analysis and market context

- [Onchain tranching is positioning itself as the future of structured finance in DeFi](https://cryptobriefing.com/onchain-tranching-structured-finance-defi/),
  Crypto Briefing, July 2026 — Reports the analysis by Silvio Busonero of Blockworks.
  The source of the two figures worth remembering: looping drives about 40 percent of
  DeFi lending revenues, and tranching is under 1 percent of lending deposits.
- [Blockworks Insights](https://blockworks.com/insights) — Where the underlying research
  is published.
- [Risk as a Product](https://dialecticgroup.substack.com/p/risk-as-a-product), Dialectic,
  July 2026 — A curator explaining why they use tranching, including using a senior
  tranche as better collateral. Written in collaboration with Royco, so read it as a
  position rather than a survey.
- [Tranched Credit Markets in DeFi: Where Things Stand in 2026](https://defiprime.com/tranched-credit-markets),
  defiprime, March 2026 — A survey of ten protocols across EVM chains and Solana,
  including Credix and Tranched.fi, which this repository does not cover.
- [DeFi Drop: Royco Dawn and Risk Tranching](https://blog.portals.fi/defi-drop-royco-dawn-risk-tranching/),
  Portals — Plain explanation of the observation period and the adaptive rate model, with
  usage figures.
- [Strata initial review](https://serenityresearch.substack.com/p/serenity-premium-strata-finance-initial),
  Serenity Research, May 2026 — Outside reading of Strata, alongside the study in
  [`09-study-strata`](./09-study-strata/README.md).
- [Strata, bringing risk tranching to DeFi yields](https://alearesearch.substack.com/p/strata-risk-tranching),
  Aleа Research — A second outside account of the same protocol, useful for triangulation.
- [Onchain protocol Re secures strategic investment from Coinbase Ventures](https://www.reinsurancene.ws/onchain-protocol-re-secures-strategic-investment-from-coinbase-ventures/),
  Reinsurance News — Worth noting that the trade press covering Re is the insurance press
  rather than the crypto press.
- [Centrifuge launches JAAA, the largest tokenised AAA CLO fund](https://cryptobriefing.com/centrifuge-jaaa-tokenized-clo-fund/)
  and [the Grove allocation](https://centrifuge.io/blog/centrifuge-janus-henderson-grove-tokenized-aaa-clo-fund)
  — The scale and composition of the largest tokenised claim on a real waterfall.
- [JAAA on RWA.xyz](https://app.rwa.xyz/assets/JAAA) — Independent asset level data on the
  same fund.
- [FalconX credit vault](https://www.falconx.io/newsroom/bringing-institutional-lending-on-chain-exploring-falconxs-credit-vault)
  and [RockawayX on the structured credit facility](https://www.rockawayx.com/insights/rockawayx-pareto-bringing-institutional-credit-on-chain)
  — The vault that Royco subsequently tranched, described by the two parties who built it.
- [Pareto on the Royco tranches](https://x.com/paretocredit/status/2070449031617061054) —
  The confirmation that recursive tranching is happening in production rather than in
  theory.

## Historical

Worth reading to see which arguments have been made before, and which protocols in them
no longer exist.

- [The Rise of Risk-Hedging DeFi Protocols](https://quantstamp.com/blog/the-rise-of-risk-hedging-defi-protocols),
  Quantstamp Labs, June 2021 — The first wave: BarnBridge, Saffron, NAOS, Pendle, 88mph.
  Four of those five are gone or transformed, which is the point.
- [Risk Tranching in DeFi](https://medium.com/waterfall-defi/risk-tranching-in-defi-fa70c8d6b0ae),
  Waterfall DeFi — Written by a protocol that attempted a three tranche sequential
  waterfall and is now defunct.
- [Introduction to Saffron](https://medium.com/saffron-finance/introduction-to-saffron-9a46f2693612)
  — The AA, A and auto balancing S tranche design, now dormant.

## Traditional structured finance

The reference the on chain versions are approximating, and the better source for what a
complete waterfall looks like.

- [Understanding Collateralized Loan Obligations](https://www.guggenheiminvestments.com/perspectives/portfolio-strategy/understanding-collateralized-loan-obligations-clo/),
  Guggenheim — Capital structure, the reinvestment period, the waterfall, the role of the
  manager.
- [Coverage Tests](https://collateralizedloanobligations.com/mechanics/coverage-tests) —
  The formulas, indicative trigger levels, a worked example, and what happens on a
  breach. The basis of
  [`01-fundamentals`](./01-fundamentals/README.md).

## Standards

- [eips.ethereum.org](https://eips.ethereum.org/) — Every specification cited in
  [`05-token-standards`](./05-token-standards/README.md) is there in its canonical form.
- [Discussion of EIP-5095](https://ethereum-magicians.org/t/eip-5095-principal-token-standard/9259)
  — Useful as a record of why a standard stalls.
