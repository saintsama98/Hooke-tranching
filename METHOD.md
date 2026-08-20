# Method

## Scope

The subject is tranching: the division of claims on a single pool into an order of payment
and an order of loss. Work is in scope when it bears on how that order is defined,
measured, priced, enforced or exited. Adjacent subjects are covered only where a tranched
structure cannot be understood without them, and are held in
[`06-optimizations/`](./06-optimizations/README.md) rather than mixed into the
protocol material.

Implementation detail is out of scope. Contract inventories, deployment addresses, source
excerpts and integration guides belong to a protocol's own documentation and are not
reproduced here. Where a design decision has an economic consequence, the consequence is
recorded and the mechanism that produces it is described in words.

Deprecated systems are out of scope except where a live one cannot be read without them.
The sector moves quickly and several protocols in this repository have already replaced
the architecture they became known for. The default is to describe what is running.

## Sourcing

Every factual claim carries a source, cited inline at the point it is made rather than
collected at the end. Sources are preferred in this order: a protocol's own technical
documentation, a public repository, an independent review by a party with no position, a
specification, then reported analysis. Secondary commentary is used for context and is
identified as such.

Figures are recorded with the date their source gives. Nothing here is continuously
updated, and a total value locked figure or a yield in this repository should be read as a
measurement taken at a moment rather than as a current quotation.

Where two sources disagree, both are given and the disagreement is stated rather than
resolved silently. Where a claim could not be verified against any source, it is either
omitted or marked explicitly as unverified. A negative result that nobody writes down
gets researched again by the next reader, so the failures to verify are recorded
deliberately.

Protocols are named only where there is something to read: a live deployment, a public
repository or published technical documentation, and preferably an audit or an outside
review. Funding announcements and press coverage are not evidence that a mechanism exists.

## Register

Headings are statements rather than questions, so that reading the headings alone gives
the argument.

Prose is written in stacked paragraphs. Lists are used where the content is genuinely a
list, meaning a bibliography, a set of parameters, or a comparison across protocols, and
not as a substitute for making an argument.

Language is plain. Terms of art are used where they are precise and are defined at first
use in [`01-fundamentals/`](./01-fundamentals/README.md). Where a common phrase is
imprecise, the more exact formulation is preferred even when it is longer.

There are no diagrams. A waterfall is an ordering and an ordering is expressible in
sentences and tables, and a picture of one tends to flatter the design by making a
sequence look like a structure.

## Confidence

Three levels are distinguished in the text and it is worth knowing which is which.

Something described as published or documented is taken from a protocol's own material
and is a claim about what the protocol says. Something described as measured or reported
comes from an independent party and is a claim about what somebody found. Something
described as derived is arithmetic performed here on published figures, and where that
happens the derivation is shown so it can be checked.

Judgements are stated as judgements. Where this repository concludes that a mechanism is
weaker than its documentation suggests, or that two protocols are not comparable, the
reasoning is given and a reader is expected to be able to disagree with it.
