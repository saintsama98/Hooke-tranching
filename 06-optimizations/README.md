# Optimizations

> Reading order: after the landscape and the liquidity sections, and before the protocol
> studies. This is the frontier material. Everything here sits alongside a tranche and
> changes its economics without changing the ordering itself.

An optimization, as the term is used here, is a construction that takes a tranched
position and makes it cheaper, safer, more liquid or more capital efficient without
altering the priority of payments underneath. Some are being run at scale today. Some are
proposals with live prototypes and unresolved objections. One is a term in circulation
with no definition behind it, and it is recorded as such.

The reason this section is the core of the repository rather than an appendix is that the
tranching primitive itself is close to solved and largely unremarkable. Ordering claims on
a pool is arithmetic, and the protocols in
[`../03-protocol-landscape/`](../03-protocol-landscape/README.md) implement it
competently. Everything that is actually unresolved is at the edges: where the first loss
capital comes from, where the valuation comes from, how anybody exits, and whether the
same exposure can be manufactured a different way for less. Those are the questions worth
working on, and each of the sections below is an attempt at one of them.

---

## A tranche is a spread on the loss distribution, which makes options and tranching the same object

This identity is the theoretical spine of everything else in this document and it is
worth stating carefully, because it is standard in structured credit and almost never
mentioned in DeFi.

Take a tranche with an attachment point `a` and a detachment point `d`, and let `L` be
cumulative loss on the pool. The tranche pays in full while `L` is below `a`, absorbs loss
linearly between `a` and `d`, and is worth nothing once `L` reaches `d`. That payoff is
exactly the difference of two option payoffs on `L`:

```
tranche payoff = (d - L)+ - (a - L)+
```

which is a spread. The standard market consequence is that any tranche can be replicated
from two base tranches, so a 3 to 7 percent tranche is a long position in the 7 percent
equity tranche and a short position in the 3 percent equity tranche. Tranche traders have
priced these as option spreads for two decades.

The consequence for this repository is direct and, as far as the surveyed protocols go,
unexploited. There are two independent routes to manufacturing a protected claim on a
risky pool. The first is capital subordination: somebody else's money absorbs loss ahead
of you, which is what every protocol in
[`../03-protocol-landscape/`](../03-protocol-landscape/README.md) does. The second is
payoff decomposition: you give up part of the upside and the payoff formula itself
guarantees your floor, with no second party required to post first loss capital at all.

Both produce a senior claim. They fail in completely different ways. A subordinated senior
claim fails when losses exceed the cushion, and its weak point is that the cushion has to
be bought from someone, which is the binding constraint identified in
[`../04-liquidity/`](../04-liquidity/README.md). A payoff constructed senior claim cannot
fail that way because nothing was promised beyond what the formula pays, and its weak
points are the cost of the optionality and the depth of the market it has to be rolled in.

The rest of this section is what happens when people actually try to build the second one.

---

## Payoff tranching removes the need for a junior tranche entirely

The clearest articulation of the second route is a 2026 proposal by Vitalik Buterin on
building index tracking assets from options rather than debt. It is not framed as
tranching and it is tranching, and the reframing is the most useful single idea available
to this field right now.

The construction is minimal. Fix a ticker, a strike `S` and a maturity `M`. One ETH mints
a pair of complementary tokens, `P` and `N`, and the pair redeems for one ETH at any time.
At maturity, with the oracle resolving to price `x`:

```
P receives  min(1, S/x)  ETH
N receives  max(0, 1 - S/x)  ETH
```

The two payoffs sum to one ETH under every possible value of `x`. That is a conservation
identity, and it is precisely the identity that Strata enforces on its own split, where
senior value plus junior value plus reserve must equal total value or the settlement
reverts. Same invariant, different construction: Strata conserves capital across two
claims funded by two sets of depositors, while this conserves a payoff across two claims
minted from one deposit.

What follows from that is the interesting part. Because the two legs always sum to the
collateral, there is no scenario in which the structure owes more than it holds, and
therefore no liquidation. Because there is no liquidation, there is no need for a
continuous price feed. The oracle is consulted once, at maturity, which means it can be
slow, expensive and adversarial rather than fast and cheap. Buterin's own framing of that
advantage is worth quoting because it generalises well past this application: put a
prediction market in front of a safe but expensive oracle, and only fall back to the
expensive oracle when there is serious disagreement.

