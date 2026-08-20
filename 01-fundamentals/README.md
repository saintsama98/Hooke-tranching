# Fundamentals of Tranching

> Reading order: this document first. It assumes no prior knowledge of structured
> finance and establishes the vocabulary the rest of the repository uses.

## Tranching orders claims on one pool rather than dividing the pool itself

A pool of assets produces income and occasionally produces loss. The investors who might
fund that pool do not all want the same exposure to it. Some will accept a materially
lower return in exchange for high confidence of repayment. Others will accept a real
probability of losing capital in exchange for the possibility of a much larger return.
Tranching serves both from a single pool by arranging claims on it into a defined order,
so that income fills the first claim before reaching the second, and loss reduces the
last claim before reaching the one above it.

The distinction that matters, and that is routinely lost in casual descriptions, is that
nothing about the assets changes. The pool is not divided, split, segregated or
ring fenced. Every claim is a claim on the same undivided pool of assets. What differs is
the sequence in which the holders of those claims are paid and impaired. A tranche is
therefore a position in a queue, not a share of a thing, and every property of the
instrument follows from where in the queue it sits.

The conventional two layer arrangement over a hundred million pool funds eighty million
as senior and twenty million as junior. The senior claim takes the lower yield, is paid
first, and is reduced last. The junior claim takes the higher yield, is paid last, and is
reduced first. The twenty million junior layer is the cushion, and the entire economic
content of the structure is the size of that cushion relative to the losses the pool
might plausibly produce. Everything else is implementation.

| Layer | Size | Yield | Order of payment | Order of loss |
|---|---|---|---|---|
| Senior | 80M | Lower | First | Last |
| Junior | 20M | Higher | Last | First |

## Cash descends and loss ascends

Two flows run in opposite directions through the same structure, and conflating them is
the most common error in reading one. Cash descends: income fills the senior entitlement
in full and only the residue passes down to the junior claim. Loss ascends: a default
reduces the junior layer first and reaches the senior layer only once the junior layer
has been written to nothing.

Structures typically run two of these waterfalls in parallel rather than one. An interest
waterfall distributes income and a principal waterfall distributes repayments of
principal, and the two can carry different orderings, different tests and different cure
mechanics. On chain implementations almost universally collapse this into a single
distribution, which is a simplification worth noticing because several traditional
mechanics only exist in the separation.

The arithmetic on a worked example is deliberately unremarkable. If the pool earns eight
million and defaults nothing, a four million senior coupon is paid and the junior claim
takes the remaining four million, a return of roughly twenty percent on twenty million of
capital. If the pool instead suffers fifteen million of defaults, the junior layer absorbs
all of it and falls to five million while the senior layer remains whole at eighty. The
leverage is visible in both directions: the junior claim earned twenty percent on a pool
that yielded eight, and lost seventy five percent of its capital on a pool that lost
fifteen.

## Attachment and detachment points define a tranche precisely

Describing a tranche as senior or junior is imprecise once there are more than two of
them, and the precise description is a pair of numbers. A tranche attaches at the level of
cumulative loss where it begins to be impaired and detaches at the level where it is
exhausted. A tranche attaching at five percent and detaching at fifteen is untouched
while losses stay below five percent of the pool, absorbs losses between five and fifteen,
and is worth nothing beyond that.

This framing is more useful than the senior and junior labels for three reasons. It makes
two structures comparable when their labels are not. It makes explicit that a tranche is
a bounded exposure rather than an unbounded one, which is what allows it to be priced. And
it exposes the quantity that actually determines risk, which is the distance between the
attachment point and the loss the pool is expected to produce, rather than the name
attached to the layer.

Very few on chain structures state their attachment and detachment points, which is a
meaningful gap. Most state a minimum coverage ratio instead, which is the same
information viewed from the other side and expressed as a ratio rather than as a
percentage of the pool.

## Coverage tests convert a passive waterfall into an active one

The mechanic that distinguishes a managed structure from a passive one is the coverage
test: a ratio measured each period against a stated trigger, whose breach changes where
cash goes. Two families of test are standard.

The overcollateralisation test asks whether enough collateral principal stands behind a
claim. It is computed as the par value of the collateral pool divided by the par value of
the liabilities senior to and including the tranche being tested. A hundred million pool
against an eighty million senior claim gives a ratio of 1.25, which passes a trigger of
1.10 comfortably. It is an asset test and it responds to defaults and to writedowns.

```
OC ratio = par value of the collateral pool
           / par value of liabilities senior to and including this tranche
```

