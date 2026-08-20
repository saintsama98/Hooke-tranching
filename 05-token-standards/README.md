# Token Standards

> Reading order: reference material, readable at any point. The specifications tranched
> products are built on, what each one covers, and which of them are actually used.

The ERCs that tranched products are built on, what each one specifies, and which ones
are actually used. Header fields were retrieved from `eips.ethereum.org` on 2026-06-19.

- [timeline](.#the-chronology-shows-what-was-standardised-and-what-was-not) — Every relevant standard in order of creation, with a
  dependency ordered reading path.
- [tranche token layer](.#the-substrate-for-representing-a-tranche-as-a-token) — Representing a tranche as a
  token: ERC-20, ERC-1155, ERC-6909, ERC-3475.
- [vault and yield layer](.#the-vault-and-yield-layer-beneath-a-tranche-structure) — The settlement and yield
  wrapper base: ERC-4626, ERC-7540, ERC-7575, and the cash flow split tokens EIP-5115
  and EIP-5095.
- [compliance branch](.#the-compliance-branch-governs-who-may-hold-not-how-a-structure-is-tranched) — ERC-1400 and ERC-1410, ERC-3643 and
  ERC-7518. Who may hold a token, which is adjacent to tranching rather than part of it.
- [adoption in practice](.#adoption-in-practice-diverges-from-what-the-standards-intended) — What each surveyed protocol
  actually uses.

## The short version

The token layer and the settlement layer are well covered. ERC-20 for the tranche token,
ERC-4626 underneath it, ERC-7540 where settlement has to be asynchronous. A reader can
follow those specifications and know what they are looking at.

Nothing specifies payment priority, loss allocation order, or coverage tests. That part
of every protocol is written from scratch, which is why the codebases in
[`03-protocol-landscape`](../03-protocol-landscape/README.md) are not readily comparable and
have to be read individually.

There is also a gap between what is best suited and what is used. ERC-3475 was designed
for tranche like instruments and is barely deployed; ERC-6909 offers shared accounting
across classes and is not used for tranches. Most protocols deploy one plain ERC-20 per
tranche instead, for reasons set out in
[`04-liquidity`](../04-liquidity/README.md#secondary-markets-relocate-the-friction-rather-than-removing-it).

---

# The substrate for representing a tranche as a token

---

# False

This note surveys the candidates for representing a tranche as a token, ordered from least to most expressive.

## ERC-20 — [specification](https://eips.ethereum.org/EIPS/eip-20) (Final)

A single fungible token per tranche, with the senior and junior tokens deployed as separate contracts. ERC-20 provides no shared accounting across tranches and no concept of seniority. It is the minimal baseline, and, as the [adoption in practice](.#adoption-in-practice-diverges-from-what-the-standards-intended) shows, the representation that most surveyed protocols use in practice.

## ERC-1155 — [specification](https://eips.ethereum.org/EIPS/eip-1155) (Final)

Multiple token identifiers within one contract, allowing the senior tranche to occupy identifier 0 and the junior tranche identifier 1. ERC-1155 establishes the multiple classes in one contract pattern but carries no bond or redemption semantics, and it mandates receiver callbacks and batch operations.

## ERC-6909 — [specification](https://eips.ethereum.org/EIPS/eip-6909) (Final; interface identifier `0x0f632fb3`)

A minimal multi token interface that retains the `balanceOf(owner, id)` model of ERC-1155 while removing its heavier requirements:

- No mandatory receiver callbacks, reducing reentrancy surface and permitting unaware counterparties.
- Both granular per identifier allowances and a blanket operator authorisation.
- Batch operations omitted from the core interface.

ERC-6909 is used in production by Uniswap v4. In effect it is an ERC-20 interface extended with an identifier argument.

## ERC-3475 — [specification](https://eips.ethereum.org/EIPS/eip-3475) (Final; requires ERC-20, ERC-721, ERC-1155)

The only standard designed specifically to represent tranche like instruments, by means of a class and nonce model:

- `classId` denotes a class of instrument, which maps onto a tranche (senior, mezzanine, junior).
- `nonceId` denotes an issuance series within a class, each with its own supply, metadata, and net asset value.
- A balance is addressed as `balanceOf(account, classId, nonceId)`.

ERC-3475 specifies tranche identity, on chain per class and per nonce metadata (`Values` and `Metadata`), a supply lifecycle (`activeSupply`, `redeemedSupply`, `burnedSupply`), and a redemption condition accessor (`getProgress`).

It does not specify a seniority ordering between classes, and it does not specify a waterfall: redemption conditions and payment priority are left to the implementing contract.

## How they compare

ERC-3475 is the most complete representation of a tranche and also the most complex, and it has minimal production adoption. ERC-6909 is lighter and more widely adopted, at the cost of leaving the tranche semantics to the implementer. In practice most protocols use neither and deploy a plain ERC-20 per tranche; see [adoption in practice](.#adoption-in-practice-diverges-from-what-the-standards-intended).

ERC-721 also appears in production for the dated or nonfungible leg of a structure — the BarnBridge senior bonds (sBONDs) and the Goldfinch junior position tokens (PoolTokens); see [`03-protocol-landscape`](../03-protocol-landscape/README.md#risk-tranching-protocols).

---

# The vault and yield layer beneath a tranche structure

These standards specify the settlement rails and yield wrappers beneath a tranche structure. Each defines a single share class and provides no concept of seniority or splitting.

## ERC-4626 — [specification](https://eips.ethereum.org/EIPS/eip-4626) (Final; requires ERC-20, ERC-2612)

A tokenized vault interface comprising `deposit`, `mint`, `withdraw`, and `redeem`, together with `convertTo*`, `preview*`, and `max*` views. ERC-4626 defines a single share class and is pari passu by construction. A common senior and junior implementation deploys two ERC-4626 vaults over one strategy, as in the Idle Perpetual Yield Tranches. The `totalAssets()` and `convertToAssets()` functions are the points at which a waterfall would override the proportional net asset value split. Reference implementations: [Solmate](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC4626.sol) and OpenZeppelin.

## ERC-7540 — [specification](https://eips.ethereum.org/EIPS/eip-7540) (Final; requires ERC-20, ERC-165, ERC-4626, ERC-7575)

An asynchronous extension of ERC-4626 providing `requestDeposit` and `requestRedeem` followed by later fulfilment. The request to fulfilment boundary corresponds to an epoch boundary, at which a waterfall would compute the per tranche split before fulfilling claims. ERC-7540 is used by Centrifuge, Ondo, and Backed for real world asset settlement.

## ERC-7575 — [specification](https://eips.ethereum.org/EIPS/eip-7575) (Final; vault interface `0x2f0a18c5`, share interface `0xf815c03d`)

Externalises the share token from the vault through a `share()` accessor, allowing multiple asset entry points to share one token (with optional "Pipes" for conversion). This is the inverse of a tranche structure: it maps many assets to one share, whereas a tranche maps one asset to many share classes. It confirms that the vault standards do not address seniority. Note the [timeline](.#the-chronology-shows-what-was-standardised-and-what-was-not): ERC-7540 requires ERC-7575 although ERC-7540 is the earlier document.

## Cash flow split tokens

The following standards represent the output of a yield split (principal versus yield) rather than a risk tranche.

### EIP-5115 — SY / Standardized Yield — [specification](https://eips.ethereum.org/EIPS/eip-5115) (Draft; requires ERC-20)

A yield wrapper interface more general than ERC-4626, authored by the Pendle team. It relaxes three ERC-4626 assumptions: it permits multiple input and output tokens (`getTokensIn`, `getTokensOut`), it permits accounting in units that are not themselves depositable (for example AMM liquidity units), and it provides for reward token handling. Its model is a generic yield generating pool with `exchangeRate() = totalAssets / totalShares`. Pendle wraps an arbitrary yield source as SY and then splits it into principal and yield tokens. The architectural consequence is that a uniform yield source adapter beneath the engine allows the engine to reason about a single normalised net asset value irrespective of source.

### EIP-5095 — Principal Token — [specification](https://eips.ethereum.org/EIPS/eip-5095) (Stagnant; requires ERC-20, ERC-2612)

Specifies the principal token leg only — ownership of an underlying ERC-20 at a future `maturity()`, redeemable at that maturity — using an ERC-4626-shaped redemption interface. It does not specify the yield token or the act of splitting. The proposal is Stagnant, and Pendle's principal tokens implement ERC-4626 rather than EIP-5095 ([discussion](https://ethereum-magicians.org/t/eip-5095-principal-token-standard/9259)).

## The pattern across both layers

The same thing happened on both kinds of split. The token legs received standards, partial or stalled ones in the cash flow case, while the engine that does the splitting received none. See [`02-forms-of-tranching`](../02-forms-of-tranching/README.md) for the two kinds side by side, and [`03-protocol-landscape`](../03-protocol-landscape/README.md#cash-flow-tranching-protocols) for who builds on these.

---

# The compliance branch governs who may hold, not how a structure is tranched

These standards govern eligibility to hold a token — identity, investor qualification, and transfer restrictions. They are adjacent to tranching rather than part of it: they compose alongside a waterfall but do not specify one. They are relevant only where the target is a regulated or real world asset structure.

## ERC-1400 and ERC-1410 — unofficial

These are not official EIPs. They were opened as GitHub issues by Adam Dossa of Polymath in September 2018: [issue #1411 (ERC-1400)](https://github.com/ethereum/EIPs/issues/1411) and [issue #1410 (ERC-1410)](https://github.com/ethereum/EIPs/issues/1410). Both are closed and labelled stale. The canonical specification is maintained in a community repository rather than in the Ethereum Foundation repository: [`SecurityTokenStandard/EIP-Spec`](https://github.com/SecurityTokenStandard/EIP-Spec).

ERC-1400 is an umbrella over five component standards: ERC-1594 (core transfer), ERC-1410 (partitions), ERC-1643 (document management), and ERC-1644 (controller operations). The tranche relevant element is ERC-1410's partition model: balances are divided into `bytes32` partitions, and `canTransferByPartition` returns a status reason code. This corresponds to a tranche as partition representation.

## ERC-3643 — T-REX — [specification](https://eips.ethereum.org/EIPS/eip-3643) (Final, 2021-07-09; requires ERC-20, ERC-173)

The finalised and most widely adopted member of this branch. It specifies permissioned, compliant transfer with on chain identity (ONCHAINID) for investor eligibility. The specification does not address tranching, seniority, or waterfall logic; it governs which parties may hold tranche tokens, not how a structure is tranched.

## ERC-7518 — DyCIST — [specification](https://eips.ethereum.org/EIPS/eip-7518) (Review, 2023-09-14; requires ERC-165, ERC-1155)

A more recent successor to the ERC-1400/1410 line, built over ERC-1155 partitions with off chain compliance vouchers, token locking, and forced transfers.

## Summary

The ERC-1410 partition model is useful as prior art rather than as a citable standard. Of the finalised options, ERC-3643 is the one in use, and ERC-7518 is the proposal to track. None of them says anything about payment priority or loss ordering; they govern who may hold a tranche token, not how the structure is tranched.

---

# The chronology shows what was standardised and what was not

The standards relevant to tranching, ordered by creation date. Header fields verified on 2026-06-19; current status is shown in the final column.

| Created | ERC | Title | Status | Role |
|---|---|---|---|---|
| 2015-11-19 | [ERC-20](https://eips.ethereum.org/EIPS/eip-20) | Token Standard | Final | Baseline; one ERC-20 per tranche |
| 2018-06-17 | [ERC-1155](https://eips.ethereum.org/EIPS/eip-1155) | Multi Token Standard | Final | Multiple token identifiers per contract |
| 2018-09-09 | [ERC-1400](https://github.com/ethereum/EIPs/issues/1411) | Security Token Standard (umbrella) | Issue; unofficial | Compliance umbrella |
| 2018-09-13 | [ERC-1410](https://github.com/SecurityTokenStandard/EIP-Spec/blob/master/eip/eip-1410.md) | Partially Fungible Token (partitions) | Issue; unofficial | Tranche as partition model |
| 2021-04-05 | [ERC-3475](https://eips.ethereum.org/EIPS/eip-3475) | Abstract Storage Bonds | Final | Purpose built tranche token substrate |
| 2021-07-09 | [ERC-3643](https://eips.ethereum.org/EIPS/eip-3643) | T-REX (regulated tokens) | Final | Compliance and identity |
| 2021-12-22 | [ERC-4626](https://eips.ethereum.org/EIPS/eip-4626) | Tokenized Vaults | Final | Vault settlement base |
| 2022-05-01 | [EIP-5095](https://eips.ethereum.org/EIPS/eip-5095) | Principal Token | Stagnant | Principal leg of a cash flow split |
| 2022-05-30 | [EIP-5115](https://eips.ethereum.org/EIPS/eip-5115) | SY / Standardized Yield | Draft | Yield source wrapper |
| 2023-04-19 | [ERC-6909](https://eips.ethereum.org/EIPS/eip-6909) | Minimal Multi Token | Final | Lightweight multi token interface |
| 2023-06-20 | [ERC-7201](https://eips.ethereum.org/EIPS/eip-7201) | Namespaced Storage Layout | Final | Upgrade safe storage substrate |
| 2023-09-14 | [ERC-7518](https://eips.ethereum.org/EIPS/eip-7518) | DyCIST | Review | Security token over ERC-1155 partitions |
| 2023-10-18 | [ERC-7540](https://eips.ethereum.org/EIPS/eip-7540) | Asynchronous Vaults | Final | Epoch and request based settlement |
| 2023-12-11 | [ERC-7575](https://eips.ethereum.org/EIPS/eip-7575) | Multi Asset Vaults | Final | Externalised share token |

## Two observations from the chronology

1. **Dependency inversion.** ERC-7540 (created 2023-10-18) depends on ERC-7575 (created 2023-12-11); ERC-7575 was separated from ERC-7540 after the fact. ERC-7575 is therefore a prerequisite for understanding ERC-7540 despite being the more recent document.

2. **There is no waterfall standard.** The chronology contains token standards, vault standards and compliance standards, and nothing that specifies payment priority, loss allocation order, or coverage tests. Anyone reading a tranched protocol should expect that part to be written from scratch.

## Reading order

A dependency ordered path is more instructive than strict chronology. Token substrate, then settlement base, then the cash flow tokens, then storage and compliance:

`ERC-20 → ERC-1155 / ERC-6909 → ERC-3475` · `ERC-4626 → ERC-7540 → ERC-7575` · `EIP-5115 / EIP-5095 (cash-flow axis)` · `ERC-7201` · `ERC-3643 → ERC-7518 → ERC-1410 (compliance, where applicable)`

Further detail: [tranche token layer](.#the-substrate-for-representing-a-tranche-as-a-token), [vault and yield layer](.#the-vault-and-yield-layer-beneath-a-tranche-structure), [compliance branch](.#the-compliance-branch-governs-who-may-hold-not-how-a-structure-is-tranched).

---

# Adoption in practice diverges from what the standards intended

The standards each surveyed protocol actually uses, drawn from the protocol notes in
[`03-protocol-landscape`](../03-protocol-landscape/README.md) and the close readings in
[`03-protocol-landscape`](../03-protocol-landscape/README.md).

| Protocol | Standards adopted | Note |
|---|---|---|
| Idle PYT | ERC-20 for AA and BB, with ERC-4626 | Two separate ERC-20 tokens over ERC-4626 accounting |
| BarnBridge | ERC-20 for jTokens, ERC-721 for sBONDs | Senior claim issued as a dated nonfungible bond |
| Centrifuge | ERC-20 for DROP and TIN; ERC-7540 in version 3 | The only ERC-7540 adopter, which it coauthored |
| Goldfinch | ERC-20 for FIDU, ERC-721 for PoolTokens | Junior position issued as nonfungible |
| Maple | ERC-20 and ERC-4626 | Standard vault shares |
| Strata | ERC-4626 with LayerZero transferability | Cooldown handled by purpose built contracts, not ERC-7540 |
| Royco | ERC-4626 style vaults for both tranches | Third leg is an AMM position rather than a tranche token |
| Re | ERC-20 for reUSD and reUSDe | Yield accrues into the price rather than rebasing |
| Pendle | ERC-20 for PT and YT, EIP-5115 for SY | Principal token implements ERC-4626, not EIP-5095 |
| Saffron, Waterfall DeFi | ERC-20 | Plain fungible tokens |
| Untangled | ERC-20 for SOT and JOT, unverified | Likely plain ERC-20 |
| Galaxy, Tradable | Unverified, likely ERC-3643 or ERC-1400 | Permissioned real world asset wrappers |

Five things fall out of that table. ERC-20 predominates, with each tranche generally
deployed as its own contract and no shared accounting between them, which is the minimal
possible representation of a structure. ERC-3475, the one standard designed specifically
to express tranche like instruments, has essentially no production adoption, and
protocols reconstruct an inferior version of it with pairs of ERC-20 tokens instead.
ERC-1155 and ERC-6909, which would give several classes shared accounting inside one
contract, are not used for tranches at all. ERC-4626 is the most widely adopted
settlement base and ERC-7540, the asynchronous extension written for exactly this
problem, has a single adopter in the protocol that coauthored it. And two standards from
outside the obvious sequence keep appearing: ERC-721 wherever a leg carries terms
specific to its holder, and EIP-5115 as the yield wrapper beneath a cash flow split.

The implication for a reader is practical rather than editorial. The standards best
suited to tranching are largely unused in that role, and the prevailing practice is
ERC-20 plus ERC-4626 with an engine written from scratch for each protocol. Anyone
approaching one of these codebases should expect the tranching logic to be bespoke and
should not expect a shared interface through which to compare two of them. The reason
the industry settled on the worse representation is liquidity rather than ignorance, and
it is set out in
[`04-liquidity`](../04-liquidity/README.md#secondary-markets-relocate-the-friction-rather-than-removing-it).
