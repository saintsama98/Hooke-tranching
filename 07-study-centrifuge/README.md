# Centrifuge

> Study of a live system. Centrifuge V3 is deployed across eight networks and its
> flagship product is the largest tokenised AAA collateralised loan obligation fund in
> existence. This document covers what is running now.

## Centrifuge is the only protocol here whose product is an actual CLO tranche

Every other protocol in this repository constructs a tranche out of a DeFi yield source.
Centrifuge does something categorically different with its flagship product: it brings on
chain a claim on tranches that already exist in traditional credit markets. JAAA, the
Janus Henderson Anemoy AAA CLO Fund, is a tokenised feeder into the flagship AAA
collateralised loan obligation strategy of a manager that runs the largest AAA CLO
exchange traded fund in traditional markets. The underlying is a portfolio of the senior
most tranches of real collateralised loan obligations, carrying floating coupons that
reset off SOFR.

The significance for this repository is that it closes a loop the rest of the sector
leaves open. [`../01-fundamentals/`](../01-fundamentals/README.md) describes the
collateralised loan obligation as the reference implementation of a complete waterfall,
with a coverage ladder at every rated level and two distinct cure paths, and observes
that no on chain protocol implements it. Centrifuge does not implement it either. It
buys it. The waterfall runs in the traditional structure where it has always run, and the
token represents the resulting claim.

That is worth stating plainly rather than treating as a disappointment. A tokenised
senior CLO tranche gives a holder exactly the exposure the coverage ladder produces,
including the diversion of cash away from junior claims during a breach, without any of
that mechanism needing to exist in Solidity. Whether the sector should be building the
engine on chain or buying its output is a genuine open question, and Centrifuge is the
largest single piece of evidence for the second answer.

## The scale is not comparable to the rest of the sector

JAAA came on chain in June 2025 and reached roughly 687 to 690 million dollars of assets
under management across eight networks. It is described by Centrifuge as the largest
tokenised AAA CLO fund, and it was seeded by a one billion dollar allocation from Grove.
The underlying portfolio has been reported as holding 36 collateralised loan obligations
with no defaults. In June 2026 it became the first tokenised AAA CLO fund available
through Kraken Custody, which matters less for the mechanics than for what it says about
the intended holder.

For comparison, the entire risk tranching sector covered in
[`../03-protocol-landscape/`](../03-protocol-landscape/README.md) runs from about ten
million dollars at Royco to about two hundred million at Re. Centrifuge's single flagship
product is larger than all of the native tranching protocols in this repository combined,
and it achieved that by not being a native tranching protocol.

The claim commonly made alongside these figures, that no AAA rated CLO tranche has ever
defaulted or suffered a principal loss in over thirty years, is accurate as far as it
goes and should be read with the calibration argument in
[`../01-fundamentals/`](../01-fundamentals/README.md) in mind. A thirty year record
covering two credit cycles is real evidence about a structure that is deliberately built
to survive them. It is not a guarantee, and the attachment point of a AAA tranche, at
roughly thirty five percent of the stack beneath it, is the reason for the record rather
than the record being the reason for confidence.

## V3 moved the pool logic onto EVM chains and kept coordination separate

The current architecture inverts the earlier arrangement. Pools, tranche tokens and
investment logic are native to EVM chains and written in Solidity, while the Centrifuge
Chain retains governance, asset documentation and protocol level metadata. Deployments
span Ethereum, Base, Arbitrum, Avalanche, BNB Chain and Plume, with interoperability
handled by Wormhole, and JAAA is present on eight networks including Solana.

Two consequences follow for anyone studying subordination rather than infrastructure. The
first is that a multichain deployment forces the accounting to be authoritative somewhere
and replicated elsewhere, which is a different problem from the single chain epoch
settlement the sector started with. The second is that the coverage and pricing
arithmetic that a reader might want to inspect has moved off chain in the course of these
migrations, so the current system is not the place to look for verifiable subordination
logic. It is the place to look for distribution.

## The multi tranche system supports mezzanine layers that the sector otherwise lacks

Centrifuge documents a multi tranche system rather than a fixed senior and junior pair.
The top tranche is the senior tranche, the bottom is the junior tranche, and any number
of tranches in between are mezzanine tranches. This puts Centrifuge alongside Re as one
of only two protocols in this repository capable of expressing more than one subordination
boundary, and it is the only one where the additional boundaries are a general feature
rather than a property of a single product.

Whether pools use that capability is a separate question from whether the protocol offers
it, and it is the question worth asking of any protocol advertising multi tranche
support. A three tranche capability with two tranche deployments is a two tranche system
with optionality. The sector wide observation in
[`../03-protocol-landscape/`](../03-protocol-landscape/README.md), that essentially every
live structure has exactly one subordination boundary, holds until a deployed pool
demonstrates otherwise.

## deRWA wrappers address the liquidity problem by making the claim tradeable

The deRWA wrappers are the second mechanism worth studying, and they attack the problem
described in [`../04-liquidity/`](../04-liquidity/README.md) from a direction none of the
native protocols use. Compliance is enforced upstream at the pool level, and the wrapper
makes the resulting token transferable and composable, so that yield bearing real world
asset tokens can trade on decentralised exchanges including Aerodrome, Balancer and
Curve.

The design insight is that permissioning and liquidity are separable concerns that the
sector usually conflates. A pool that must know its holders does not thereby require a
token that cannot move. Enforce eligibility at the point of entry to the pool, and the
claim itself can be a plain transferable token afterwards. Set against the alternatives,
this is a third answer to the exit problem: Centrifuge neither restricts exit like a
coverage gate nor pays a market maker like Royco, it simply makes the position saleable
and lets a market price it.

The limitation is the same one that applies to every secondary market answer. A holder who
sells has exited at whatever discount the market applies, and that discount will be widest
under exactly the conditions that would have triggered a gate elsewhere. What is gained is
that the pool is not drained, since a sale moves the claim between holders without
withdrawing capital, which preserves the subordination structure for everyone who stays.

## Reading Centrifuge is now a distribution study rather than a mechanism study

The honest summary is that Centrifuge has become the most commercially successful entity
in on chain structured credit by moving away from the thing this repository studies. The
subordination logic that made its earlier work interesting is not where the assets are.
The assets are in a tokenised feeder into a traditional fund, distributed across eight
chains, wrapped for composability, and custodied by a regulated crypto custodian.

That trajectory is itself a finding, and it recurs. Pareto, covered in
[`../08-study-pareto/`](../08-study-pareto/README.md), made the same move from a general
purpose tranching engine toward institutional private credit. Re, covered in
[`../11-study-re/`](../11-study-re/README.md), started from the institutional side and
built inward. The direction of travel across the sector is from mechanism toward
distribution, and a reader who wants to understand where tranching is actually being done
at scale should expect to find it wrapped rather than built.

## Sources

[Centrifuge multi tranche documentation](https://docs.centrifuge.io/learn/multi-tranche-system/),
[deRWA tokens](https://centrifuge.io/derwa-tokens),
[JAAA launch and the Grove allocation](https://centrifuge.io/blog/centrifuge-janus-henderson-grove-tokenized-aaa-clo-fund),
[JAAA scale and composition](https://cryptobriefing.com/centrifuge-jaaa-tokenized-clo-fund/),
[JAAA in Kraken Custody](https://cryptobriefing.com/centrifuge-jaaa-tokenized-clo-kraken-custody/),
[JAAA asset page on RWA.xyz](https://app.rwa.xyz/assets/JAAA).
Figures are as reported at the dates given in those sources and are not continuously
updated here.