Set against the findings in the root [`README.md`](../README.md), this attacks two of the
six at once. It needs no first loss capital, so the junior demand problem does not arise.
And it needs no continuous valuation, so the trust assumption that every surveyed protocol
carries is reduced to a single reading at a known moment.

### The objections are serious and mostly about carry

The thread that followed is more valuable than the proposal, because the practitioners in
it attacked the economics rather than the concept.

The tracking objection came from Norswap, who worked through whether `P` actually tracks
the intended index and concluded that it is directionally short, that only far out of the
money options provide meaningful leverage, and that nothing in the construction tracks a
cross rate close to linearly in the long direction. The underlying point is structural:
the system redistributes a fixed pool of ETH collateral between two claimants and cannot
manufacture long exposure to an asset that is not in the pool.

The carry objection came from Equivrel and is the sharpest. A one year put with the
underlying at 2500 and a strike at 1500 bleeds roughly 0.40 to 0.50 percent per day in
value. A protected claim assembled from long dated options is paying that theta whether or
not anything goes wrong, and at that rate holding to maturity is not economically viable,
which undercuts the slow oracle benefit because the position has to be actively managed
anyway.

Czar102 answered both with a refinement that materially improves the design. Roll short
dated options, three to four days, rather than holding long dated ones, and the carry
falls to roughly 0.10 percent in a worse than average scenario instead of the one to four
percent per year the original proposal accepted. There is no dynamic rebalancing in that
version, only predictable rolling. The same reply proposes settling with covered options
in which both participants accept either asset at expiry, which removes the oracle
completely when the asset exists on chain, and argues that perpetual futures are unsound
by comparison because they create a circular dependency in which traders are forced to pay
whatever funding rate an oracle says other traders have decided.

Buterin's own stated worst case is the one to watch. It is very easy to lose two percent
per year or more to repeated rounds of rebalancing slippage, and that is the largest single
risk by which the whole scheme becomes uncompetitive. A protected claim that costs two
percent a year in slippage is competing against a subordinated senior tranche that costs
whatever premium the junior tranche demands, which in the protocols surveyed here runs
from a few hundred basis points upward. The comparison is genuinely close, and it is
decided by market microstructure rather than by design elegance.

### Four variants are already live and they trade off differently

The proposal is not hypothetical. Four distinct implementations or designs appeared in the
same discussion, and the differences between them are a useful map of the design space.

The physical settlement variant is the most radical and removes the oracle from settlement
altogether. Rather than resolving a price, `N` exercises by paying the strike asset in and
taking the collateral out, so settlement is expressed as asset movement rather than price
resolution. It is deployed on Base as split.markets for a WETH and USDC pair, and it
carries formal verification work across several layers, including Lean 4 proofs and
symbolic execution, aimed at proving that settlement output is independent of any external
price data and that solvency holds across all reachable states. For a field whose central
weakness is the valuation input, a structure that provably does not read a price is a
significant object.

The perpetual variant, from Xatarrer, replaces expiry with a saturation price. Below
saturation the position sits in a convex zone with constant leverage, and above it in a
path dependent saturation zone, with the payoff following a power law in the ratio of
price to saturation price. It needs no expiry management and no funding fees, and it pays
for that with a continuous time weighted oracle rather than a single reading, and with
path dependence. It is reported live with public dashboards.

The barrier variant, from The Risk Protocol, keeps the complementary claim structure and
adds path dependent barriers that end an epoch early, with prices read at the start and
end of each epoch. Its own author raises the right objection against it, which is whether
epoch boundary oracle dependence is meaningfully different from liquidation oracle
dependence.

The comparison that matters is oracle frequency against management burden. Maturity only,
with rolling. Continuous time weighted, with no rolling. Barrier triggered, with epochs.
Or none at all, with physical exercise. Nobody has yet demonstrated that any of these
beats a subordinated tranche on total cost, and that is the open question.

---

## Principal protection can be assembled from a bond and a call, and the bond already exists on chain

