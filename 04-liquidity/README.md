# Liquidity in Tranching

> Reading order: after [`../03-protocol-landscape/`](../03-protocol-landscape/README.md).
> How capital gets into a tranched pool and, far more importantly, how it gets out. This is
> where most of these designs are actually decided, and where the ones that failed
> generally failed.

How capital gets into a tranched pool and, far more importantly, how it gets out. This is
where most of these designs are actually decided, and where the ones that failed
generally failed. Four notes follow.
[epochs and settlement](.#settlement-happens-in-batches-for-a-structural-reason) covers why tranched pools settle
in batches at all and what an epoch boundary is for.
[exit friction and gating](.#every-protocol-restricts-exit-and-the-instruments-differ) covers the instruments
protocols use to slow an exit, which range from a flat refusal to a graduated fee.
[junior demand and first loss](.#junior-demand-is-the-binding-constraint-on-the-whole-sector) covers the problem
of finding capital willing to absorb loss first, and how thin that layer is in practice.
[secondary markets](.#secondary-markets-relocate-the-friction-rather-than-removing-it) covers why tranche tokens are usually
plain ERC-20s and what happens when a protocol decides to pay somebody to be the exit.

A tranched pool has a structural liquidity problem that a single class pool does not, and
it follows directly from what makes the structure work. The senior claim is protected
only while the junior cushion is actually present. If junior holders can leave freely,
they will leave precisely when conditions deteriorate, which is the moment their capital
is doing the work it was paid for. So every tranched design has to restrict junior exit
in some fashion, and every restriction makes the junior position harder to sell in the
first place. Junior demand was the binding constraint on almost every protocol in this
sector, and several of them died of it.

The instruments in use span a wider range than the sector's uniform appearance suggests.
Centrifuge refuses outright, rejecting any epoch allocation that would leave the
subordination band. Strata tiers the friction across up to three coverage bands, each
applying some combination of a share lock, an asset lock and a fee. Royco pauses the
market entirely for a fixed term after a loss, on the reasoning that a drawdown which
reverses inside the window was never a loss worth realising, and separately pays a third
party to stand ready to buy the senior leg so that exiting need not touch the pool at
all. Re does not gate anything and instead accepts a quarterly redemption window on its
mezzanine leg, because the underlying reinsurance treaty simply cannot return capital
faster than that. Some friction, as in the Idle integration with Ethena, is not a design
choice at all but a property of the asset underneath.

The second order effect is the one nobody has solved. Coverage driven restrictions are
procyclical: they tighten as conditions worsen, which is exactly when holders most want
to leave, which gives every holder a reason to leave before the others do. A design that
looks prudent in calm conditions can accelerate the run it was built to prevent, and a
junior holder who can read the coverage bands knows that exiting early is cheaper than
exiting late. Nothing surveyed here escapes this. The designs differ only in how
gradually the door closes, and in whether anyone is paid to stand on the other side of it.

---

# Settlement happens in batches for a structural reason

## Why batch at all

A tranched pool cannot settle a deposit or a withdrawal in isolation, because the price
of every claim depends on a split that has to be computed against the whole pool. If one
holder could redeem at any moment, the split would have to be recomputed on every
interaction, and each holder would settle against a different view of the pool.

Batching solves this. Orders accumulate over a period, the pool is valued once at the
boundary, the per tranche split is computed once, and everybody in that batch settles
against the same figures. The cost paid for it is that entry and exit are no longer
immediate.

This is why the epoch boundary is the natural place for a waterfall to execute. It is
the one moment at which the pool has been valued, the orders are known, and nothing is
in flight.

## How the batch is decided

The decision at the boundary is harder than it looks. When every order cannot be filled,
something has to choose who is filled and by how much, subject to the constraints of the
pool. That is a constrained optimisation, and it is expensive to solve on chain.

Centrifuge Tinlake inverts it: anyone may compute a candidate allocation off chain and
submit it, and the contract only checks it against the constraints and ranks it. A
challenge window opens on the first valid submission and the best one seen at the close
is executed. Checking is cheap even though solving is not. The mechanism is described in
[`07-study-centrifuge`](../07-study-centrifuge/README.md).

The weights that decide which allocation is better are a policy choice, not arithmetic:
the Tinlake default fills redemptions before subscriptions and senior before junior.
That is a reasonable answer and it is not the only one.

## Asynchronous settlement as a standard

The request and later fulfilment pattern is common enough that ERC-7540 standardises it,
as an asynchronous extension of ERC-4626. A holder requests a deposit or a redemption
and claims it once the pool has fulfilled it.

Adoption is thin. Centrifuge, which coauthored it, uses it in Liquidity Pools v3. Most
other protocols that need deferred redemption build their own. Strata needed variable
duration, state dependent deferral and built three separate lock contracts alongside
plain ERC-4626 rather than adopting the standard written for the case. See
[`09-study-strata`](../09-study-strata/README.md).

## What moving settlement changes

Centrifuge is the clearest case of the trade. Tinlake computes the coverage arithmetic
on chain and settles in epochs driven by the on chain solver. Liquidity Pools v3
reissues the tranches as ERC-7540 vaults across chains and moves the coverage and
pricing arithmetic off chain. The settlement interface got cleaner and standardised, and
the part that was independently checkable stopped being so. See
[`07-study-centrifuge`](../07-study-centrifuge/README.md).

---

# Every protocol restricts exit, and the instruments differ

Every tranched protocol restricts exit somehow, because the junior cushion only protects
the senior claim while it is still there. The designs differ in how the restriction is
expressed, and the range is wider than it first appears.

## A hard refusal

Centrifuge Tinlake bounds the subordination ratio between a minimum and a maximum, and
refuses to execute any epoch allocation that would leave the band. Junior redemptions
are therefore throttled to whatever the band permits.

This is binary. An allocation either satisfies the constraint or it does not, and there
is no middle setting. It is also, notably, not a cure: cash is not taken from the junior
claim and applied to the senior claim to repair a ratio that is already breached. It
prevents the ratio deteriorating through redemption and does nothing about a ratio that
deteriorated for any other reason. See
[`07-study-centrifuge`](../07-study-centrifuge/README.md).

## Graduated friction

Strata expresses the same concern as a set of ranges over senior coverage. Up to three
bands are defined, and each applies any combination of three instruments: a lock on
shares, a lock on assets, and a fee. As coverage falls the pool moves into a tighter
band and exit becomes progressively more expensive and slower.

Below a floor, in the Ethena market a coverage of 105%, senior issuance and junior
redemption are suspended together.

The band triple is the most transferable idea in that design. Three ranges, each with a
share lock, an asset lock and a fee, is a compact and complete description of exit
friction that varies with the state of the pool, and it is a much finer instrument than
a single threshold. Detail in
[`09-study-strata`](../09-study-strata/README.md).

## Waiting out a drawdown

Royco adds a fourth mechanism, and it is the most interesting one because it is not
really about liquidity at all.

A market normally runs perpetual, with both legs liquid. After a loss it can enter a
fixed term state that restricts withdrawals for a set period. The reasoning is that a
drawdown which reverses inside the window was never a loss, so realising it would harm
holders for nothing. Only a loss that persists past the window reaches junior capital.

That is a trade of liquidity for the avoidance of forced marking, and it is closer to how
a traditional fund handles a bad quarter than to normal DeFi behaviour. It is also
squarely a judgement call: the length of the window decides which movements count as
real, and it is set per market. One outside review records a two day window on one
underlying market and none at all on another, where losses are absorbed on the spot. See
[`10-study-royco`](../10-study-royco/README.md#mechanics).

Note what this is not. It does not repair a ratio and it does not move cash between legs.
It delays the recognition of a loss, which is a separate idea from a cure and, for a
volatile underlying, arguably a more honest one.

## Inherited waiting

Some friction is not a design choice at all but a property of the underlying asset. The
Idle integration with Ethena holds sUSDe, whose redemption requires a cooldown, so exits
are deferred behind that cooldown whether or not the tranche structure wants them to be.
The integration handles it with a clone contract per redemption request.

This case is worth separating from the other two. The delay carries no information about
the state of the pool. It is latency, not a signal, and a holder cannot infer anything
from it. See
[`08-study-pareto`](../08-study-pareto/README.md).

## Reading a gate

Three questions distinguish these mechanisms, and they are worth asking of any protocol.

The first question is what is being measured: a subordination ratio, a coverage ratio, or
nothing at all. The second is what happens on a breach, and the available answers are to
refuse, to slow, to charge, to redirect, or to wait. Only redirection is a cure.
Everything else either preserves a ratio without repairing it or defers the moment at
which the question gets asked, and the difference matters, because a structure that
prevents deterioration is not the same as one that recovers from it.

The third question is who gets restricted, and it is the one most often skipped.
Restricting junior exit protects the senior claim at the direct expense of the party
providing the protection, which is defensible only if the junior holder was paid for that
specific restriction rather than for the credit risk alone. Restricting both sides, as
Strata does below its floor, is a different instrument and a fairer one, though it
freezes the pool for everybody. Re restricts by term rather than by state, since its
mezzanine holders accept a quarterly redemption window from the outset, which at least
has the virtue of being knowable in advance.

## The unresolved part

All of these tighten as conditions worsen. That is the point, and it is also the risk: a
junior holder who can see the bands knows that exiting early is cheaper than exiting
late, which is an argument for leaving before anyone else does. None of the surveyed
designs resolves this. See
[junior demand and first loss](.#junior-demand-is-the-binding-constraint-on-the-whole-sector).

---

# Junior demand is the binding constraint on the whole sector

The junior tranche is the part that makes the structure work and the part that is
hardest to sell. Insufficient junior demand is the most common cause of failure among
the protocols surveyed here.

## Why it is hard to fill

A junior holder takes a leveraged position on the performance of the pool, absorbs loss
first, and is the party whose exit gets restricted when conditions deteriorate. The
compensation is a higher return, and the question is always whether it is high enough
for the risk actually taken.

Protocols answer it in four ways. The simplest is a fixed share of yield, and it is also
the worst. The junior claim receives the same share of income whether its cushion is
large or small, so its compensation bears no relation to the protection it is actually
providing on any given day, and the position becomes least attractive precisely when it
is most useful.

The second varies the share with coverage. Idle recomputes the senior share of yield from
the relative size of the two tranches, so the senior claim cedes more as the junior
cushion grows, and the junior claim is designed always to earn above the underlying
yield. That is a deliberate attempt to make the junior position scalable rather than
merely available, and the reasoning is set out in
[`08-study-pareto`](../08-study-pareto/README.md).

The third looks like the second and behaves differently. Strata prices the senior rate and
hands the junior claim the residual, and its risk premium curve turns out to be nearly
flat across the entire operating range, moving by only a couple of percentage points of
base yield between very high coverage and the circuit breaker. Junior compensation there
comes almost entirely from the leverage multiple, meaning the ratio of senior size to
junior size, rather than from the premium repricing. A leverage multiple and a risk
premium behave very differently as the composition of a pool moves, and a junior holder
should know which one is paying them.

The fourth sidesteps the question. Re does not sell its first loss layer at all: the
equity of the reinsurer absorbs first, and both tokenised legs sit above it. That removes
the demand problem for the layer that matters most and replaces it with a different one,
since the adequacy of that equity is now something a depositor takes on trust from an off
chain balance sheet rather than reads from a contract. See
[`11-study-re`](../11-study-re/README.md#mechanics).

## How thin it is in practice

The traditional benchmark is instructive. Equity in a collateralised loan obligation is
roughly 8 to 10% of the structure. On chain, the cover position in a Maple pool is on
the order of 0 to 3% and often unused. Royco sits between the two, with coverage floors
of roughly 3 to 20% depending on the market; an outside review of one live senior vault
measured actual coverage at 12.4 to 12.5% against a 10% requirement, so about two and a
half percentage points of cushion above the point at which the senior leg is exposed.
Sources: [Dialectic](https://dialecticgroup.substack.com/p/risk-as-a-product),
[Yearn risk report](https://github.com/yearn/risk-score/blob/master/reports/report/royco-srroyusdc.md).

That gap is the single most important number in this repository. A senior claim
described as protected by a 3% first loss layer is a materially different instrument
from one protected by a 10% layer, whatever either is called, and a structure with a
thin junior layer is closer to an undifferentiated pool with extra steps than to a
tranched one.

## What went wrong before

BarnBridge wound down after regulatory action. Saffron, Waterfall DeFi and Tranche
Finance are dormant or defunct. The recurring pattern in the ones that failed
commercially rather than legally is the same: not enough junior capital at a price the
senior side would accept, and the structure has no meaning without it.

The recent entrants approach the problem from the other end, by bringing institutional
credit on chain where a first loss investor base already exists. See
[`03-protocol-landscape`](../03-protocol-landscape/README.md#real-world-asset-securitisation-entrants).

## A distinction worth keeping

A junior tranche in a structure with a rate floor is carrying two different exposures:
it is long first loss credit risk and short an interest rate floor. Strata settles both
through a single debit of the junior balance, so a yield shortfall against the floor
impairs the junior tranche in exactly the way a default would. They are not the same
risk and pricing them as one obscures what is actually being held.

---

# Secondary markets relocate the friction rather than removing it

## Why tranches are usually plain ERC-20s

The standards best suited to representing a tranche are barely used for it. ERC-3475 was
designed for exactly this instrument and has minimal production adoption. ERC-1155 and
ERC-6909 offer several classes in one contract with shared accounting, and neither is
used for tranches in practice.

What protocols do instead is deploy one ordinary ERC-20 per tranche, with no shared
accounting between them. The reason is liquidity. A plain ERC-20 can be listed on a
decentralised exchange, supplied to a lending market, and used as collateral, without
anything on the other side knowing what a tranche is. A multi token identifier or a
class and nonce pair cannot, without integration work by every venue.

So the representation that is worse at describing the instrument is better at trading
it, and trading wins. Detail in
[`05-token-standards`](../05-token-standards/README.md#adoption-in-practice-diverges-from-what-the-standards-intended).

## What a secondary market provides

A tradable tranche token changes the liquidity position materially. A holder who wants
out does not have to wait for an epoch, a cooldown, or a coverage band to permit a
redemption. They can sell, at whatever discount the market applies.

That does not remove the friction, it relocates it. Redemption friction becomes a
discount to net asset value, and the discount widens under exactly the conditions that
would have tightened a gate. It does mean the pool itself is not drained, which is the
part that matters for the structure: a holder exiting through the market takes no
capital out of the pool at all, so the junior cushion stays where it is.

This is the one genuine advantage the on chain form has over the traditional structure,
where a junior position is generally illiquid until maturity.

## Paying somebody to be the exit

Royco takes the opposite approach to every other protocol here. Rather than treating
secondary liquidity as something that may or may not appear, it makes it a third position
in the structure and pays for it out of the senior yield.

The senior liquidity provider supplies an automated market maker pool of senior shares
against the quote asset of the market. It earns a liquidity premium, the trading fees of
the pool, and the discount it captures when a senior holder sells into it. In the words
of the documentation, it "gets paid to be the exit". Source:
[Royco overview](https://docs.royco.org/royco-overview).

This is a real answer to a real problem, and it is worth being precise about what it does
and does not solve. It gives a senior holder a way out that does not touch the pool, so
the junior cushion is undisturbed. It does not make the exit free: the price is whatever
the pool quotes, and the provider is compensated precisely for taking the other side when
nobody else will.

It also shows how thin such liquidity is in practice. An independent review of one live
Royco senior vault found the secondary pool holding about 490,000 dollars against a
supply of roughly 10.7 million, with the primary exit still an asynchronous queue of up
to fourteen days. It had grown from about 600 dollars at launch, so the direction is
right and the scale is small. See
[`10-study-royco`](../10-study-royco/README.md#an-outside-review-reached-an-unflattering-conclusion).

## The nonfungible leg

Two protocols issue one leg as an ERC-721 rather than an ERC-20: BarnBridge issued its
senior claim as dated sBONDs, and Goldfinch issues junior Backer positions as
PoolTokens.

The reason is that the leg carries terms specific to the holder, a maturity date or an
entry point in the loan, which a fungible token cannot express. The cost is that the
position is much harder to trade, which brings back the illiquidity the ERC-20 form was
chosen to avoid.

## Where cash flow tranching differs

Pendle builds its own automated market maker for the principal and yield tokens rather
than relying on general venues, because the pricing relationship between the two legs
and the time to maturity is specific enough that a general purpose pool prices it badly.
Its oracle takes the maximum of two indices specifically to resist downward manipulation.

That is worth noting as a limit on the composability argument. Where a tranche token has
a genuinely non standard payoff, listing it as a plain ERC-20 on a general venue is not
by itself sufficient for it to trade well.
