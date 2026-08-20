# Tranching in DeFi

Research on how tranching works on chain: what it is, the forms it takes, who is running
it, how liquidity behaves inside it, and what the live systems actually do when you work
through their economics.

This is a research repository. It does not propose a standard, ship a library, or
document a product. Where a note records a gap between traditional structured finance and
the on chain versions, it records it as a finding about the sector.

## Organisation

Every directory holds one document, numbered in reading order. Sections one to six are
general and are meant to be read in sequence. Sections seven to eleven are studies of
individual protocols and are self contained.

| | Directory | Subject |
|---|---|---|
| 01 | [`01-fundamentals/`](./01-fundamentals/README.md) | What a tranche is, the direction cash and loss travel, attachment points, coverage tests, the collateralised loan obligation as reference, and the line between fixed arithmetic and calibration |
| 02 | [`02-forms-of-tranching/`](./02-forms-of-tranching/README.md) | Risk against cash flow tranching, structures with a third leg, and why protocols describing themselves identically are often doing different things |
| 03 | [`03-protocol-landscape/`](./03-protocol-landscape/README.md) | The five parts every tranched protocol shares, and the survey of who has built what |
| 04 | [`04-liquidity/`](./04-liquidity/README.md) | Batch settlement, exit friction, junior demand, and secondary markets |
| 05 | [`05-token-standards/`](./05-token-standards/README.md) | The specifications these products are built on, and the gap between what was standardised and what is used |
| 06 | [`06-optimizations/`](./06-optimizations/README.md) | **The frontier material.** Payoff tranching against capital tranching, looping, recursive tranching, curator first loss, slashing tranches, correlation, and what is genuinely unbuilt |
| 07 | [`07-study-centrifuge/`](./07-study-centrifuge/README.md) | The largest tokenised claim on a real collateralised loan obligation waterfall |
| 08 | [`08-study-pareto/`](./08-study-pareto/README.md) | A general purpose tranching engine repointed at institutional private credit |
| 09 | [`09-study-strata/`](./09-study-strata/README.md) | One engine across several strategies, with the calibration published per market |
| 10 | [`10-study-royco/`](./10-study-royco/README.md) | Three legs, the third paid to provide an exit |
| 11 | [`11-study-re/`](./11-study-re/README.md) | Reinsurance capital and a three layer stack |

[`METHOD.md`](./METHOD.md) states how sources are handled and what the conventions mean.
[`SOURCES.md`](./SOURCES.md) is the annotated bibliography.

## Findings

Six observations recur across the whole body of work and are the reason it is worth
reading in order.

**Valuation is the trust assumption, not subordination.** Every guarantee in a tranched
structure is conditional on the number the waterfall is applied to. That number is
sometimes an oracle, sometimes an actuarial estimate from a regulated entity, and in at
least one live case a figure entered by an administrator with no oracle behind it. The
arithmetic above it can be audited seventeen times and formally verified without changing
this.

**Nothing on chain cures a breach.** Traditional structures measure a coverage ratio each
period and redirect cash to repair it. Across the sector, protocols refuse settlements,
tier exit friction, defer loss recognition, or divert income to meet a senior entitlement.
None retires senior principal to restore a ratio. The structures do not delever.

**First loss capital is thin.** Equity in a collateralised loan obligation is roughly
eight to ten percent. On chain cover positions run from zero to three percent at the low
end to around twelve percent at the measured high end. A senior claim behind a three
percent cushion is a different instrument from one behind twenty, and the documentation
rarely makes the difference visible.

**Junior demand is the binding constraint.** The protocols that failed commercially failed
for want of capital willing to stand first in line at a price the senior side would
accept. Every mechanism that protects the senior claim by restricting junior exit makes
the junior claim harder to sell, and the sector has not escaped that circle.

**The engine is always bespoke.** The token layer and the settlement layer are well
standardised. Payment priority, loss ordering and coverage testing are not standardised at
all, so every protocol writes its own and no two codebases are readily comparable.

**A protected claim can be manufactured two ways, and the sector only uses one.**
Subordination buys protection with somebody else's capital and needs a junior tranche.
Payoff decomposition buys it by giving up upside and needs nobody, because a tranche is a
spread on the loss distribution and the same exposure can be assembled from options. Every
protocol here uses the first route. The second is being built elsewhere and is not
generally recognised as tranching. See
[`06-optimizations`](./06-optimizations/README.md).

**The direction of travel is from mechanism toward distribution.** Centrifuge became the
largest entity in the sector by tokenising a claim on a traditional waterfall rather than
building one. Pareto repointed a working tranching engine at institutional private credit.
Re started from an insurance balance sheet. The scale is going where the assets already
had an institutional buyer.

## Conventions

Sources are cited inline and are preferably primary: a specification, a repository, a
protocol's own documentation, or an independent review. Figures carry the date their
source gives and are not continuously updated. Where a claim could not be verified, the
text says so rather than filling the gap, and terms with no source at all are recorded in
[`06-optimizations/`](./06-optimizations/README.md) as unverified.

There are no diagrams. Everything is prose, tables, or the formulas a protocol publishes.
Deprecated systems are covered only where a live system cannot be understood without them.