The traditional market solved payoff based protection long before anyone tranched anything
on chain, and it did so with an assembly so simple it is almost disappointing. Buy a zero
coupon bond that matures at par, spend the difference between the discounted price and par
on call options, and the result is a note that cannot lose principal at maturity while
retaining participation in the upside. Calamos runs a tiered version of exactly this on
Bitcoin, with one hundred percent, ninety percent and eighty percent protection levels
launched in October 2025, and BlackRock's Bitcoin premium income product writes covered
calls on twenty five to thirty five percent of the portfolio to target a fifteen to twenty
five percent annualised distribution.

The on chain observation is that the hard component of that assembly is already built and
liquid. A principal token from a yield splitting protocol is a zero coupon bond: it
redeems one for one for the underlying at a stated maturity and trades at a discount until
then. Anyone holding one has already bought the bond leg. The protected note is that
position plus a call, and the call is funded by the discount.

That is a route to a senior claim which requires no junior tranche, no coverage ratio, no
subordination gate and no first loss capital. It requires an options market with enough
depth at the relevant strike and maturity, which on chain is the binding constraint rather
than the concept. It also carries a different failure mode worth stating plainly: the
protection is at maturity and not before, so the position can be well underwater on a mark
to market basis in the interim, which matters to anybody who might need to exit early or
who is posting the note as collateral somewhere else.

---

## Looping a senior tranche puts back exactly the risk the tranche removed

The most widely practised optimization in this area is also the one whose logic most
deserves scrutiny.

A senior tranche carries contractual downside protection, which makes it better collateral
than the raw yield bearing asset beneath it. A lending market can reasonably lend against
it at a higher ratio. So a curator tranches a strategy, keeps the senior leg, borrows
against it at a favourable rate, and redeploys, with the protection nominally still in
place underneath the leverage. Dialectic describes doing precisely this in work published
with Royco in July 2026.

The industrial scale version of the same idea uses principal tokens rather than senior
tranches. Supply the principal token as collateral, borrow a stablecoin against it, swap
back into more of the same principal token, and repeat. Pendle offers this as a single
click operation against Morpho markets, and in August 2026 advertised an estimated maximum
looping return of 27.13 percent on one principal token maturing that October and 15.96
percent on another maturing in August. The reason lenders will accept the collateral at a
tight ratio is genuinely sound: a principal token has a known redemption value on a known
date, so the collateral risk is far more tractable than it is for a floating asset.

The critical observation, and the reason this belongs in a document about tranching rather
than in a lending survey, is what the composition does to the risk profile. Tranching
transfers a defined slice of risk to a counterparty who agreed to take it. Looping does not
transfer anything to anybody; it enlarges the same exposure and manages the consequences
with a liquidation. Putting the second on top of the first therefore reintroduces, in the
form of a liquidation level, precisely the discontinuous failure that the tranche was
bought to avoid. The senior claim is protected against loss on the underlying and is not
protected at all against being liquidated on the way there.

Whether the combination is prudent depends on numbers specific to the deployment: the
ratio, the borrow cost against the tranche premium, and whether the liquidation level sits
inside or outside the range the protection covers. That last one is the question to ask,
and it is rarely answered in public. There is also a maturity trap in the principal token
version, since the collateral converges to par on a known date while the borrowing does
not, so the position's risk profile changes shape as maturity approaches.

---

## Recursive tranching is live, and every layer inherits what is underneath it

A tranche token is an ordinary token, so it can be tranched again. This was theory in this
repository until recently and is now production.

Royco applied its senior and junior split on top of the Pareto FalconX credit vault, which
already contains equity tranches absorbing first loss on a book of fixed rate loans to
trading firms and hedge funds. Pareto's own account describes the Royco tranches as making
the vault more appealing to liquidity providers wanting flexibility across the risk and
yield spectrum. The reported size of the outer senior leg was about 2.7 million dollars
against a vault that reached 144 million.

Two other composition patterns are in use. Tranche tokens are paired on markets, which
Royco takes furthest by paying a dedicated party to keep a pool for the senior leg. And
tranche tokens are split again on a yield splitting protocol, which produces a fixed rate
on a protected claim and stacks two different kinds of protection with different
maturities on one position.

