# The Protocol Landscape

> Reading order: after [`../02-forms-of-tranching/`](../02-forms-of-tranching/README.md).
> This document is the survey. It sets out the shape every tranched protocol shares, then
> works through who has built what.

Every tranched protocol surveyed here is assembled from the same five parts. Knowing the
parts is useful less as a taxonomy than as a reading order: the differences between these
protocols are almost entirely a matter of which parts they have, how much of each part
lives on chain, and which of them the documentation quietly declines to describe.

The first part is the **asset pool**, the source of both the cash flow and the loss.
Usually this is a strategy reached through an adapter, either an ERC-4626 vault or an
[`05-token-standards`](../05-token-standards/README.md#the-vault-and-yield-layer-beneath-a-tranche-structure), though in the case of Re
it is a book of reinsurance treaties and not a DeFi position at all. The second part is
**valuation and loss accounting**, meaning the determination of what the pool is worth
and the recognition that a loss has occurred. No standard covers this, and it carries the
trust assumption on which everything above it rests.

The third part is the **waterfall** itself: cash fills the senior claim first and the
junior claim takes the remainder, while loss reduces the junior claim first. The fourth
is **coverage tests and cure**, meaning the periodic measurement of a ratio and the
rerouting of cash flow while that ratio sits below its trigger, described in
[`01-fundamentals`](../01-fundamentals/README.md). The fifth is
**settlement**, meaning how value actually reaches a holder, whether immediately, through
an epoch, or through a queue. Parts three and four sit between the accounting and the
tokens, and where there is an epoch, the waterfall executes at its boundary.

Part four is the one that is generally missing. If you took the best available
implementation of each part from across the sector you would end up with the `IdleCDO`
engine and its dynamic yield split, the Centrifuge subordination ratio gate, ERC-7540
settlement, and the fixed excess spread allocation Goldfinch uses. Assembled, that covers
parts one, two, three and five, and still contains no mechanism that actively diverts
cash flow to repair a breached ratio. Strata comes closest by diverting junior capital to
meet the senior interest entitlement, and even it never retires senior principal, so the
structure does not delever. The protocol by protocol detail is in
[risk tranching protocols](.#risk-tranching-protocols).

The frame also makes the differences legible. Royco adds a sixth thing that is not on
this list at all, a paid provider of exit liquidity, which belongs to part five and turns
it from a constraint into a priced position. Re has the most complete version of part
three, with three ordered layers, and the weakest on chain version of part two, since the
value of a reinsurance position is an actuarial estimate reported in from outside.
Centrifuge had a strong part four in Tinlake and moved it off chain in its successor. In
each case the question worth asking is not whether the protocol is tranched but which
parts it actually implements and where each of them runs.

---

# Risk tranching protocols

Protocols that implement senior and junior subordination, meaning the legs differ in who
absorbs loss first. Each is described against the five parts above. Figures are as
reported at the dates their sources give and are not continuously updated.

## Pareto

The protocol formerly called Idle Finance, and the most complete general purpose
tranching engine built in DeFi. A yield source feeds an engine that tracks a virtual
price and divides interest between a senior class and a junior class, with the division
recomputed from the total value of both classes at every deposit and redemption rather
than fixed at launch. The junior class absorbs loss first and the senior class is reached
only once the junior class is exhausted.

The engine is now pointed at institutional private credit rather than at DeFi yield.
Pareto operates credit vaults with configurable rates, lockups, withdrawal cycles,
reserve ratios and tranches, lending to institutional borrowers underwritten off chain,
and issues USP, a synthetic dollar minted one for one against stablecoin deposits and
backed by that credit exposure. The FalconX credit vault reached a reported 144 million
dollars on 23 June 2026 at a thirty day gross yield of 8.25 percent before fees, and
includes equity tranches absorbing first loss. Reported protocol value locked sits between
151 and 227 million dollars depending on the source, both figures from March 2026.

Provides valuation, a waterfall priced continuously by coverage, and settlement. There is
no coverage test and no diversion of cash flow, so loss absorption is passive. Full study
in [`../08-study-pareto/`](../08-study-pareto/README.md).

## BarnBridge SMART Yield (discontinued)

The senior claim was issued as sBONDs (ERC-721; fixed, dated, and guaranteed) and the junior claim as jTokens (ERC-20; residual, first loss), with a moving average yield oracle setting the senior rate. The structure is notable for representing seniority as a nonfungible, dated instrument and juniority as a fungible, perpetual one. The protocol was discontinued in July 2023 and was the subject of a [Securities and Exchange Commission settlement](https://www.sec.gov/newsroom/press-releases/2023-258) in December 2023 — a regulatory outcome rather than a technical failure. Source: [SPEC.md](https://github.com/BarnBridge/BarnBridge-SmartYieldBonds/blob/master/SPEC.md).

## Centrifuge

Pools, tranche tokens and investment logic are native to EVM chains under the current
architecture, with the Centrifuge Chain retaining governance, asset documentation and
protocol metadata, and deployments spanning Ethereum, Base, Arbitrum, Avalanche, BNB
Chain and Plume. The documented tranche model is general rather than fixed at two: a
senior tranche at the top, a junior tranche at the bottom, and any number of mezzanine
tranches between them, which makes Centrifuge one of only two protocols in this survey
capable of expressing more than one subordination boundary.

The commercially significant product is not a native tranche at all. JAAA is a tokenised
feeder into the flagship AAA collateralised loan obligation strategy of Janus Henderson,
so the underlying is a portfolio of the senior most tranches of real collateralised loan
obligations with floating coupons resetting off SOFR. It came on chain in June 2025,
reached roughly 687 to 690 million dollars of assets under management across eight
networks, and entered Kraken Custody in June 2026. Separately, deRWA wrappers enforce
compliance at the pool level and leave the resulting token transferable, so these
positions trade on decentralised exchanges including Aerodrome, Balancer and Curve.

Provides an asset pool, valuation, a waterfall, and settlement, with the coverage and
pricing arithmetic held off chain. Centrifuge is by a wide margin the largest entity in
this survey and it got there by distributing a claim on a traditional waterfall rather
than by building one. Full study in
[`../07-study-centrifuge/`](../07-study-centrifuge/README.md).

## Goldfinch

The Senior Pool issues FIDU (ERC-20); Backers occupy the junior, first loss position and hold PoolTokens (ERC-721). A leverage model sizes the senior position as a multiple of the junior, and a fixed 20% of senior nominal interest is reallocated to the junior position. This reallocation is the closest on chain analogue to traditional excess spread capture. Provides valuation and a waterfall; there is no coverage test or cure. Source: [Backers documentation](https://docs.goldfinch.finance/goldfinch/goldfinch-v1/protocol-mechanics/backers).

## Maple

Lender positions are ERC-4626 shares. The loss order is writedown, then borrower collateral, then liquidation of the cover (junior) position ahead of lenders, then recoveries. Cover is thin — on the order of 0–3% and often unused — which demonstrates that on chain first loss capital is structurally smaller than the 8–12% equity typical of traditional structures. This is a constraint for the specification rather than a mechanic. Source: [defaults and impairments](https://docs.maple.finance/maple-for-lenders/defaults-and-impairments).

## Strata

A strategy feeds a CDO contract that measures gain as the change in total value since the last interaction, then applies the Dynamic Yield Split across three balances: Srt, the senior tranche, Jrt, the junior first loss tranche, both ERC-4626 with LayerZero OFT transferability, and a reserve.

A generalised wrapper applying one engine to seven heterogeneous strategies. The senior rate is `MAX(benchmark floor, base APR × (1 − x − y·r^k))` where `r` is the senior share of total value; the junior claim is the residual, and the published junior formula is the conservation identity rearranged rather than an independent rule. Accounting is discrete and lazy — advanced only by a protocol action or a rate feed update — and the invariant `jrtTVL + srtTVL + reserveTVL == totalTVL` is restored on each advance. Loss and income share a single unconditional debit of the junior balance, so a yield shortfall against the floor impairs the junior tranche exactly as a default would.

Exit friction is tiered across up to three coverage bands, each applying any combination of share lock, asset lock, and fee; below a minimum coverage threshold (105% for the Ethena market) senior issuance and junior redemption are suspended together. This is a finer instrument than Centrifuge's binary subordination gate, and Case B constitutes a genuine interest coverage cure by diverting junior capital to the senior entitlement. No mechanism retires senior principal to restore the ratio, so the structure still does not delever. Provides valuation, a waterfall, a partial coverage response, and settlement, the last through purpose built cooldown contracts rather than ERC-7540. Sources: [Dynamic Yield Split](https://docs.strata.markets/protocol-mechanism/dynamic-yield-split), [Protocol Overview](https://docs.strata.markets/technical-documentation/protocol-overview). A full economics level review is in [`09-study-strata`](../09-study-strata/README.md).

## Royco

Three legs rather than two: a senior tranche, a junior tranche, and a senior liquidity
provider that runs an automated market maker pool of senior shares against the quote
asset of the market. Yield flows out of the senior leg to both of the others, paying
separately for coverage and for an exit. Every senior position has a minimum amount of
junior capital in front of it and is not impaired until that capital is exhausted, with
the floor set per market at roughly 3 to 20 percent.

The division of yield is set by one of three curves, the more advanced two of which move
with utilisation, in the same family as the Idle and Strata splits. After a loss a market
can enter a fixed term state that restricts withdrawals, so a drawdown that reverses
inside the window is never realised.

Provides an asset pool, valuation, a waterfall, a partial coverage response and
settlement. There is no diversion of cash flow to repair a breached ratio, so the
structure does not delever. What it adds instead is a paid counterparty on the exit,
which no other protocol here has. Sources:
[Royco overview](https://docs.royco.org/royco-overview),
[Royco Dawn repository](https://github.com/roycoprotocol/royco-dawn/). Full study in
[`10-study-royco`](../10-study-royco/README.md).

## Re

Re tranches an underwriting result rather than a credit spread. Stablecoin capital is
deposited on chain and allocated into fully collateralised quota share reinsurance
treaties written by a licensed reinsurer, and the premiums earned on those treaties are
what the depositors are paid. The capital stack has three ordered layers rather than the
usual two: the equity of the reinsurer absorbs the first loss, recorded at 77 million
dollars as of June 2026, then reUSDe as the mezzanine layer, then reUSD as the senior
layer. Both tokenised legs therefore sit above a cushion the operator funds itself,
which is the reverse of the arrangement everywhere else in this survey.

Pricing is fixed rather than adaptive. Both legs earn a blended benchmark, the seven day
trailing sUSDe yield for capital held on chain and SOFR for capital held off chain, and
the senior leg adds 250 basis points to it while the mezzanine leg adds 850. The
difference between the legs shows up more sharply in redemption than in yield: senior
redeems instantly when liquidity allows, capped at 20 percent of daily capacity, while
mezzanine redeems only in quarterly windows set by when regulatory collateral is
released. That illiquidity is a property of the underlying treaty rather than a policy
lever, so unlike a coverage gate it does not relax when conditions are calm.

Provides an asset pool that is genuinely uncorrelated with the rest of DeFi, a
valuation that is an actuarial estimate reported in from off chain, a three layer
waterfall, and settlement. There is no coverage test in the sense used here. Re is also
the largest protocol in this survey by a wide margin, with about 202 million dollars in
the senior leg alone. Sources:
[Re documentation](https://docs.re.xyz/),
[how the protocol works](https://docs.re.xyz/protocol/how-the-re-protocol-works). Full
study in [`11-study-re`](../11-study-re/README.md).

## Saffron and Waterfall DeFi (dormant / defunct)

- **Saffron.** AA, A, and an auto balancing S tranche, on 14-day epochs, with the junior position staking SFI as insurance at approximately ten times leverage. Now dormant. Source: [introduction](https://medium.com/saffron-finance/introduction-to-saffron-9a46f2693612).
- **Waterfall DeFi.** A three tranche structure (Senior, Mezzanine, Junior) with cash flow distributed top down and loss absorbed bottom up, on 7-day epochs. Now defunct (CertiK score 2.1/10). It demonstrates that a three or more tranche sequential waterfall is implementable but was not sustained. Source: [what is tranching](https://waterfall-defi.gitbook.io/waterfall-defi/introduction/what-is-tranching).

## Assessment

The surveyed protocols provide nondecreasing or checkpointed net asset value accounting, strict junior first loss exhaustion, dynamic coverage pricing, and subordination gating (Centrifuge). They do not provide active coverage test cures, excess spread capture beyond a fixed allocation, sustained three or more tranche waterfalls, or adequately thick first loss capital. See [`01-fundamentals`](../01-fundamentals/README.md).

---

# Cash flow tranching protocols

Protocols that split a yield asset into a principal component and a yield component, rather than into senior and junior risk layers. The relevant token standards are described in [`05-token-standards`](../05-token-standards/README.md#the-vault-and-yield-layer-beneath-a-tranche-structure), and the difference between the two kinds of split is set out in [`02-forms-of-tranching`](../02-forms-of-tranching/README.md).

## Pendle

A yield asset is wrapped as SY, the EIP-5115 wrapper that is a superset of ERC-4626, and the wrapper is then split into PT, the principal token that redeems one for one at maturity, and YT, which carries the variable yield until maturity.

The principal token (PT) and yield token (YT) are both ERC-20. SY normalises the underlying yield source, and the split operates only against the SY `exchangeRate()`. The internal pyIndex is designed not to decrease, so yield volatility is absorbed by the YT first and the PT is ring fenced; the PT therefore occupies a position analogous to the senior tranche and the YT a position analogous to a first loss claim on yield. The structure has a fixed maturity, and the PT/YT oracle takes the maximum of the SY and PY indices to resist downward manipulation. Pendle's principal tokens implement ERC-4626 rather than EIP-5095. Sources: [Pendle documentation](https://docs.pendle.finance/pendle-v2/Introduction), [Minting](https://docs.pendle.finance/pendle-v2/ProtocolMechanics/YieldTokenization/Minting), [mixbytes analysis](https://mixbytes.io/blog/yield-tokenization-protocols-how-they-re-made-pendle).

## Spectra

The same principal and yield split applied to interest bearing tokens: the principal token provides a fixed return at maturity and the yield token supports speculation on or hedging of future yield. Source: [spectra-core](https://github.com/perspectivefi/spectra-core).

## Two things worth taking from this

The nondecreasing index is a valuation technique that applies to risk tranching as much as to cash flow tranching: it makes the protected leg insensitive to short term volatility in the source.

Normalising first and splitting second means the split engine only ever reasons about one figure, the exchange rate of the wrapper, whatever the underlying yield source is. That is why one engine can sit over many strategies. Strata reaches the same result on the risk axis by a different route, see [`09-study-strata`](../09-study-strata/README.md).

---

# Real world asset securitisation entrants

Recent entrants, from 2025 and 2026, that come closest in intent to a full waterfall. They treat loss recognition as a first class element, which most DeFi designs do not.

## Untangled Finance

A credit pool and a Credit Oracle sit behind two tokens: SOT, the Senior Obligation Token, and JOT, the Junior Obligation Token.

Untangled uses explicit securitisation terminology and a Credit Oracle, which represents a direct attempt to provide on chain loss recognition. It is the most traditionally structured of the on chain models surveyed. The token standard is not yet verified and is likely ERC-20. Source: [documentation](https://docs.untangled.finance/docs/credio/Onchain-Private-Credit/Intro-Untangled-Pool/).

## Galaxy CLO 2025-1

A tokenised collateralised loan obligation (senior tranche at SOFR plus 570 basis points; initial 75M, expandable to 200M; on Avalanche). The waterfall operates off chain within the traditional structure, and the token represents the claim. This is useful as a reference for the form of a real collateralised loan obligation tranche rather than as on chain mechanics. Source: [PRNewswire](https://www.prnewswire.com/news-releases/galaxy-announces-initial-closing-of-debut-tokenized-clo-at-75-million-302662045.html).

## Tradable

Tokenised private credit on ZKsync, following the same wrapper pattern of an off chain waterfall and an on chain claim. Likely permissioned (ERC-3643 or ERC-1400 family); not verified. Source: [Chainlink analysis](https://chain.link/article/onchain-private-lending).

## Assessment

These wrappers demonstrate demand for tokenised tranches but retain the waterfall off chain; they tokenise the output of a traditional structure rather than specifying the engine. Untangled is the exception in bringing loss recognition on chain. The token standards used by Untangled, Galaxy, and Tradable are not confirmed; see [`05-token-standards`](../05-token-standards/README.md#adoption-in-practice-diverges-from-what-the-standards-intended).