The interest coverage test asks whether the pool generates enough income to service the
claim. It is computed as interest collected over the period divided by interest due on
the same set of liabilities. Eight million of income against four million of senior
coupon gives 2.0 against a typical trigger of 1.20. It is an income test and it responds
to non accrual and to falling rates on the asset side.

```
IC ratio = interest collected over the period
           / interest due on liabilities senior to and including this tranche
```

The consequence of a breach is what matters. While the tests pass, the senior coupon is
paid and the junior claim receives the residue. While a test is breached, the senior
coupon is still paid, junior distributions are suspended, and the cash that would have
reached the junior claim is applied instead to reducing senior principal until the ratio
recovers. The structure delevers itself out of the breach using the junior holder's
distributions.

In stress the junior layer therefore absorbs the shortfall first through two separate
channels: it takes the loss on the asset side, and it forgoes its distributions on the
income side while the structure repairs itself. That double effect is the protection
tranching is supposed to provide, and it is the part on chain implementations almost
never build.

## On chain structures reproduce the ordering and omit the tests

Surveyed on chain protocols reproduce the layered structure faithfully, usually as a
senior token and a junior token, and then omit the coverage tests entirely. Their junior
layer absorbs loss passively, by its net asset value falling, and no cash is ever
redirected in response to a measured ratio.

| Behaviour | Present on chain |
|---|---|
| Passive loss absorption, junior value declines | Yes, essentially universal |
| Suspension of new senior issuance on breach | Centrifuge, Strata |
| Restriction of exit that varies with coverage | Strata, Royco |
| Deferral of loss recognition after a drawdown | Royco |
| Diversion of income to satisfy the senior entitlement | Strata |
| Diversion of cash to retire senior principal and cure | None |
| Excess spread trapped in a reserve before the residual | Approximated by the fixed allocation Goldfinch uses |
| Sequential waterfall over three or more tranches | Attempted by Waterfall DeFi, now defunct |

Reading that table from the top down is reading the sector in order of how far it has
got. Everything above the line of diversion is a way of preventing a ratio from
deteriorating or of postponing the moment it is measured. Nothing on chain repairs a
ratio that has already broken. The evidence protocol by protocol is in
[`../03-protocol-landscape/`](../03-protocol-landscape/README.md).

## The collateralised loan obligation is the reference implementation

The collateralised loan obligation is the canonical at scale version of the mechanic
above, and it is the structure against which the on chain designs are best measured. It
is a managed securitisation of below investment grade corporate loans arranged into a
rated capital stack, and the figures below are indicative ranges from educational and
manager sources rather than any single indenture.

| Layer | Share of the stack | Character |
|---|---|---|
| AAA senior | About 65 percent | Lowest coupon, most remote from loss, paid first |
| Mezzanine | About 4 to 12 percent per tranche | Rated AA, A, BBB, BB, with coupon and risk rising down the stack |
| Equity | About 8 to 10 percent | Unrated, no coupon, residual, absorbs loss first |

Three features of the reference structure are absent from every on chain implementation
in this repository, and each is worth stating separately.

The first is that coverage is a ladder rather than a single test. A collateralised loan
obligation computes an overcollateralisation and an interest coverage ratio at every
rated level of the stack, each with its own trigger, so a deterioration that is
immaterial to the AAA claim can still divert cash away from the BB claim. Indicative
triggers run from roughly 127 to 132 percent at AAA down to roughly 103 to 107 percent at
BB. A structure with one test at one boundary is a considerably simpler instrument.

| Tranche | Overcollateralisation trigger | Interest coverage trigger |
|---|---|---|
| AAA | 127 to 132 percent | About 120 to 130 percent |
| AA | 118 to 122 percent | Not separately stated |
| A | 113 to 117 percent | Not separately stated |
| BBB | 108 to 112 percent | Not separately stated |
| BB | 103 to 107 percent | Not separately stated |

The second is that there is more than one way to cure. After the reinvestment period the
cure is senior paydown, meaning cash is diverted to retire senior principal and shrink
the denominator of the ratio. During the reinvestment period, which typically runs up to
five years, a higher threshold test diverts a portion of interest proceeds to purchase
additional collateral instead, curing the ratio by rebuilding the numerator. A structure
that permits only paydown has one of the two, and a structure that permits neither has
neither.

The third is thickness of first loss capital, and it is the sharpest contrast in this
repository. Equity in a collateralised loan obligation is roughly eight to ten percent of
the structure. The cover position in a Maple pool is on the order of zero to three
percent and frequently unused. Royco publishes coverage floors of roughly three to twenty
percent depending on the market, and an independent review of one live vault measured
actual coverage at 12.4 to 12.5 percent against a ten percent requirement. Re is the
outlier in the other direction, carrying a first loss layer funded by the operator rather
than sold to depositors.

