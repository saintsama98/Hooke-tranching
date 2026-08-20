# Contributing

This is an open research repository on tranching. Corrections, better sources and new
material are welcome, and anyone is welcome to comment on an issue or pick one up. The
standards applied to submissions are set out in [`METHOD.md`](./METHOD.md) and are worth
reading before opening a pull request.

## Ways to contribute

Open an issue to report an error, question a claim, or propose a protocol worth covering.
Issues are the open work thread here. Open a pull request for a correction, an addition to
an existing document, or a new protocol study.

## What goes where

Every directory holds exactly one document. Additions extend the relevant document rather
than adding a file beside it.

| Subject | Document |
|---|---|
| How tranching works in general | [`01-fundamentals`](./01-fundamentals/README.md) |
| The kinds of split and how they differ | [`02-forms-of-tranching`](./02-forms-of-tranching/README.md) |
| The shared shape, and the protocol survey | [`03-protocol-landscape`](./03-protocol-landscape/README.md) |
| Entry, exit, gating, secondary markets | [`04-liquidity`](./04-liquidity/README.md) |
| Specifications and their adoption | [`05-token-standards`](./05-token-standards/README.md) |
| Optimizations alongside tranching, and unverified terms | [`06-optimizations`](./06-optimizations/README.md) |
| A close reading of one protocol | A numbered study directory |
| A source worth recording | [`SOURCES.md`](./SOURCES.md) |

A new protocol study is a new numbered directory containing one `README.md`, added to the
table in the root [`README.md`](./README.md).

## Standards for material

Cite a primary source where one exists, at the point the claim is made. Prefer a
protocol's technical documentation, a public repository, or an independent review over a
secondary write up.

Say when something is unverified. Several passages here carry an explicit note that a
figure or a token standard could not be confirmed, and that is strongly preferred to a
confident guess. Terms with no source at all belong in
[`06-optimizations`](./06-optimizations/README.md).

Describe live systems. Where a protocol has replaced the architecture it became known
for, the current one is the subject and the earlier one appears only where the current one
cannot be understood without it.

Leave implementation detail to the protocols. Contract inventories, addresses and source
excerpts are not reproduced here. Where a design decision has an economic consequence,
record the consequence.

Write in stacked paragraphs with headings that are statements. No diagrams.

## Licensing

Contributions are licensed under the dual scheme of this repository: CC-BY-4.0 for prose
and MIT for any code. See [`LICENSE`](./LICENSE) and [`LICENSE-SPEC`](./LICENSE-SPEC).
