# Forms of Tranching

> Reading order: after [`../01-fundamentals/`](../01-fundamentals/README.md). This
> document distinguishes the kinds of split that exist, because protocols that describe
> themselves identically are frequently doing different things.

## There is more than one thing you can split

Tranching divides one pool into several claims, and the useful first question about any
tranched product is which dimension it divides. Two answers account for nearly everything
built on chain, and a third exists in traditional markets and barely on chain at all.

Risk tranching divides by who absorbs loss first. The legs are senior and junior, with
mezzanine layers in between where the structure supports them, and the protected leg is
protected because another leg has agreed to be impaired ahead of it. Cash flow tranching
divides by which component of the return a holder owns. The legs are principal and yield,
and the protected leg is protected because it is defined to redeem at par at a stated
maturity while the other leg absorbs everything variable. Sequential tranching divides by
strict payment order across three or more layers, and is the arrangement a collateralised
loan obligation uses.

| | Risk tranching | Cash flow tranching |
|---|---|---|
| Dimension divided | Exposure to loss | Composition of the return |
| Legs | Senior and junior, sometimes mezzanine between | Principal and yield |
| Protected leg | Senior, protected by a capital cushion | Principal, protected by definition at maturity |
| Source of protection | Another party's capital | An accounting rule about the split |
| Term | Usually perpetual | Usually a fixed maturity |
| Typical representation | One token per tranche, sometimes vault shares | Principal and yield tokens over a yield wrapper |
| Examples | Pareto, Centrifuge, Strata, Royco, Re, Goldfinch, Maple | Pendle, Spectra |

## Risk tranching sells a capital cushion

The structure is an ordered set of claims on one pool. The senior class is paid before the
junior class and impaired after it, and the junior class is compensated for that ordering
with a larger share of the income. The quantity that determines how much protection the
senior class actually has is the size of the junior class relative to the pool, and a
senior claim standing behind a twenty percent cushion is a different instrument from one
standing behind three percent whatever the documentation calls either of them.

Three approaches to computing the split appear in practice and they behave very
differently. A fixed share of income is the simplest and the worst, because the junior
claim receives the same compensation whether its cushion is thick or thin, so its reward
is unrelated to the protection it is providing on any given day. A share that varies with
coverage repairs that, and is what Pareto and Strata both do, though from opposite
directions: Pareto recomputes the senior share of income from the relative size of the two
tranches, while Strata prices the senior rate directly and treats the junior claim as the
residual. A senior rate with the junior claim as residual has a further consequence worth
isolating, which is that the junior claim becomes short an interest rate floor in addition
to being long first loss credit exposure, and a period in which the underlying earns less
than the floor debits the junior balance in exactly the way a default would.

Loss allocation is almost universally passive on chain. The net asset value falls, the
junior claim absorbs the fall until it reaches zero, and only then is the senior claim
touched. No ratio is measured and no cash is redirected. Traditional structures add the
active step described in [`../01-fundamentals/`](../01-fundamentals/README.md), and the
two on chain protocols that come closest each stop short in a different way. Centrifuge
refuses to execute a settlement that would push the subordination ratio out of its band,
which prevents deterioration through redemption and repairs nothing. Strata diverts junior
capital to satisfy the senior income entitlement, which is a genuine cure on the income
side, and never retires senior principal, so the structure does not delever.

## Cash flow tranching sells a definition rather than a cushion

An interest bearing asset is separated into two tokens. The principal token redeems one
for one for the underlying at a fixed maturity. The yield token carries whatever the asset
earns between now and that maturity. Buying the principal token at a discount and holding
it to maturity produces a fixed return; buying the yield token is a position on the future
yield of the source.

The critical distinction, and the one most often blurred when the two forms are discussed
together, is that there is no first loss capital here in the credit sense. The principal
token is insulated from variation in the yield because it is defined to redeem at par, not
because any party has agreed to absorb a default ahead of it. If the underlying source
fails outright, no counterparty stands in front of the principal holder. A protocol that
describes itself as tranched is therefore not necessarily providing subordination, and the
question to ask is what happens if the source does not merely underperform but breaks.

The construction technique is nonetheless the more transferable of the two. Wrapping an
arbitrary yield source into a standardised form before splitting anything means the split
engine reasons about a single number, an exchange rate, whatever the source happens to be.
One engine then works over staking derivatives, lending receipts and liquidity positions
alike. The internal index is designed not to decrease, so volatility in the source is
absorbed by the yield leg and the principal leg is insulated from it, which is
structurally the same technique a risk tranche uses to keep a senior claim insensitive to
short term movement.

## The third leg in a structure need not absorb loss

Royco splits a yield source three ways rather than two, and the third leg absorbs nothing.
It supplies secondary liquidity so that the senior claim can be sold rather than redeemed,
and it is paid for that service out of the senior yield. The senior holder is therefore
buying two distinct things from one stream of income: protection, from the junior leg, and
an exit, from the liquidity provider.

This is the only structure in this repository where liquidity is priced as its own
position rather than managed as a constraint, and it is a genuine addition to the design
space rather than a variation on the existing one. It is discussed in
[`../04-liquidity/`](../04-liquidity/README.md) and studied in
[`../10-study-royco/`](../10-study-royco/README.md).

## A first loss layer need not be sold to depositors

Re inverts a different assumption. Its stack has three ordered layers, but the layer that
absorbs first is the equity of the reinsurer itself, and depositors cannot buy it. The two
tokenised legs, mezzanine and senior, both sit above a cushion the operator funds.

That arrangement dissolves the junior demand problem described in
[`../04-liquidity/`](../04-liquidity/README.md), which was the binding constraint on
almost every protocol that failed in this sector, and replaces it with a different
problem. The adequacy of that first loss layer is now something a depositor takes on trust
from an off chain balance sheet rather than reads from a contract. Both arrangements have
a weak point. In the conventional design the weak point is whether anyone will buy the
junior claim at a price the senior claim will accept. In this one it is whether the equity
said to stand behind the structure is really there and really available.

## Sequential tranching exists in theory and barely on chain

Traditional securitisation runs three or more layers paid strictly in order, with a
coverage test at every boundary. On chain this is almost entirely absent. Essentially
every live structure has exactly one subordination boundary. Waterfall DeFi attempted a
three tranche sequential waterfall and is defunct. Centrifuge documents support for
mezzanine layers, which is a capability rather than a demonstrated deployment.

The exception is instructive and belongs in this document rather than elsewhere. Centrifuge
is running the largest tokenised claim on a genuine multi layer waterfall, and it does so
by tokenising a feeder into a fund that holds the senior tranches of real collateralised
loan obligations. The full coverage ladder exists, the diversion mechanics exist, and none
of it is on chain. The claim is. That is covered in
[`../07-study-centrifuge/`](../07-study-centrifuge/README.md).

## What the two forms share

Both take one source of return, apply a rule for dividing it, and issue a protected leg
and an exposed leg. In both, the token representations received standards and the engine
performing the split did not, so every protocol writes its own and no two are readily
comparable. And in both, the protection offered to the protected leg is an accounting rule
about how a measured quantity gets divided, which means the soundness of the protection
depends entirely on whether the measurement is honest. That dependency is the subject of
[`../01-fundamentals/`](../01-fundamentals/README.md) and it is the constant across
everything in this repository.
