---
eip: <to be assigned>
title: Waterfall Tranching Standard
description: A generic interface for waterfall-based tranching of cash flows on EVM smart contracts.
author: TBD (@saintsama98)
discussions-to: <Ethereum Magicians topic URL — to be added>
status: Draft
type: Standards Track
category: ERC
created: 2026-06-18
requires: 20, 165
---

## Abstract

This standard defines an interface for distributing cash flow and allocating loss across an ordered set of tranches according to a priority of payments, or "waterfall". A conforming contract arranges tranches in a strictly decreasing order of seniority, distributes incoming cash flow to senior tranches before junior tranches, allocates loss to junior tranches before senior tranches, and evaluates coverage tests whose breach re-routes cash flow from junior tranches toward the repayment of senior obligations. The standard specifies the distribution and loss-allocation behaviour and the coverage-test interface; it does not specify how cash flow or loss is recognised, nor how tranche ownership is tokenised, both of which are delegated to composed standards.

## Motivation

Existing standards address the representation of tranche tokens (ERC-20, ERC-1155, ERC-6909, ERC-3475) and the vault and settlement layers on which tranche products are built (ERC-4626, ERC-7540, ERC-7575). None specifies the logic that allocates cash flow and loss between tranches. As a result, every protocol that offers senior and junior exposure implements its own bespoke engine, with no shared interface for distribution, loss allocation, or coverage testing, and no common vocabulary against which such systems can be composed, audited, or compared.

The defining mechanic of traditional securitisation is the coverage test: a periodic measurement of overcollateralisation or interest coverage whose breach actively re-routes cash flow, diverting distributions that would otherwise reach junior holders toward the repayment of senior principal until the measure is restored. A survey of deployed protocols finds this mechanic absent: existing systems either absorb loss passively, by reducing the net asset value of the junior tranche, or, in a single case, suspend new senior issuance on breach without redirecting cash flow. This standard specifies that mechanic as a composable interface.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in RFC 2119 and RFC 8174.

### Definitions

- **Tranche** — A class of claim on the cash flow and loss of a single underlying pool, identified by an integer `trancheId`.
- **Seniority** — The priority of a tranche. Tranches are indexed `0` to `trancheCount() - 1`. Index `0` is the most senior; seniority is strictly decreasing with index. A tranche with a lower index is senior to a tranche with a higher index.
- **Distribution** — The routing of an inflow of the underlying asset to tranches in order of seniority.
- **Loss allocation** — The reduction of tranche claims to reflect a realised loss, in reverse order of seniority.
- **Coverage test** — A measured ratio compared against a configured trigger, the breach of which modifies subsequent distribution.

### Asset and topology

A conforming contract MUST manage a single underlying asset that conforms to ERC-20.

```solidity
function asset() external view returns (address assetTokenAddress);
function trancheCount() external view returns (uint256 count);
```

`trancheCount()` MUST return the number of tranches. The ordering of `trancheId` from `0` to `trancheCount() - 1` MUST be strictly decreasing in seniority and MUST be stable for the lifetime of the contract.

A conforming contract MAY expose the token that represents ownership of each tranche. Where it does, the token SHOULD conform to ERC-20, ERC-6909, or ERC-3475. The representation of tranche ownership is otherwise outside the scope of this standard.

```solidity
function trancheToken(uint256 trancheId) external view returns (address token);
```

### Accounting

```solidity
function totalAssets() external view returns (uint256 assets);
function trancheAssets(uint256 trancheId) external view returns (uint256 assets);
function trancheTarget(uint256 trancheId) external view returns (uint256 target);
```

`totalAssets()` MUST return the total value of the underlying pool, denominated in the underlying asset. `trancheAssets(trancheId)` MUST return the value currently claimable by the given tranche. The sum of `trancheAssets` over all tranches MUST NOT exceed `totalAssets()`. `trancheTarget(trancheId)` MUST return the obligation against which the tranche's coverage is measured, such as its outstanding principal plus accrued and unpaid interest.

The recognition of `totalAssets()`, including the recognition and quantification of loss, is delegated to the composed accounting or oracle layer and is outside the scope of this standard. See Security Considerations.

### Coverage tests

```solidity
enum CoverageTestType { Overcollateralization, InterestCoverage }

function coverageRatio(uint256 trancheId, CoverageTestType testType) external view returns (uint256 ratio);
function coverageTrigger(uint256 trancheId, CoverageTestType testType) external view returns (uint256 trigger);
function isBreached(uint256 trancheId, CoverageTestType testType) external view returns (bool breached);
```

Ratios and triggers are expressed as fixed-point numbers with 18 decimals, where `1e18` represents a ratio of one.

`coverageRatio(trancheId, Overcollateralization)` MUST return the ratio of the value of the pool to the aggregate obligation of the given tranche and all tranches senior to it. `coverageRatio(trancheId, InterestCoverage)` MUST return the ratio of interest collected over the current period to the interest due to the given tranche and all tranches senior to it.

`isBreached(trancheId, testType)` MUST return `true` if and only if `coverageRatio(trancheId, testType)` is less than `coverageTrigger(trancheId, testType)`.

A conforming contract MAY support only a subset of the test types for a given tranche. Where a test type is not applied to a tranche, `coverageTrigger` for that pair MUST return `0`, and `isBreached` MUST return `false`.

Triggers are calibration parameters. A conforming contract SHOULD permit triggers to be configured by a governing authority, and MUST emit `CoverageTriggerUpdated` on each change.

```solidity
function setCoverageTrigger(uint256 trancheId, CoverageTestType testType, uint256 trigger) external;
```

