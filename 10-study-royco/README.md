# Royco

> Study of a live system. Three legs instead of two, and the third one is paid to
> stand ready to buy. This document covers the structure, the mechanics, and what an
> independent reviewer found when they looked at a deployed vault.

Royco is a risk and liquidity tranching protocol, and it earns a place in this collection
for one structural reason: it is the only protocol here that splits a yield source three
ways rather than two, and its third leg absorbs no loss at all. That leg is paid to
provide an exit. Every other design in this repository treats the ability to leave as a
constraint to be managed by restricting withdrawals; Royco treats it as a service with a
price, funded out of the senior yield. Whether that is an improvement is genuinely open,
but it is a different answer to a problem the sector has otherwise addressed in only one
way, and it is running in production.

Reading about Royco is confusing until you separate three products that share a name.
**Royco IAM**, the Incentivized Action Marketplace, is the original and is not a tranching
protocol at all: incentive providers post rewards for action providers to perform an on
chain action, either a deposit into an ERC-4626 vault or an arbitrary sequence of
transactions, and offers converge on a price before actions and payment settle together.
**Royco Dawn** is the risk tranching protocol, with two legs, and its repository now
carries the note that it is a legacy deployment with no planned upgrades even though the
contracts still run. **Royco Day** is the current generation and the one that adds the
third leg, described by the protocol as splitting a yield source into senior, junior and
senior liquidity provider tranches so that a depositor chooses their liquidity profile as
well as their risk. If you read the Dawn source expecting to find the third leg you will
not, and if you read the current documentation expecting it to describe the deployed Dawn
contracts, it does not do that either.