The history here is not encouraging and should be stated rather than glossed. The
traditional market built exactly this, called it a collateralised debt obligation squared,
and the results in 2008 were poor. That is not an argument that it cannot be done well. It
is an argument for examining each layer separately, which reduces to four questions.

Whose valuation sits at the bottom, since every layer inherits the trust assumption of the
layer below and no amount of stacking dilutes it. How thick is each cushion, since a senior
claim on a junior claim protected by twelve percent is a different instrument from one
protected by three, and both will quote similar yields. Do the exits agree, since a liquid
outer claim wrapped around an inner claim that pauses withdrawals during a loss is
protected on paper and trapped in practice at the exact moment the protection was supposed
to matter. And are the layers correlated, since two tranches over two strategies resting
on the same underlying source are one exposure wearing two hats.

On the live example, the answers are: off chain underwriting of institutional credit, thin
at the outer layer, no, and yes.

---

## Making the curator post the first loss capital solves demand and creates contagion

The binding constraint across this whole sector is that somebody has to want the junior
tranche. A December 2025 paper by Zbandut and Goldstein on institutionalising risk
curation in decentralised credit proposes an answer that is being adopted in practice:
have the curator who assesses the risk commit the first loss capital.

The argument is an alignment one rather than a capital one. A curator who evaluates credit
and takes no exposure to being wrong is selling an opinion. A curator who must stand first
in line behind their own assessment is selling a position, and the market can price their
credibility directly by observing what they are willing to stand behind. This replaces
uniform collateral requirements with differentiated assessment, and it converts the junior
tranche from something that must be sold to strangers into something the operator supplies
as the cost of doing business.

Re reaches the same destination by a different route, since its first loss layer is the
equity of the reinsurer and is not offered to depositors at all. Between the two, a real
pattern is visible: the sector is quietly moving the first loss layer from the market onto
the operator's own balance sheet.

The paper is honest about the cost, and it is the right cost to worry about. Curator
networks create contagion channels, because a curator failure propagates across every
protocol that curator touches, and the same reputational mechanism that disciplines them in
normal conditions synchronises their behaviour in stressed ones. A sector in which the
first loss capital is concentrated in a small number of curators has replaced a demand
problem with a correlation problem, which is a worse problem and a less visible one.

---

## Slashing risk is now being tranched, which takes the primitive outside credit entirely

Everything else in this repository tranches a credit spread, a yield, or an underwriting
result. Symbiotic, with Re², has proposed slashing insurance vaults that tranche the risk
of validator slashing, which is a different kind of risk with a different shape.

The structure is conventional and the application is not. Delegators pool capital and
self insure, junior capital absorbs the first losses and earns the highest coupon,
mezzanine capital covers mid range slashing events, and senior capital is untouched until
extreme events. Premiums are set from expected excess loss, so a tranche expected to lose
more than its proportional share pays a correspondingly higher premium, which is a more
principled pricing basis than any of the credit protocols in this repository use.

The reason this deserves attention is that slashing risk has properties credit risk does
not. It is technical rather than economic, so it correlates with client bugs and operator
mistakes rather than with the market. It is bounded and quantifiable per event in a way a
credit loss is not. And crucially the loss event is observable on chain and does not
require anybody's valuation, which means the failure mode that dominates this entire
repository, an unverifiable net asset value, simply does not exist here. If the argument
in [`../01-fundamentals/`](../01-fundamentals/README.md) is right that valuation is the
weak point of tranching, then risks that settle observably on chain are the natural place
for tranching to work properly.

The stated extensions are the interesting part of the roadmap and each is a research
problem in itself. Undercollateralised tranches raise capital efficiency and reintroduce
the possibility of a shortfall. Reinsurance vaults pool tail risk across several insurance
vaults, which is a second order structure and carries the correlation question with it.
Dynamic pricing through bonding curves or auctions is an attempt to discover the premium
rather than declare it, which is something no protocol in this repository currently does.

---

## Correlation is the input nobody on chain can price, and the reason is uncomfortable

