# Ethena (sUSDe) Integration

The Ethena variant illustrates how Idle binds the engine to a yield source whose redemption is not immediate, and does so without any decentralised-exchange swap. The integration comprises four files in [`Idle-Labs/idle-tranches`](https://github.com/Idle-Labs/idle-tranches):

- `contracts/IdleCDOEthenaVariant.sol`
- `contracts/strategies/ethena/EthenaSusdeStrategy.sol`
- `contracts/strategies/ethena/EthenaCooldownRequest.sol`
- `contracts/interfaces/ethena/IStakedUSDeV2.sol`

## Underlying and strategy token

The engine's underlying asset is USDe; the strategy token is sUSDe, Ethena's `StakedUSDeV2`, which is itself an ERC-4626 vault whose asset is USDe. Confirmed addresses, from the strategy test fixture:

```solidity
address internal constant USDe  = 0x4c9EDD5852cd905f086C759E8383e09bff1E68B3; // underlying / deposit asset
address internal constant SUSDe = 0x9D39A5DE30e57443BfF2A8307A4256c8797A3497; // strategy token (StakedUSDeV2)
address internal defaultUnderlying = USDe;
```

The strategy is a thin extension of `ERC4626Strategy`:

```solidity
contract EthenaSusdeStrategy is ERC4626Strategy {
  function initialize(address _vault, address _underlying, address _owner) public {
    _initialize(_vault, _underlying, _owner);
  }
}
```

Because the underlying equals the strategy's base asset (USDe), the engine never converts between assets. Any conversion — for example from USDC to USDe — is performed by the depositor or a front-end before deposit, outside the engine, and bears its own liquidity and slippage risk. This is the single-asset denomination principle: the engine operates entirely in one asset, and conversion is external.

## Deposit: native staking

On deposit, the underlying USDe is staked into sUSDe through the strategy. No swap occurs.

```solidity
function _deposit(uint256 _amount, address _tranche, address _referral)
  internal override whenNotPaused returns (uint256 _minted) {
  _minted = super._deposit(_amount, _tranche, _referral);
  IIdleCDOStrategy(strategy).deposit(_amount); // USDe -> sUSDe
}
```

## Redemption: the native cooldown

sUSDe cannot be redeemed immediately; `StakedUSDeV2` enforces a per-address cooldown, after which the position is unstaked to USDe. The variant uses this native path rather than a swap, and manages the per-address constraint by deploying a minimal-proxy clone for each withdrawal:

```solidity
EthenaCooldownRequest clone = EthenaCooldownRequest(
  cooldownImpl.clone(abi.encodePacked(address(this), msg.sender), msg.value));
IERC20Detailed(strategyToken).safeTransfer(address(clone), SUSDeRedeemed);
clone.startCooldown();
```

The clone (`EthenaCooldownRequest`, implementation `0xe0C4a2B14F0ACd936226A598BE6BfeD190E098d1`) holds the sUSDe for one request and exposes:

```solidity
function startCooldown() external { require(msg.sender == _getCDO(), '6');
  IStakedUSDeV2(SUSDE).cooldownShares(/* full sUSDe balance */); }
function unstake() external { IStakedUSDeV2(SUSDE).unstake(_getUser()); } // sends USDe to the user
function rescue(address _token) external; // timelock multisig only
```

The CDO address and the user address are stored as immutable arguments in the clone's bytecode, so `unstake()` delivers USDe directly to the original withdrawer after the cooldown elapses.

## Rationale for the clone pattern

Ethena's cooldown is keyed per address, with one cooldown slot per holder. Routing every withdrawal through a single contract would cause a new request to overwrite a prior request's timer and amount. A clone per withdrawal gives each request an isolated cooldown slot at minimal deployment cost.

## Consequence for settlement

The integration trades an asset-conversion problem for a redemption-latency problem: exits are not immediate but are deferred behind the cooldown. An asset with redemption latency cannot settle synchronously, which is the practical case for composing an asynchronous, request-based settlement layer. This is direct evidence for the settlement choice in the proposed standard; see [`mapping-to-standard.md`](./mapping-to-standard.md) and [`../../standards/vault-and-yield-layer.md`](../../standards/vault-and-yield-layer.md).