### Distribution

```solidity
enum DistributionKind { Interest, Principal }

function distribute(uint256 amount, DistributionKind kind) external;
```

`distribute(amount, kind)` routes `amount` of the underlying asset across the tranches. A conforming contract MUST observe the following rules.

1. Tranches MUST be served in order of seniority, beginning at `trancheId` `0`.
2. Before any amount is distributed to a tranche, every coverage test protecting a tranche senior to it MUST be evaluated.
3. If a coverage test protecting tranche `k` is breached, any amount that would otherwise be distributed to a tranche junior to `k` MUST instead be applied to reduce the obligations of tranche `k` and the tranches senior to it, in order of seniority, until the breached test is no longer breached.
4. An amount redirected under rule 3 MUST be reported by a `Diversion` event.
5. Each amount delivered to a tranche MUST be reported by a `Distribution` event.

A conforming contract MAY separate the interest and principal waterfalls, applying distinct ordering or coverage rules to each `DistributionKind`.

### Loss allocation

```solidity
function allocateLoss(uint256 amount) external;
```

`allocateLoss(amount)` reduces tranche claims to reflect a realised loss of `amount`. A conforming contract MUST observe the following rules.

1. Loss MUST be allocated in reverse order of seniority, beginning at the most junior tranche, `trancheId` `trancheCount() - 1`.
2. A tranche MUST be reduced to zero before any loss is allocated to the next more senior tranche.
3. Each reduction MUST be reported by a `LossAllocated` event.

### Events

```solidity
event Distribution(uint256 indexed trancheId, uint256 amount, DistributionKind kind);
event Diversion(uint256 indexed fromTrancheId, uint256 indexed toTrancheId, uint256 amount, CoverageTestType testType);
event LossAllocated(uint256 indexed trancheId, uint256 amount);
event CoverageBreach(uint256 indexed trancheId, CoverageTestType testType, uint256 ratio, uint256 trigger);
event CoverageCured(uint256 indexed trancheId, CoverageTestType testType, uint256 ratio, uint256 trigger);
event CoverageTriggerUpdated(uint256 indexed trancheId, CoverageTestType testType, uint256 previousTrigger, uint256 newTrigger);
```

A conforming contract MUST emit `CoverageBreach` when a coverage test transitions from not breached to breached, and `CoverageCured` when it transitions from breached to not breached.

### Interface detection

A conforming contract MUST implement ERC-165. It MUST return `true` for the interface identifier of the interface defined in this standard.

## Rationale

**An ordered integer index for seniority.** Seniority is the single property on which the entire mechanism depends, yet neither ERC-3475 nor ERC-6909 provides it. Representing it as a strictly ordered integer index makes the priority of every operation unambiguous and inexpensive to evaluate, and admits an arbitrary number of tranches rather than the two that deployed protocols generally hardcode.

**Separation of distribution from recognition.** The standard specifies how cash flow and loss are *distributed and allocated*, and deliberately does not specify how they are *recognised*. Loss recognition depends on the nature of the underlying assets and on an external attestation of value, and is the principal trust assumption of any tranche system. Holding it outside the interface keeps the standard asset-agnostic and confines the standard's normative claims to behaviour that can be verified on-chain.

**Coverage tests as a first-class element.** The active re-routing of cash flow on a coverage breach is the mechanic that distinguishes a managed waterfall from passive loss absorption, and it is the element absent from deployed protocols. It is therefore specified as normative behaviour rather than left to implementations.

**Triggers as governable parameters.** A coverage trigger is a calibration judgement, not a determinable constant. The standard specifies the deterministic response to a breach while exposing the trigger as an explicit, auditable parameter, so that the calibration of a deployment is inspectable rather than embedded.

**Composition rather than inheritance.** Settlement (ERC-4626, ERC-7540) and tranche tokenisation (ERC-20, ERC-6909, ERC-3475) are addressed by existing standards. This standard composes with them rather than restating them.

## Backwards Compatibility

This standard introduces a new interface and does not modify any existing one. It composes with ERC-20 as the underlying asset, with ERC-4626 and ERC-7540 as settlement layers, and with ERC-20, ERC-6909, or ERC-3475 as the tranche-token representation. Interface detection follows ERC-165.

## Reference Implementation

See [`../reference/`](../reference/).

## Security Considerations

**Loss recognition and net-asset-value attestation.** The behaviour specified here is only as sound as the `totalAssets()` value it acts upon. An incorrect or manipulated valuation misallocates both distribution and loss. Implementations that derive value from off-chain assets SHOULD document their attestation mechanism and SHOULD provide a dispute or delay mechanism commensurate with the trust placed in the attesting party.

**Procyclicality and redemption runs.** Coverage-driven diversion reduces junior distributions precisely when conditions deteriorate, which may accelerate junior withdrawal and compound stress. Implementations SHOULD consider gating, notice periods, or rate limits on redemption.

**Adequacy of first-loss capital.** The protection of senior tranches depends on the junior tranche being large enough to absorb plausible loss. Implementations SHOULD make the subordination level explicit and SHOULD consider a minimum subordination floor.

**Parameter governance.** Coverage triggers, and any governance over tranche configuration, are powerful controls. Their administration SHOULD be constrained and transparent, and changes are required to emit events.

**Ordering and precision.** Distribution and loss allocation traverse tranches in a defined order; implementations MUST guard against reentrancy across that traversal and SHOULD define rounding such that the sum of tranche claims cannot exceed `totalAssets()`.

## Copyright

Spec text is licensed under CC-BY-4.0 (see [`../LICENSE-SPEC`](../LICENSE-SPEC)).
