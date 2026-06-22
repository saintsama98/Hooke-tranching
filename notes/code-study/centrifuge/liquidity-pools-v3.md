# Liquidity Pools (v3)

The current Centrifuge codebase ([`centrifuge/liquidity-pools`](https://github.com/centrifuge/liquidity-pools), also `centrifuge/protocol`) reissues tranches as **ERC-7540 asynchronous vaults** and extends the protocol across chains. It is the reference for the settlement interface, not for the coverage logic.

## Tranche-relevant contracts (`src/`)

| Contract | Role |
|---|---|
| `token/Tranche.sol` | The tranche token: a restricted ERC-20 (the DROP/TIN equivalent). |
| `token/RestrictionManager.sol` | Transfer restrictions / compliance (the memberlist equivalent). |
| `ERC7540Vault.sol` | One async vault per tranche: `requestDeposit`/`requestRedeem` followed by later fulfilment. The settlement rail. |
| `InvestmentManager.sol` | Processes investment and redemption requests and applies the fulfilled prices, minting and burning tranche tokens. |
| `PoolManager.sol` | Registers pools, tranches, and currencies, and deploys vaults and tranche tokens via the factories. |
| `Escrow.sol` | Holds pending assets and shares between request and fulfilment. |
| `factories/ERC7540VaultFactory.sol`, `factories/TrancheFactory.sol` | Deployment of vaults and tranche tokens. |

## The off-chain relocation

The contracts above hold tokens, accept requests, and apply prices, but they do **not** compute the senior ratio, the NAV, or the epoch solution. That logic runs **off-chain on the Centrifuge chain**, and the results are delivered to the EVM contracts as messages through the cross-chain layer (`gateway/Gateway.sol` and its adapters). The `InvestmentManager` then applies the fulfilled prices that arrive.

The practical consequence: the explicit coverage math that is on-chain and auditable in Tinlake's Assessor and Coordinator is, in v3, **not present on the EVM side at all**. The senior/junior pricing and the subordination decision are computed by an off-chain system and trusted on arrival.

## Why ERC-7540 rather than ERC-4626

Tinlake's redemption is asynchronous and liquidity-constrained: an investor submits an order and is fulfilled, possibly partially, at epoch close. A synchronous ERC-4626 vault cannot represent this. ERC-7540 is the asynchronous vault standard (`requestDeposit`/`requestRedeem` → later fulfilment), so v3 expresses Tinlake's epoch/order model through the standard built for it. This is direct evidence for composing ERC-7540 as the settlement layer beneath a tranche engine; see [`../../standards/vault-and-yield-layer.md`](../../standards/vault-and-yield-layer.md).

## Relevance to the standard

v3 confirms two things. First, the asynchronous settlement model is real and recurring, and ERC-7540 is its standardised form. Second, at the modern multichain layer no protocol keeps the coverage logic on-chain — Centrifuge moved it off-chain entirely — which is precisely the gap the proposed standard addresses by specifying the coverage engine as verifiable on-chain behaviour. See [`mapping-to-standard.md`](./mapping-to-standard.md).
