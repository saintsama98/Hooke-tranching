# Re

> Study of a live system. Reinsurance capital, a three layer stack, and the only
> first loss layer in this repository that is funded by the operator rather than sold
> to depositors.

Re is a reinsurance protocol. Stablecoin capital is deposited on chain, allocated into
fully collateralised quota share reinsurance contracts written by a licensed reinsurer,
and the premiums earned on those contracts are paid back to depositors as yield. It is
included in this repository because the capital structure sitting behind that
arrangement is a genuine three layer waterfall, which makes it the only protocol covered
here whose stack matches the shape traditional securitisation actually uses rather than
the two layer simplification the rest of the sector has settled on.

The distinction that matters when reading Re is that its first loss layer is not on
chain and is not held by depositors. Losses are absorbed first by the equity of the
reinsurer itself, recorded at 77 million dollars as of June 2026, then by the mezzanine
layer, and only then by the senior layer. Both of the layers that depositors can buy sit
above a cushion provided by the operator, which is the opposite of the arrangement in
every other protocol in this repository, where the junior layer is sold to whoever will
take it and the protocol itself has nothing at stake. Whether that is better depends
entirely on how the reinsurer's equity is verified, and that question is answered off
chain.

The two on chain legs are reUSD and reUSDe. reUSD is the senior claim, priced for
capital preservation, and it earns a blended benchmark plus 250 basis points. reUSDe is
the mezzanine claim, priced for return, and it earns the same benchmark plus 850 basis
points in exchange for standing in front of the senior layer. The 600 basis point gap
between them is the price of subordination in this structure, quoted plainly, which is
more than most protocols in this space disclose. The mechanics follow below.