Worked figures, drawn from the same sources: a five hundred million pool against a three
hundred million AAA tranche gives an overcollateralisation ratio of 167 percent against a
130 percent trigger, a comfortable pass. On the income side, a pool earning SOFR plus 450
basis points against a AAA tranche paying SOFR plus 135 produces roughly 47.5 million of
income against 19.05 million of expense, an interest coverage ratio near 249 percent
against a 125 percent trigger.

## The arithmetic is exact and the thresholds are judgements

A waterfall has two layers that behave very differently, and treating them as one is the
most consequential category error available in this subject.

The first layer is fixed. The priority of payments, meaning pay the senior claim,
distribute the remainder to the junior claim, and divert distributions to cure a breach,
is arithmetic and ordering. For a given set of inputs the result is exact and
reproducible, and anyone can verify after the fact that a distribution followed the
stated rules.

The second layer is calibration, and every number feeding the first layer belongs to it.
The overcollateralisation trigger, the interest coverage trigger, the attachment and
detachment points that set the division between senior and junior, and the assumed
correlation of defaults across the pool are all modelling judgements rather than
determinable constants. They are conditional estimates: the senior layer is protected
only while losses remain inside the junior cushion and defaults do not correlate more
strongly than assumed.

The 2008 failure of the Gaussian copula approach is the standard illustration and is
usually described imprecisely. The payment mechanism worked exactly as specified. What
failed was a correlation assumption that held in calm conditions and did not hold when
defaults arrived together, so senior tranches rated as remote from loss were impaired.
The mechanism was sound and the calibration was wrong, and no amount of care in the first
layer compensates for error in the second.

The practical consequence for reading any protocol in this repository is that two designs
running identical arithmetic can behave completely differently because their thresholds
differ. Separate what the code computes from what somebody chose. The advantage the on
chain form has over traditional practice is that the calibration can be public and
inspectable rather than buried in a rating and a private indenture, and whether a given
protocol takes that advantage is among the more revealing questions to ask of it. Strata
is the clearest case where it does, publishing its parameter set per market and keeping
it separate from the arithmetic that consumes it.

## Terms used in this repository

**Attachment and detachment point.** The lower and upper bounds of the loss range a
tranche absorbs.

**Combined ratio.** In insurance, claims and expenses as a percentage of premiums earned.
Above one hundred percent the book lost money on underwriting.

**Coverage ratio.** A measure of how well a claim is covered, by assets in the
overcollateralisation test and by income in the interest coverage test.

**Cure.** Restoring a breached coverage ratio above its trigger, either by retiring senior
principal or by rebuilding the collateral.

**Diversion.** Sending cash that would have reached a junior claim to a senior claim
instead, while a coverage test is breached.

**Epoch.** A settlement period. Orders accumulate during the epoch and settle together at
its boundary, so the per tranche split is computed once for everybody.

**Excess spread.** Income remaining after every obligation has been met. In traditional
structures a portion is trapped in a reserve rather than paid to the residual claim.

**First loss.** The claim that absorbs loss before any other.

**Junior tranche.** The subordinated claim. Paid last, impaired first, compensated with a
higher return.

**Mezzanine.** Any claim ranking between the senior and the residual claim.

**Net asset value.** What the pool is held to be worth. The principal trust assumption in
any tranched structure.

**Quota share.** A reinsurance arrangement in which the reinsurer takes an agreed
proportion of both premiums and claims.

**Senior tranche.** The protected claim. Paid first, impaired last, compensated with a
lower return.

**Subordination.** The general property of one claim ranking behind another.

**Subordination ratio.** The share of a pool funded by junior capital.

**Waterfall.** The ordered sequence by which cash is distributed across claims.

## Sources

Coverage test formulas, indicative triggers, the worked example and the breach mechanics:
[collateralizedloanobligations.com](https://collateralizedloanobligations.com/mechanics/coverage-tests).
Capital structure, the reinvestment period, the parallel waterfalls and the role of the
manager: [Guggenheim, Understanding Collateralized Loan Obligations](https://www.guggenheiminvestments.com/perspectives/portfolio-strategy/understanding-collateralized-loan-obligations-clo/).
On chain coverage behaviour, protocol by protocol, with citations:
[`../03-protocol-landscape/`](../03-protocol-landscape/README.md).