Traditional tranche markets do not merely tolerate correlation, they trade it. A bespoke
tranche is a customised slice of portfolio credit risk built to a chosen sector, maturity
and pair of attachment and detachment points, and what a buyer of a mezzanine tranche is
really expressing is a view on how defaults cluster rather than on how many there are. The
market reached roughly 169 billion dollars of bespoke tranche notional by July 2005, which
against typical tranche sizes implies underlying deal notional in the trillions, and it
froze in 2008 when rating arbitrage disappeared and nobody would hold illiquid complexity.

No on chain tranche market prices correlation, and the reason is not technical
sophistication. It is that the pools are not diversified enough for correlation to be a
meaningful parameter. A tranche over a single strategy, a single stablecoin basis trade or
a single lending vault has one underlying risk and therefore no correlation structure to
estimate. The input that destroyed the previous generation of structured credit is absent
here because the products are too concentrated to have one.

That cuts two ways and both are worth holding. It means on chain tranches are safer than
their 2008 predecessors in one specific respect, since they cannot be mispriced through a
wrong correlation assumption when there is nothing to correlate. It also means they are
not delivering the main benefit tranching offers, which is that a diversified pool has a
loss distribution whose tail can be sold separately from its body. A senior tranche over
one strategy is a claim that pays until the strategy breaks, at which point the cushion is
consumed in one event rather than absorbed gradually. Concentration converts a
subordination structure into a binary one, and the coverage ratio flatters it right up
until it does not.

The unbuilt product follows directly. A tranche over a genuinely diversified basket of on
chain credit, with an attachment point set from an estimated joint loss distribution rather
than from a governance parameter, does not exist. It is the single largest gap between what
this sector calls tranching and what tranching actually is.

---

## A slow oracle is sufficient when the structure only needs a value at the boundary

This section is a synthesis rather than a report, and it is flagged as such because no
protocol currently does it.

The central finding of this repository is that valuation is the trust assumption. A
tranched structure computes a coverage ratio and allocates loss against a net asset value,
and that value is variously an oracle, an actuarial estimate reported in from a regulated
entity, or a figure entered by an administrator. Every guarantee above it is conditional on
it, and the arithmetic can be formally verified without improving the situation at all.

The oracle argument from the options thread applies to this directly and does not appear to
have been made. A tranched pool that settles in epochs does not need a continuous
valuation. It needs one value, at the epoch boundary, at a moment known well in advance to
everybody. That is the easiest possible oracle problem: there is no latency requirement, no
manipulation window measured in blocks, and the number is worth disputing because a
disagreement can be resolved before settlement rather than after a liquidation has already
happened. It is exactly the situation the prediction market in front of an expensive oracle
technique was designed for, and the expensive fallback only has to run when parties
disagree.

The obstacle is not technical. It is that most tranched protocols have moved toward
continuous settlement because depositors prefer it, and continuous settlement demands a
continuous value. The protocols in this repository that retain an epoch boundary have the
option available and are not using it. There is a real design here for somebody: an
epoch settled tranche whose net asset value at each boundary is established by a
prediction market with an escalation path, rather than asserted by whoever holds the
administrative key.

---

## Some terms in circulation have no source behind them

**Beta layering. Status: no primary source found.** Searched across protocol
documentation, public repositories, structured finance literature and general research.
Nothing connects the phrase to any protocol in this repository, to tranching, or to a
definable mechanism.

The closest legitimate idea is ordinary portfolio construction: mixing assets of differing
sensitivity to the broader market so that part of a book moves with the market and part
does not. That is an allocation policy. It says nothing about payment priority, loss
ordering, attachment points or coverage, and it is not a tranching mechanism.

There is, however, a rigorous idea that the phrase might be reaching for, and it is worth
offering as a replacement. Because a tranche is a spread on the loss distribution, a
capital structure is a set of stacked, non overlapping exposures to different regions of
that distribution, and each region has different sensitivity to the underlying. Layering
exposures to different parts of one loss distribution is a real and precise thing to do.
If somebody uses the phrase about a specific product, the useful question is whether they
mean that, whether they mean recursive tranching, or whether they mean an allocation
policy, because those are three different activities.