Re is materially larger than the other protocols studied here. reUSD alone showed a
total value locked of about 202 million dollars against a supply of roughly 177 million
tokens, with a current yield of 6.4 percent on the senior leg and 12.3 percent on the
mezzanine leg. For comparison, Strata sits near 78 million and Royco near 10 million.
Whatever else is true of the reinsurance thesis, the capital has arrived, and it has
arrived at the layer that produces the lower return, which is a reasonable signal about
who is buying. Figures from the [Re application](https://app.re.xyz/reusd) and are
current as of the date they were read rather than continuously.

The reason a repository about tranching should care about an insurance protocol is that
insurance is where tranching came from. A quota share treaty is a proportional sharing
of premium and loss between a cedent and a reinsurer, which is the same primitive a
waterfall generalises. Re is the case in this collection where the risk being tranched
is not a credit spread or a staking yield but an underwriting result, and the loss it
protects against is a catastrophe rather than a default. The arithmetic is the same and
the correlation structure is completely different, which is worth sitting with.

Sources: [Re documentation](https://docs.re.xyz/),
[how the protocol works](https://docs.re.xyz/protocol/how-the-re-protocol-works),
[Hacken audit](https://hacken.io/audits/re-protocol/sca-re-re-defi-aug2024/),
[DefiLlama](https://defillama.com/protocol/re).

---

## Mechanics

## The capital stack

Re runs three layers, absorbed in order. The equity of the reinsurer takes the first
loss, and the documentation puts that layer at 77 million dollars as of June 2026,
alongside retained reserves. The mezzanine layer, held on chain as reUSDe, absorbs loss
once the reinsurer's equity is exhausted. The senior layer, held on chain as reUSD, is
reached last, and the documentation states that "only severe scenarios that exhaust
lower layers of capital expose reUSD to losses."

Read against [`01-fundamentals`](../01-fundamentals/README.md)
this is an ordinary subordination waterfall with one unusual property: the layer that
absorbs first is not for sale. Depositors cannot buy the equity leg, and the protocol
operator is the party carrying it. In the terms
[`04-liquidity`](../04-liquidity/README.md#junior-demand-is-the-binding-constraint-on-the-whole-sector)
uses, Re does not have to solve the junior demand problem for its first loss layer,
because it did not try to sell it. It has moved the problem rather than removed it,
since the adequacy of that equity is now something a depositor has to take on trust from
an off chain balance sheet rather than something they can read from a contract.

## Where the yield comes from

Capital is deposited into what the protocol calls Insurance Capital Layers, which
allocate it to fully collateralised quota share reinsurance agreements backed by a
licensed reinsurer. Cover Re SPC Ltd, a Cayman Islands entity, is described as the sole
reinsurance partner. A quota share treaty is a proportional arrangement: the reinsurer
takes an agreed share of the premiums a cedent writes and pays the same share of the
claims, so the capital is exposed to the underwriting result of a book of policies
rather than to the credit of a single borrower.

This is a genuinely different risk from everything else in this repository. The credit
protocols surveyed in [`03-protocol-landscape`](../03-protocol-landscape/README.md#risk-tranching-protocols)
tranche a default risk that correlates with the crypto cycle and with rates. An
underwriting book correlates with weather, litigation and catastrophe, and it does not
care what the market is doing. That is the strongest argument for the product and it is
also the reason the usual on chain intuitions about when a junior layer gets hit are not
transferable here.

## How the two legs are priced

Both legs earn a blended benchmark plus a fixed spread, and the benchmark depends on
where the capital is actually sitting rather than on a single reference rate. Capital
deployed on chain is benchmarked to the seven day trailing average sUSDe yield. Capital
deployed off chain is benchmarked to SOFR. The senior leg adds 250 basis points to that
blend, and the mezzanine leg adds 850.

| Leg | Token | Position in the stack | Spread over the blend | Redemption |
|---|---|---|---|---|
| Senior | reUSD | Third to absorb loss | 250 basis points | Instant when liquidity allows |
| Mezzanine | reUSDe | Second to absorb loss | 850 basis points | Quarterly windows |
| Equity | Not tokenised | First to absorb loss | Underwriting result | Not applicable |

The pricing is fixed rather than adaptive, which puts Re at the opposite end of the
spectrum from the protocols in
[`09-study-strata`](../09-study-strata/README.md) and
[`08-study-pareto`](../08-study-pareto/README.md),
where the split moves continuously with coverage. A fixed spread is easier to understand
and easier to sell, and it means the compensation a mezzanine holder receives does not
respond to how much protection they are actually providing on any given day. Whether
that is a flaw depends on whether the composition of the stack moves much, and in a
structure whose first loss layer is a reinsurer's balance sheet rather than a deposit
queue, it moves a good deal less than it does elsewhere.

## Redemption, which is the real difference between the legs

The senior leg redeems instantly when on chain liquidity is available, subject to a cap
of 20 percent of daily capacity in total and 10 percent per wallet, with a redemption fee
of 0.06 percent. The mezzanine leg redeems in quarterly windows, and those windows are
set by when regulatory collateral is released and by an actuarial assessment of
available capacity, with requests filled pro rata if they exceed the surplus.

That quarterly window is doing a great deal of work and deserves more attention than the
spread does. Reinsurance capital is committed for a treaty period, and it cannot be
returned before the exposure it is collateralising has run off, whatever a depositor
would prefer. The mezzanine leg is therefore illiquid for a structural reason rather
than a policy one, which distinguishes it from the coverage gates described in
[`04-liquidity`](../04-liquidity/README.md#every-protocol-restricts-exit-and-the-instruments-differ).
Those gates tighten when a protocol decides conditions warrant it. This one is a
property of the underlying contract and does not relax when things are calm.

## The stated loss probabilities

The documentation publishes stress figures: a 0.03 percent probability of impairment for
reUSD and 0.9 percent for reUSDe under a severe scenario defined as a 135 percent
combined ratio. It also states that combined ratios have been below 100 percent across
every underwriting year since inception.

These numbers should be read the way any modelled loss distribution is read, which is as
the output of an assumption rather than as a measurement. A combined ratio above 100
percent means claims and expenses exceeded premiums, and the 135 percent scenario is a
choice about how bad a bad year is. The section on calibration in
[`01-fundamentals`](../01-fundamentals/README.md)
applies directly and with some force: the arithmetic of the waterfall is exact, the
probability attached to reaching any particular layer of it is a judgement, and the 2008
episode that discredited the previous generation of structured credit was a failure of
exactly that judgement rather than of the payment mechanism. A published track record of
combined ratios below 100 percent is real evidence and it is also a short record taken
during a period without a major catastrophe loss.

## Assessment against the common shape

Set against the five parts described in
[`03-protocol-landscape`](../03-protocol-landscape/README.md#the-protocol-landscape), Re provides an
asset pool that is not a DeFi strategy at all but a book of reinsurance treaties, which
is the most genuinely uncorrelated source in this collection. Its valuation and loss
accounting are the weakest part of the on chain story and the strongest part of the off
chain one: the value of a reinsurance position is an actuarial estimate produced by a
regulated entity and reported in, rather than a figure any contract can derive. The
waterfall is present and unusually complete, with three ordered layers instead of two.

There is no coverage test in the sense this repository uses the term. Nothing measures a
ratio each period and reroutes cash flow to repair it, which places Re alongside every
other protocol here. What it has instead is a first loss layer that is thick by on chain
standards and provided by the operator, which addresses the same worry from a different
direction. Settlement is the part where the two worlds visibly collide: a senior leg that
redeems in seconds sits on top of a mezzanine leg that redeems quarterly and a treaty
that runs for a year, and the protocol manages that mismatch with caps rather than
resolving it.

Sources: [Re documentation corpus](https://docs.re.xyz/llms-full.txt),
[how the protocol works](https://docs.re.xyz/protocol/how-the-re-protocol-works),
[reUSDe FAQ](https://docs.re.xyz/insurance-capital-layers/reusde-faq),
[smart contract addresses](https://docs.re.xyz/protocol/smart-contract-addresses).
