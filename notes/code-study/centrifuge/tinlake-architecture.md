# Tinlake Architecture

Tinlake ([`centrifuge/tinlake`](https://github.com/centrifuge/tinlake)) follows the MakerDAO "dss" style: many small single-purpose contracts wired together at deployment, with privileged calls gated by an `auth` allowlist. A pool comprises a borrower side (loans and a NAV feed) and a lender side (the tranches). The tranche mechanism lives in `src/lender/`.

Pool value is defined as **NAV + Reserve**: the off-chain-priced value of the real-world loans plus the on-chain liquid `currency` held in the reserve.

## The lender contracts (`src/lender/`)

| Contract | Role |
|---|---|
| `token/restricted.sol` + `token/memberlist.sol` | The DROP (senior) and TIN (junior) ERC-20 tokens, with a memberlist gating transfers to permitted (KYC-ed) addresses. |
| `operator.sol` | Investor entry point: submit `supplyOrder`/`redeemOrder`, and `disburse` fulfilled amounts after an epoch. |
| `tranche.sol` | Per-tranche unit: aggregates supply/redeem orders, holds the tranche's currency between epochs, and mints/burns the tranche token on execution. Deployed once per tranche, so **twice per pool**. |
| `reserve.sol` | The liquid `currency` buffer; the on-chain half of pool value. |
| `assessor.sol` | Valuation and coverage: the senior ratio, its bounds, senior debt accrual, and the senior/junior token prices. See [`assessor-and-coordinator.md`](./assessor-and-coordinator.md). |
| `coordinator.sol` | The epoch solver that executes orders subject to the coverage constraints. See [`assessor-and-coordinator.md`](./assessor-and-coordinator.md). |
| `definitions.sol` | An abstract base holding the shared pure calculation helpers (`calcSeniorRatio`, `calcAssets`, the token-price calcs). |
| `deployer.sol`, `fabs/` | The factory layer (see below). |

## The shared base and dependency layer

Tinlake distributes its functionality across base contracts and external packages, so much of what a file uses is inherited rather than declared locally. For example:

```solidity
contract Assessor is Definitions, Auth, Interest { ... }
abstract contract Definitions is FixedPoint, Math { ... }
```

- **`Definitions`** (`src/lender/definitions.sol`) — the pure calc helpers (`calcSeniorRatio`, `calcExpectedSeniorAsset`, `calcAssets`, token prices), inherited by both the Assessor and the Coordinator so the two compute identical values.
- **`Auth`** (`lib/tinlake-auth/src/auth.sol`) — the access-control layer: a `mapping(address => uint256) wards` (1 = authorised), with `rely`/`deny` to grant/revoke and an `auth` modifier gating every privileged function (the parameter setters, `depend`, `file`).
- **`Interest`** (`lib/tinlake-math/src/interest.sol`) — ray-based interest math (`chargeInterest`, `rpow` per-second compounding); the basis of senior debt accrual.
- **`FixedPoint`** (`src/fixed_point.sol`) — defines `Fixed27`, a struct wrapping a `uint256` interpreted as a 27-decimal fixed-point number (a "ray", where `1e27 = 1.0 = 100%`). Used for all ratios and rates.
- **`Math`** (`lib/tinlake-math/src/math.sol`) — `safeAdd`/`safeSub`/`safeMul` (manual overflow checks, since the code targets Solidity < 0.8) and the ray/wad operators `rmul`/`rdiv`.

External packages (`tinlake-math`, `tinlake-auth`, `tinlake-erc20`, `tinlake-title`) are git submodules under `lib/`, resolved through import remappings (for example `tinlake-math/ = lib/tinlake-math/src/`). They are present only after `git submodule update --init --recursive`.

## Dependency injection: the `depend` pattern

A contract declares a typed, `public`, initially-unset state variable for each neighbour it calls, for example `CoordinatorLike public coordinator;` or `ERC20Like public currency;`. The address is injected at deployment by an `auth`-gated setter:

```solidity
function depend(bytes32 what, address addr) public auth {
    if (what == "coordinator") coordinator = CoordinatorLike(addr);
    // ...
}
```

The `...Like` interfaces (`ERC20Like`, `CoordinatorLike`, `AssessorLike`) are minimal local declarations listing only the methods that contract calls on its neighbour; the implementations live in the real contracts at the injected addresses.

## The per-pool deployment model

Tinlake deploys an **entire lender stack per pool** — two tranches, an assessor, a coordinator, a reserve — isolated to that one pool, so that one pool cannot affect another. This is what the factory layer is for: `src/lender/fabs/` holds a fabricator per component (`fabs/tranche.sol`, `fabs/assessor.sol`, …), and `src/lender/deployer.sol` orchestrates a deployment, calling the tranche fabricator **twice** (senior and junior) and wiring every contract together with `rely`/`depend`. The set of `rely` calls in `deployer.sol` defines the trust graph (for example, the coordinator is made a ward of the assessor, tranches, and reserve so it can drive them at epoch close).

This per-tranche-as-its-own-contract layout, with a shared coordinator and assessor above, is the opposite of Idle's single-engine layout; the design fork it implies for the proposed standard is discussed in [`mapping-to-standard.md`](./mapping-to-standard.md) and [`../../synthesis/substrate-fork.md`](../../synthesis/substrate-fork.md).
