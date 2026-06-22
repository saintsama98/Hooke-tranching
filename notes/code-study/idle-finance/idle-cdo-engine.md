# The IdleCDO Engine

The engine is `contracts/IdleCDO.sol`. It holds the strategy position, accounts for value through a per-tranche virtual price, and divides yield between the two tranches. This note describes its structure at the level supported by the repository's documentation and the variant overrides examined for this study; precise arithmetic should be confirmed against the source.

## Components

| Contract | Role |
|---|---|
| `IdleCDO.sol` | The engine: deposit, withdrawal, harvest, price update, and yield split. |
| `IdleCDOTranche.sol` | The tranche token. Each of the senior (AA) and junior (BB) tranches is a minimal ERC-20 minted and burned by the engine. |
| `IdleCDOStorage.sol` | The storage layout, separated from logic to support upgradeability. |
| `IdleCDOFactory.sol` | Deployment of new instances. |
| `IdleCDO<Strategy>Variant.sol` | Per-strategy variants overriding deposit, withdrawal, or harvest as a strategy requires. |

## Tranche representation

The senior and junior tranches are two independent ERC-20 contracts (`IdleCDOTranche`), minted on deposit and burned on withdrawal. This is the plain "one ERC-20 per tranche" representation noted in [`../../standards/token-substrate-layer.md`](../../standards/token-substrate-layer.md): the engine, not the token, carries the seniority and accounting logic.

## Net-asset-value accounting

The engine maintains a virtual price for each tranche, updated on harvest, representing the value of one tranche token in the underlying asset. Gains raise the virtual price; losses reduce it. The senior and junior virtual prices diverge according to the yield split (see [`adaptive-yield-split.md`](./adaptive-yield-split.md)) and the allocation of loss. This checkpointed virtual price is component ② of the [reference architecture](../../protocols/reference-architecture.md): value is recognised internally from the strategy's reported position, an assumption discussed under loss recognition in [`../../synthesis/open-questions-and-next.md`](../../synthesis/open-questions-and-next.md).

## Deposit and withdrawal

On deposit, the engine mints tranche tokens against the deposited underlying and routes the underlying to the strategy. The Ethena variant illustrates the pattern with a minimal override:

```solidity
function _deposit(uint256 _amount, address _tranche, address _referral)
  internal override whenNotPaused returns (uint256 _minted) {
  _minted = super._deposit(_amount, _tranche, _referral);
  IIdleCDOStrategy(strategy).deposit(_amount);
}
```

On withdrawal, the engine burns tranche tokens and returns the underlying, recovering it from the strategy where necessary. For strategies whose redemption is not immediate, the variant defers the exit; see [`ethena-integration.md`](./ethena-integration.md).

## Loss allocation

Loss is absorbed by the junior tranche first. The senior tranche is affected only after the entire junior tranche value is exhausted, which gives strict subordination. There is one ordered boundary between two tranches, and the protection is passive: a loss reduces the junior virtual price directly, with no test evaluated and no cash flow redirected. The active redirection of cash flow on a coverage breach, present in traditional structures, is absent; this is the subject of [`mapping-to-standard.md`](./mapping-to-standard.md).

## The variant pattern

The engine is bound to a yield source through a strategy adapter and a thin `IdleCDO<Strategy>Variant`. The adapter presents a uniform interface (`IIdleCDOStrategy`) over a specific protocol, and the variant overrides only the methods that the strategy's mechanics require. This separation of a stable engine from a per-source adapter is the same composition principle the proposed standard adopts in placing a yield adapter beneath the engine; see [`../../protocols/reference-architecture.md`](../../protocols/reference-architecture.md).