An entry belongs in this list when a term was searched for in good faith and no primary
source was found. The point is to record the negative result, because a negative result
nobody writes down gets researched again by the next reader.

---

## What is genuinely unbuilt

Collecting the gaps identified above, in rough order of how tractable they look.

An epoch settled tranche with a prediction market oracle at the boundary. The pieces all
exist, no protocol has assembled them, and it addresses the sector's central weakness
directly.

A protected claim assembled from a principal token and a call, priced against the
subordinated equivalent. The bond leg is liquid, the comparison is arithmetic, and nobody
appears to have published it.

A tranche over a genuinely diversified basket of on chain credit with an attachment point
derived from a joint loss distribution. This is the product the sector says it is building
and is not.

A cure mechanism, of any kind, on chain. No protocol surveyed retires senior principal to
restore a breached ratio. Every one of them either prevents deterioration or postpones
recognition of it. The first design that actually delevers will be the first complete
waterfall on chain.

A published comparison of total cost between payoff constructed and subordination
constructed senior claims. The options route is dismissed on carry and the subordination
route is dismissed on junior demand, and as far as this research found, nobody has put the
two sets of numbers side by side.

---

## Sources

The options construction and the discussion around it:
[Building index tracking assets on top of options instead of debt](https://ethresear.ch/t/building-index-tracking-assets-on-top-of-options-instead-of-debt/25036),
Ethereum Research, 2026, including the replies from Norswap, Equivrel, Czar102, Xatarrer,
The Risk Protocol and mmchougule, and the physical settlement implementation at
split.markets on Base. Contemporary reporting:
[The Defiant](https://thedefiant.io/news/defi/vitalik-buterin-options-based-defi-replace-liquidation-driven-debt),
[Unchained](https://unchainedcrypto.com/vitalik-buterin-proposes-options-based-defi-to-end-forced-liquidations-and-the-real-time-oracle-problem/).

The tranche as a spread on the loss distribution: standard structured credit result, see
[CDO tranche sensitivities in the Gaussian copula model](https://www.math.lsu.edu/~sengupta/papers/MengSenguptaTranche08.pdf)
and [Forwards and European options on CDO tranches](https://www.researchgate.net/publication/228745093_Forwards_and_European_options_on_CDO_tranches).

Principal protection assembly and the traditional products:
[Keyrock on onchain structured product strategies](https://keyrock.com/knowledge-hub/onchain-structured-product-strategies-a-guide/),
[Spark on Bitcoin structured products](https://www.spark.money/tools/bitcoin-structured-product-comparison).

Looping: [Dialectic, Risk as a Product](https://dialecticgroup.substack.com/p/risk-as-a-product),
July 2026, written with Royco;
[Pendle principal token looping on Morpho](https://cryptobriefing.com/pendle-finance-pt-usdai-pt-susdd-morpho-27-apy/),
August 2026; [Crypto Briefing on tranching against looping](https://cryptobriefing.com/onchain-tranching-structured-finance-defi/),
reporting analysis by Silvio Busonero of Blockworks.

Recursive tranching in production:
[Pareto on the Royco tranches](https://x.com/paretocredit/status/2070449031617061054),
[FalconX credit vault](https://www.falconx.io/newsroom/bringing-institutional-lending-on-chain-exploring-falconxs-credit-vault).

Curator first loss capital: Zbandut and Goldstein,
[Institutionalizing risk curation in decentralized credit](https://arxiv.org/pdf/2512.11976),
arXiv 2512.11976, December 2025.

Slashing insurance vaults: [Symbiotic](https://resources.symbiotic.fi/),
with [secondary description](https://crypto-economy.com/symbiotic-introduces-slashing-insurance-vaults-to-mitigate-restaking-risk/).

Bespoke tranches and correlation trading:
[Single tranche CDO](https://en.wikipedia.org/wiki/Single-tranche_CDO),
[Consistent valuation of bespoke CDO tranches](https://arxiv.org/pdf/1004.1758),
[Correlation trading of synthetic CDO tranches](https://www.cambridge.org/core/books/abs/synthetic-cdos/correlation-trading-of-synthetic-cdo-tranches/2B2C53D42CC79B8598A07FCECBB467B9).