Three things make it worth the reading time, in descending order of how transferable they
are. The third leg is a real idea and is portable to any tranched structure. The
observation period, in which a market pauses withdrawals for a fixed term after a loss on
the reasoning that a drawdown reversing inside the window was never a loss worth
realising, is a second idea of the same quality and is discussed in
[`04-liquidity`](../04-liquidity/README.md#every-protocol-restricts-exit-and-the-instruments-differ).
And it is by a distance the most heavily reviewed protocol covered here, with seventeen
audits across the products including formal verification by Certora and a competitive
audit on Cantina, alongside a 250,000 dollar bug bounty on Immunefi.

Set against that, the most useful single document about Royco is an unflattering one. Yearn
published an independent assessment of a live Royco senior vault and scored it elevated
risk, on grounds that have almost nothing to do with the tranching arithmetic and
everything to do with where the valuation comes from. That review is covered in
[independent review](.#an-outside-review-reached-an-unflattering-conclusion), and it should be read alongside the
mechanics rather than after them, because it is the clearest illustration in this
repository of the point that a tranched structure is only ever as sound as the number the
waterfall is applied to.

On scale, the figures below are what the sources say at the dates they give, and should be
treated as an indication rather than as current. Total value locked was about 10 million
dollars across six live markets with over 155,000 users; the USDC market showed a senior
yield near 8.3 percent against a junior yield above 40 percent at 96 percent utilisation;
and the protocol front page carries an all time volume figure of 3.1 billion dollars. That
last number is worth reading carefully rather than glossing over, because most of it
belongs to the Incentivized Action Marketplace moving large sums through short lived
incentive campaigns, and not to the tranching protocol at all.

Sources:
[Royco overview](https://docs.royco.org/royco-overview),
[Royco Dawn repository](https://github.com/roycoprotocol/royco-dawn/),
[Royco IAM repository](https://github.com/roycoprotocol/royco-iam),
[security page](https://www.royco.org/security),
[Portals](https://blog.portals.fi/defi-drop-royco-dawn-risk-tranching/).

---

## Mechanics

How the protocol divides a yield source, and what the contracts behind it do. Sources
are the protocol documentation and the Dawn repository, both cited inline. Where the
documentation describes the current generation and the repository describes the legacy
one, this note says which is which.

## The three legs

Royco splits one yield source into three positions rather than the usual two.

The senior tranche takes the least yield and receives two distinct things in return:
seniority, meaning junior capital absorbs loss before it does, and access to secondary
liquidity, meaning it can leave without waiting for the underlying source to release
capital. The junior tranche takes the most yield, serves as first loss capital, and earns
the base yield of the asset plus a risk premium paid out of the senior yield. Neither of
those is unusual.

The third leg is. A senior liquidity provider supplies an automated market maker pool
made of senior shares against the quote asset of the market, so that a senior holder who
wants out can sell into a standing bid rather than queue for a redemption. It earns a
liquidity premium, the trading fees of the pool, and the discount it captures when a
senior holder sells shares into it. The documentation puts the role plainly: the provider
"gets paid to be the exit".

Yield flows from the senior leg to both of the others, "paying for Senior's coverage and
liquidity based on pre defined ratios". So the senior holder is buying two separate
things out of the same yield: protection, from the junior leg, and an exit, from the
SLP. That is the structural novelty here, and it is worth separating in your head from
the ordinary senior and junior arrangement.
Source: [Royco overview](https://docs.royco.org/royco-overview).

## Coverage

The rule is a minimum rather than a band: "Every ST position has a minimum amount of JT
capital in front of it. ST is not impaired until this JT capital is fully exhausted."

The coverage floor varies by market. One curator writing about the protocol gives a
range of roughly 3 to 20 percent across markets, which is worth holding next to the 8 to
10 percent equity layer of a collateralised loan obligation in
[`01-fundamentals`](../01-fundamentals/README.md).
Source: [Dialectic, Risk as a Product](https://dialecticgroup.substack.com/p/risk-as-a-product),
July 2026.

## How the yield is divided

The Dawn repository names three yield distribution models, which decide how much of the
senior yield flows to the junior leg.

| Model | Behaviour |
|---|---|
| Static curve | A fixed piecewise curve |
| Adaptive V1 | The curve scales with how far utilisation has moved from its target |
| Adaptive V2 | The curve moves vertically, at a configurable speed of adaptation |

The intent, as described from outside, is to keep a slight surplus of junior capital: as
the surplus shrinks more yield is directed to the junior leg to attract more of it.
Source: [Royco Dawn repository](https://github.com/roycoprotocol/royco-dawn/) and
[Portals blog](https://blog.portals.fi/defi-drop-royco-dawn-risk-tranching/).

This is the same family of idea as the Idle Adaptive Yield Split and the Strata Dynamic
Yield Split. All three vary the division of yield with coverage rather than fixing it.
See [`08-study-pareto`](../08-study-pareto/README.md)
and [`09-study-strata`](../09-study-strata/README.md).

## The observation period

The mechanism most worth taking from Royco after the SLP.

A market runs in one of two states. In the normal state, called perpetual, both legs are
liquid. After a loss the market can move to a fixed term state, which restricts
withdrawals for a set period.

The reasoning is that a drawdown which reverses inside the window was never a loss, so
realising it would harm holders for no reason. Only a loss that persists past the window
is applied to junior capital. It is a deliberate trade of liquidity for the avoidance of
forced marking, and it is closer to the discipline of a traditional fund than to normal
DeFi behaviour.

The length is set per market. An outside review of one Royco vault records a two day
fixed term state on one underlying market and none at all on another, where losses are
absorbed immediately with no recovery period.
Sources: [Royco Dawn repository](https://github.com/roycoprotocol/royco-dawn/),
[Portals blog](https://blog.portals.fi/defi-drop-royco-dawn-risk-tranching/),
[Yearn risk report](https://github.com/yearn/risk-score/blob/master/reports/report/royco-srroyusdc.md).

## Any asset can be tranched because the valuation is pluggable

The design keeps the party that computes value separate from the party that moves it, and
puts the pricing of the underlying behind an interchangeable component. The practical
consequence is that the engine does not need to know what it is tranching. Published
support extends to plain tokens, to standard vault shares, and to positions specific to
Maple, Pareto, Metastreet, Re and Makina.

Two of those names deserve a second look. Pareto is the renamed Idle and runs its own
senior and junior structure, and Re runs the three layer reinsurance stack studied in
[`../11-study-re/`](../11-study-re/README.md). So the list is not merely a list of assets.
It is a list that includes two tranching protocols, which means Royco is positioned to
tranche a tranche, and the section on recursive structures in
[`../06-optimizations/`](../06-optimizations/README.md) records that this has
now happened in production.

## Where the valuation comes from

This is the part to read most carefully, and it is covered in full in
[independent review](.#an-outside-review-reached-an-unflattering-conclusion).

The coverage arithmetic is on chain and checkable: each market exposes its coverage ratio
and its loss waterfall through the Accountant. What that arithmetic is applied to is
another matter. In at least one live vault the total asset figure is reported by an
administrator through a function call, with no oracle behind it.

The whole structure sits on that number, exactly as
[`03-protocol-landscape`](../03-protocol-landscape/README.md#the-protocol-landscape) says every tranched
structure does.

## Assessment against the common shape

Set against the five parts in
[`03-protocol-landscape`](../03-protocol-landscape/README.md#the-protocol-landscape).

The asset pool is provided and is unusually general, because the pluggable quoter means
the engine does not care what it is tranching. That is the same result Pendle reaches by
wrapping a source before splitting it, arrived at from the other direction. Valuation and
loss accounting are provided with a trust assumption that varies by market: per market
coverage is verifiable on chain through the Accountant, while the value that coverage is
measured against may not be. The waterfall is provided in the ordinary two leg form,
junior first on loss and senior first on income, with the division set by whichever yield
distribution model the market uses.

Coverage tests and cure are partial. There is a minimum coverage level and a loss state
that restricts withdrawals, and there is no diversion of cash flow to repair a breached
ratio, so like every other protocol in this repository the structure does not delever.
What the observation period contributes is a delay before a loss is recognised at all,
which is a different thing from a cure and, for a volatile underlying, arguably a more
honest one.

Settlement is where Royco actually differs. Every other protocol answers the exit problem
by restricting exit, and Royco answers it by paying a third party to stand ready to buy,
which moves the friction from a queue into a price. That change and its limits are
discussed in
[`04-liquidity`](../04-liquidity/README.md#secondary-markets-relocate-the-friction-rather-than-removing-it).

---

## An outside review reached an unflattering conclusion

Yearn published a risk assessment of srRoyUSDC, a live senior vault built on Royco Dawn,
and scored it 3.80 out of 5.0, which its own scale calls elevated risk. It is the most
useful document about Royco in this repository, because almost nothing it objects to is
about tranching. Source:
[Yearn risk score report](https://github.com/yearn/risk-score/blob/master/reports/report/royco-srroyusdc.md).

## What the vault is

Depositors put USDC into a senior vault and receive srRoyUSDC, an ERC-4626 token
representing senior exposure spread across several whitelisted markets. At the time of
the report the capital sat across five underlying protocols: Avant, Neutrl, Aave v3,
Auto and Cap Finance. Junior depositors in each market absorb loss before the senior
holders do.

## The findings

The central finding is that the value is reported rather than measured. The whole asset
figure, about 10.73 million dollars, depends on an administrator calling a function to
adjust the total, and the report is blunt about what that means: "There is no oracle.
Deployed asset values are admin reported." Custody compounds it, because once funds reach
the treasury multisig there is "no onchain guarantee that forwarded funds will be
deployed to approved markets or returned to the vault". Between those two facts, the
tranching arithmetic is operating on a number that no contract can verify and over assets
no contract controls.

The remaining findings are more familiar but not less serious. The coverage buffer was
thin, with junior protection at 12.4 to 12.5 percent against a 10 percent requirement, so
roughly two and a half percentage points of cushion stood between the structure and the
point at which the senior leg is exposed. A single externally owned account held about 69
percent of the supply, which is a liquidity risk and a governance risk at once. Around 70
percent of the allocation sat in two protocols the report describes as "relatively
unknown" with "limited public track record". And a three of five multisig could upgrade
the strategy proxy immediately, with no timelock in the way.

## On liquidity, specifically

Worth reading against the account of the third leg above.

The primary exit is an asynchronous withdrawal queue taking up to fourteen days. The
secondary exit is a Curve pool, which held about 490,000 dollars at the time of the
report, having grown from roughly 600 dollars at launch. There is a larger position of
about 1.99 million dollars on Morpho Blue, but it exits into pmUSD rather than into USDC
directly.

Two things follow. Secondary liquidity for a senior tranche does not appear on its own,
and at the point measured it covered a small fraction of the vault. And an exit that
lands in a different token than the one you deposited is not quite an exit.

## Why this note is here

Everything in this repository about valuation being the real trust assumption is
abstract until you read a document like this one. The tranching arithmetic in Royco is
on chain, audited seventeen times, and formally verified in places. The number that
arithmetic is applied to, in this particular vault, is typed in by an administrator.

That is not a criticism unique to Royco. It is what
[`03-protocol-landscape`](../03-protocol-landscape/README.md#the-protocol-landscape) predicts for any
tranched structure, and the reason valuation sits above the waterfall in that list
rather than beside it. Royco is simply the case where somebody independent went and
checked.

The report recommends limited approval with strict monitoring, including continuous
auditing of the calls that adjust the total asset figure.
