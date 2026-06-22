# Contract Map — What Is Tranching, What Is Not

This note isolates the contracts relevant to the tranche mechanism in each generation, and gives a reading order.

## Tinlake — for the mechanism

Tranche-relevant, all under `src/lender/`:

| Contract | Role |
|---|---|
| `definitions.sol` | Shared pure calc helpers (`calcSeniorRatio`, `calcAssets`, token prices), inherited by the assessor and coordinator. |
| `token/memberlist.sol`, `token/restricted.sol` | The DROP/TIN tranche tokens and their transfer allowlist. |
| `operator.sol` | Investor entry: supply/redeem orders and disbursement. |
| `tranche.sol` | Per-tranche order aggregation, currency custody, and mint/burn. |
| `reserve.sol` | The liquid currency buffer (the on-chain half of pool value). |
| `assessor.sol` | Valuation, senior ratio, bounds, senior debt accrual, token prices. |
| `coordinator.sol` | The epoch solver enforcing the coverage constraints. |

Supporting (read as needed): `src/fixed_point.sol` (`Fixed27`), `lib/tinlake-math/src/{math,interest}.sol` (ray math and interest), `lib/tinlake-auth/src/auth.sol` (`wards`/`auth`).

Out of scope for the mechanism: the borrower side and the NAV feed (the asset side that supplies `nav` to the assessor); `deployer.sol` and `fabs/*` (deployment); `admin/*` (governance setters); `adapters/mkr/*` (a MakerDAO DAI credit line).

### Reading order (Tinlake)
`definitions.sol` → `token/memberlist.sol` + `token/restricted.sol` → `operator.sol` → `tranche.sol` → `reserve.sol` → `assessor.sol` → `coordinator.sol`. The mechanism, in that order: investors submit orders through the operator, stored per `tranche`; at epoch close the `coordinator` reads NAV and `reserve`, asks the `assessor` for prices and senior-ratio bounds, selects the order fulfilment that respects subordination, and calls back into each `tranche` to mint or burn.

## Liquidity Pools v3 — for the settlement interface

Tranche-relevant, under `src/`: `ERC7540Vault.sol`, `InvestmentManager.sol`, `PoolManager.sol`, `token/Tranche.sol`, `token/RestrictionManager.sol`, `Escrow.sol`, `factories/{ERC7540VaultFactory,TrancheFactory}.sol`.

Out of scope for tranching: `gateway/*` and `adapters/*` (cross-chain messaging), `libraries/*`, `Root.sol`/`Auth.sol`/`admin/Guardian.sol` (access control), `CentrifugeRouter.sol` (router). Note that the coverage and pricing logic is not in these contracts at all — it runs off-chain (see [`liquidity-pools-v3.md`](./liquidity-pools-v3.md)).

## Recommendation

Study **Tinlake** for the mechanism — specifically `assessor.sol` and `coordinator.sol` after the lead-up files — because that is where the subordination logic is on-chain and explicit. Use **v3** (`ERC7540Vault.sol`, `InvestmentManager.sol`) only to see the async settlement interface.
