# FeeVaultInitializer

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Summary](#summary)
- [Upgrade Path](#upgrade-path)
- [Functions](#functions)
  - [`constructor`](#constructor)
- [Events](#events)
  - [`FeeVaultDeployed`](#feevaultdeployed)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

A contract responsible for deploying new vault implementations while preserving the current vault configuration.
It is intended to be used in a Network Upgrade Transaction (NUT) series context to upgrade each vault to the
new implementation once deployed.

## Upgrade Path

To deploy and activate new FeeVault implementations through NUTs, we must:

1. **Precalculate:**

   - `FeeVaultInitializer` address to be deployed
   - Each vault implementation address to be deployed by the `FeeVaultInitializer`

2. **Deploy the `FeeVaultInitializer`:** The constructor deploys the new implementations while
   preserving the current configuration values.

3. **Upgrade each vault proxy:** For each vault proxy, call `upgradeTo` (or `upgradeToAndCall` if required
   by local policy) from the admin or via the `address(0)` NUT flow to point the proxy to the emitted
   implementation addresses (requires 4 separate NUTs).

## Functions

### `constructor`

When the initializer is deployed, it will:

- Read and retrieve the current values for `recipient()`, `withdrawalNetwork()`, and `minWithdrawalAmount()`
  from each live vault (`SequencerFeeVault`, `L1FeeVault`, `BaseFeeVault`, `OperatorFeeVault`).
- Handle legacy vaults that may not have the `WITHDRAWAL_NETWORK` function by using a low-level staticcall and
  assigning the default value of `WithdrawalNetwork.L2` when the configuration cannot be read.
- Deploy a new implementation for each vault, passing the retrieved values per vault as constructor
  immutables, and passing `L2` as the `WITHDRAWAL_NETWORK` constructor immutable.
- Emit the `FeeVaultDeployed` event for each newly deployed vault.

```solidity
constructor()
```

- MUST deploy implementations whose constructor immutables match the current configuration values
- For legacy vaults, MUST assign a default `WithdrawalNetwork.L2` value to the network configuration on the new implementation
- For each vault, MUST emit the `FeeVaultDeployed` event

## Events

### `FeeVaultDeployed`

Emitted when a new fee vault implementation has been deployed.

```solidity
event FeeVaultDeployed(
    string indexed vaultType,
    address indexed newImplementation,
    address recipient,
    WithdrawalNetwork network,
    uint256 minWithdrawalAmount
)
```
