# Centrifuge — Overview

Centrifuge tokenises real-world-asset credit pools and divides each pool into a senior and a junior tranche. It exists in two generations, and the right one to study depends on the concern.

## The two generations

| | Tinlake (v2) | Liquidity Pools / Protocol (v3) |
|---|---|---|
| Chains | Single-chain (Ethereum) | Multichain |
| Tranche tokens | **DROP** (senior) / **TIN** (junior), restricted ERC-20 | Restricted ERC-20 tranche token per vault |
| Settlement | Epoch-based order solving, on-chain | **ERC-7540** async vaults |
| Coverage / pricing math | **On-chain** (Assessor, Coordinator) | **Off-chain** (computed on the Centrifuge chain, applied on the EVM side) |
| Toolchain | dapptools / MakerDAO "dss" style | Foundry |
| Repository | [`centrifuge/tinlake`](https://github.com/centrifuge/tinlake) | [`centrifuge/liquidity-pools`](https://github.com/centrifuge/liquidity-pools), [`centrifuge/protocol`](https://github.com/centrifuge/protocol) |

## Where to study which concern

- **The tranche/coverage *mechanism*** → Tinlake. Its valuation, subordination, and epoch logic are on-chain and explicit. See [`tinlake-architecture.md`](./tinlake-architecture.md) and [`assessor-and-coordinator.md`](./assessor-and-coordinator.md).
- **The ERC-7540 settlement *interface*** → Liquidity Pools v3. See [`liquidity-pools-v3.md`](./liquidity-pools-v3.md).

The consequence for the proposed standard: the coverage logic worth learning from is in Tinlake. v3 deliberately moved that logic off-chain, which is the inverse of the standard's goal of verifiable on-chain coverage. This is examined in [`mapping-to-standard.md`](./mapping-to-standard.md).

## Token vocabulary (Tinlake)

Three distinct token types appear in the code and should not be conflated:

- **`currency`** — the pool's underlying ERC-20 stablecoin (DAI in deployments). What investors deposit and redeem into.
- **Tranche tokens** — **DROP** (senior) and **TIN** (junior); the investor's shares, priced by the Assessor.
- **Collateral** — NFTs representing the real-world loans (borrower side); not part of the tranche mechanism.

An investment deposits `currency` and receives tranche tokens. A **redeem** is a queued request to convert tranche tokens back into `currency`, fulfilled (possibly partially) at epoch close, subject to liquidity and the senior-coverage constraints. Centrifuge has **no ERC-4626 vault** in Tinlake: the protocol predates ERC-4626, and its epoch/async model cannot be expressed by a synchronous vault interface; the vault concept appears only in v3, as ERC-7540.
