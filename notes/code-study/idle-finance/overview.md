# Idle Finance — Overview

## The protocol

Idle Finance is an Ethereum yield-automation protocol. It originated the on-chain Perpetual Yield Tranches model, in which a single yield strategy is divided into a senior tranche (AA) and a junior tranche (BB) over one underlying asset. The senior tranche receives a protected, lower yield; the junior tranche receives a higher yield and absorbs loss first. The protocol is the most architecturally complete two-tranche engine in production and is therefore the primary reference for this study, independent of its current scale.

## Current scale

The protocol is small by current total value locked and is past its earlier peak. Figures retrieved 2026-06.

| Metric | Value | Source |
|---|---|---|
| Total value locked | Approximately 3.6 million USD (DefiLlama); approximately 4.77 million USD (Stelareum) | [DefiLlama](https://defillama.com/protocol/idle), [Stelareum](https://www.stelareum.io/en/defi-tvl/protocol/idle.html) |
| Chain concentration | Approximately 98.6% on Ethereum | [DefiLlama](https://defillama.com/protocol/idle) |
| IDLE token | Micro-capitalisation (ranked approximately 9806) | [CoinGecko](https://www.coingecko.com/en/coins/idle) |

## The Pareto transition

Idle is rebranding to Pareto (token PAR), repositioning as a credit-coordination protocol for institutional, real-world-asset private credit, with reported committed value in excess of 25 million USD on the new direction. Sources: [Pareto governance — Rebranding Idle to Pareto](https://gov.pareto.credit/t/rebranding-idle-to-pareto-and-growth-plan/1279), [pareto.idle.finance](https://pareto.idle.finance/).

This transition is itself relevant to the proposed standard. The migration of an originator of on-chain tranching toward structured institutional credit is consistent with the direction of Centrifuge, Maple, and the recent securitisation entrants surveyed in [`../../protocols/rwa-securitization-entrants.md`](../../protocols/rwa-securitization-entrants.md), and supports the case for a waterfall standard oriented toward credit.

## Repository structure

The tranche system is implemented in [`Idle-Labs/idle-tranches`](https://github.com/Idle-Labs/idle-tranches). The components relevant to this study are:

- `contracts/IdleCDO.sol` — the engine.
- `contracts/IdleCDOTranche.sol` — the senior and junior tranche token.
- `contracts/IdleCDOStorage.sol` — the storage layout.
- `contracts/IdleCDOFactory.sol` — the deployment factory.
- `contracts/IdleCDO<Strategy>Variant.sol` — per-strategy variants, each binding the engine to a specific strategy and underlying. Observed variants include Best Yield, Amphor, Epoch, Ethena, Gearbox, Instadapp Lite, Leveraged Euler, PoLido, Truefi, and Usual.
- `contracts/strategies/<protocol>/` — the strategy adapters, for example `strategies/ethena/EthenaSusdeStrategy.sol`.

Each variant follows one rule: the engine's underlying asset is the base asset of the strategy it wraps. The protocol therefore has no single underlying asset; each deployment is denominated in its strategy's base asset, predominantly major stablecoins, with some liquid-staking variants. The Ethena variant, examined in [`ethena-integration.md`](./ethena-integration.md), is denominated in USDe.
