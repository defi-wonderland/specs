# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Overview](#overview)
- [SequencerFeeVault](#sequencerfeevault)
  - [Functions](#functions)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount)
    - [`setRecipient`](#setrecipient)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork)
  - [Events](#events)
    - [`MinWithdrawalAmountUpdated`](#minwithdrawalamountupdated)
    - [`RecipientUpdated`](#recipientupdated)
    - [`WithdrawalNetworkUpdated`](#withdrawalnetworkupdated)
  - [Invariants](#invariants)
- [BaseFeeVault](#basefeevault)
  - [Functions](#functions-1)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount-1)
    - [`setRecipient`](#setrecipient-1)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork-1)
  - [Events](#events-1)
    - [`MinWithdrawalAmountUpdated`](#minwithdrawalamountupdated-1)
    - [`RecipientUpdated`](#recipientupdated-1)
    - [`WithdrawalNetworkUpdated`](#withdrawalnetworkupdated-1)
  - [Invariants](#invariants-1)
- [L1FeeVault](#l1feevault)
  - [Functions](#functions-2)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount-2)
    - [`setRecipient`](#setrecipient-2)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork-2)
  - [Events](#events-2)
    - [`MinWithdrawalAmountUpdated`](#minwithdrawalamountupdated-2)
    - [`RecipientUpdated`](#recipientupdated-2)
    - [`WithdrawalNetworkUpdated`](#withdrawalnetworkupdated-2)
  - [Invariants](#invariants-2)
- [OperatorFeeVault](#operatorfeevault)
  - [Functions](#functions-3)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount-3)
    - [`setRecipient`](#setrecipient-3)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork-3)
  - [Events](#events-3)
    - [`MinWithdrawalAmountUpdated`](#minwithdrawalamountupdated-3)
    - [`RecipientUpdated`](#recipientupdated-3)
    - [`WithdrawalNetworkUpdated`](#withdrawalnetworkupdated-3)
  - [Invariants](#invariants-3)
- [FeeSplitter](#feesplitter)
  - [Functions](#functions-4)
    - [`disburseFees`](#disbursefees)
    - [`setL1FeeWallet`](#setl1feewallet)
    - [`setL1FeeWalletShare`](#setl1feewalletshare)
    - [`setOptimismWallet`](#setoptimismwallet)
    - [`setFeeDisbursementInterval`](#setfeedisbursementinterval)
- [Security Considerations](#security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The `FeeSplitter` predeploy manages the distribution of all L2 fees. Fee vault contracts (`OperatorFeeVault`,
`BaseFeeVault`, `L1FeeVault`, and `SequencerFeeVault`) update their configuration via setter functions for
minimum withdrawal amounts, withdrawal networks, and recipients without requiring new deployments.

Using the `FeeSplitter` requires vaults to use `WithdrawalNetwork.L2` and set the `FeeSplitter` as their
fee recipient. New chains integrate with the `FeeSplitter` by default but can override this configuration for
alternative setups. Existing chains can optionally adopt it, ensuring backward compatibility.

## SequencerFeeVault

This contract's configuration includes the withdrawal network and the recipient to which the fees will be
sent.

- **WithdrawalNetwork.L1**: Funds are withdrawn to an L1 address (default behavior)
- **WithdrawalNetwork.L2**: Funds are withdrawn to an L2 address

For existing chains that choose to use the `FeeSplitter` predeploy, `WithdrawalNetwork.L2` as the
withdrawal network and the `FeeSplitter` as the recipient MUST be set using the setter functions.

New chain deployments that choose to use the `FeeSplitter` MUST use the default configuration.

### Functions

#### `setMinWithdrawalAmount`

Updates the minimum amount of funds the SequencerFeeVault contract must hold before they can be withdrawn.

```solidity
function setMinWithdrawalAmount(uint256 _minWithdrawalAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `MinWithdrawalAmountUpdated` event

#### `setRecipient`

Updates the recipient of sequencer fees when they are withdrawn from the vault.

```solidity
function setRecipient(address _recipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `RecipientUpdated` event

#### `setWithdrawalNetwork`

Updates the network to which sequencer fees will be withdrawn.
This can be either `WithdrawalNetwork.L1` to withdraw them to an address on L1 by using the `L2ToL1MessagePasser`
predeploy, or `WithdrawalNetwork.L2` to withdraw them to an address on the same chain.

```solidity
function setWithdrawalNetwork(WithdrawalNetwork _withdrawalNetwork) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `WithdrawalNetworkUpdated` event

### Events

#### `MinWithdrawalAmountUpdated`

Emitted when the minimum withdrawal amount for the vault is updated.

```solidity
event MinWithdrawalAmountUpdated(uint256 oldWithdrawalAmount, uint256 newWithdrawalAmount)
```

#### `RecipientUpdated`

Emitted when the fee recipient for this vault is updated.

```solidity
event RecipientUpdated(address oldRecipient, address newRecipient)
```

#### `WithdrawalNetworkUpdated`

Emitted when the withdrawal network for this vault is updated.

```solidity
event WithdrawalNetworkUpdated(WithdrawalNetwork oldWithdrawalNetwork, WithdrawalNetwork newWithdrawalNetwork)
```

### Invariants

- Only `ProxyAdmin.owner()` is allowed to call the setter functions.
- If using the `FeeSplitter`, the withdrawal network MUST be set to `WithdrawalNetwork.L2` and the recipient
  MUST be set to the `FeeSplitter` predeploy address.
- The balance of the vault MUST be preserved between implementation upgrades.

## BaseFeeVault

This contract's configuration includes the withdrawal network and the recipient to which the fees will be
sent.

- **WithdrawalNetwork.L1**: Funds are withdrawn to an L1 address (default behavior)
- **WithdrawalNetwork.L2**: Funds are withdrawn to an L2 address

For existing chains that choose to use the `FeeSplitter` predeploy, `WithdrawalNetwork.L2` as the
withdrawal network and the `FeeSplitter` as the recipient MUST be set using the setter functions.

New chain deployments that choose to use the `FeeSplitter` MUST use the default configuration.

### Functions

#### `setMinWithdrawalAmount`

Updates the minimum amount of funds the BaseFeeVault contract must hold before they can be withdrawn.

```solidity
function setMinWithdrawalAmount(uint256 _minWithdrawalAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `MinWithdrawalAmountUpdated` event

#### `setRecipient`

Updates the recipient of base fees when they are withdrawn from the vault.

```solidity
function setRecipient(address _recipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `RecipientUpdated` event

#### `setWithdrawalNetwork`

Updates the network to which base fees will be withdrawn.
This can be either `WithdrawalNetwork.L1` to withdraw them to an address on L1 by using the `L2ToL1MessagePasser`
predeploy, or `WithdrawalNetwork.L2` to withdraw them to an address on the same chain.

```solidity
function setWithdrawalNetwork(WithdrawalNetwork _withdrawalNetwork) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `WithdrawalNetworkUpdated` event

### Events

#### `MinWithdrawalAmountUpdated`

Emitted when the minimum withdrawal amount for the vault is updated.

```solidity
event MinWithdrawalAmountUpdated(uint256 oldWithdrawalAmount, uint256 newWithdrawalAmount)
```

#### `RecipientUpdated`

Emitted when the fee recipient for this vault is updated.

```solidity
event RecipientUpdated(address oldRecipient, address newRecipient)
```

#### `WithdrawalNetworkUpdated`

Emitted when the withdrawal network for this vault is updated.

```solidity
event WithdrawalNetworkUpdated(WithdrawalNetwork oldWithdrawalNetwork, WithdrawalNetwork newWithdrawalNetwork)
```

### Invariants

- Only `ProxyAdmin.owner()` is allowed to call the setter functions.
- If using the `FeeSplitter`, the withdrawal network MUST be set to `WithdrawalNetwork.L2` and the recipient
  MUST be set to the `FeeSplitter` predeploy address.
- The balance of the vault MUST be preserved between implementation upgrades.

## L1FeeVault

This contract's configuration includes the withdrawal network and the recipient to which the fees will be
sent.

- **WithdrawalNetwork.L1**: Funds are withdrawn to an L1 address (default behavior)
- **WithdrawalNetwork.L2**: Funds are withdrawn to an L2 address

For existing chains that choose to use the `FeeSplitter` predeploy, `WithdrawalNetwork.L2` as the
withdrawal network and the `FeeSplitter` as the recipient MUST be set using the setter functions.

New chain deployments that choose to use the `FeeSplitter` MUST use the default configuration.

### Functions

#### `setMinWithdrawalAmount`

Updates the minimum amount of funds the L1FeeVault contract must hold before they can be withdrawn.

```solidity
function setMinWithdrawalAmount(uint256 _minWithdrawalAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `MinWithdrawalAmountUpdated` event

#### `setRecipient`

Updates the recipient of L1 fees when they are withdrawn from the vault.

```solidity
function setRecipient(address _recipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `RecipientUpdated` event

#### `setWithdrawalNetwork`

Updates the network to which L1 fees will be withdrawn.
This can be either `WithdrawalNetwork.L1` to withdraw them to an address on L1 by using the `L2ToL1MessagePasser`
predeploy, or `WithdrawalNetwork.L2` to withdraw them to an address on the same chain.

```solidity
function setWithdrawalNetwork(WithdrawalNetwork _withdrawalNetwork) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `WithdrawalNetworkUpdated` event

### Events

#### `MinWithdrawalAmountUpdated`

Emitted when the minimum withdrawal amount for the vault is updated.

```solidity
event MinWithdrawalAmountUpdated(uint256 oldWithdrawalAmount, uint256 newWithdrawalAmount)
```

#### `RecipientUpdated`

Emitted when the fee recipient for this vault is updated.

```solidity
event RecipientUpdated(address oldRecipient, address newRecipient)
```

#### `WithdrawalNetworkUpdated`

Emitted when the withdrawal network for this vault is updated.

```solidity
event WithdrawalNetworkUpdated(WithdrawalNetwork oldWithdrawalNetwork, WithdrawalNetwork newWithdrawalNetwork)
```

### Invariants

- Only `ProxyAdmin.owner()` is allowed to call the setter functions.
- If using the `FeeSplitter`, the withdrawal network MUST be set to `WithdrawalNetwork.L2` and the recipient
  MUST be set to the `FeeSplitter` predeploy address.
- The balance of the vault MUST be preserved between implementation upgrades.

## OperatorFeeVault

This contract's configuration includes the withdrawal network and the recipient to which the fees will be
sent.

- **WithdrawalNetwork.L1**: Funds are withdrawn to an L1 address (default behavior)
- **WithdrawalNetwork.L2**: Funds are withdrawn to an L2 address

For existing chains that choose to use the `FeeSplitter` predeploy, `WithdrawalNetwork.L2` as the
withdrawal network and the `FeeSplitter` as the recipient MUST be set using the setter functions.

New chain deployments that choose to use the `FeeSplitter` MUST use the default configuration.

### Functions

#### `setMinWithdrawalAmount`

Updates the minimum amount of funds the OperatorFeeVault contract must hold before they can be withdrawn.

```solidity
function setMinWithdrawalAmount(uint256 _minWithdrawalAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `MinWithdrawalAmountUpdated` event

#### `setRecipient`

Updates the recipient of operator fees when they are withdrawn from the vault.

```solidity
function setRecipient(address _recipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `RecipientUpdated` event

#### `setWithdrawalNetwork`

Updates the network to which operator fees will be withdrawn.
This can be either `WithdrawalNetwork.L1` to withdraw them to an address on L1 by using the `L2ToL1MessagePasser`
predeploy, or `WithdrawalNetwork.L2` to withdraw them to an address on the same chain.

```solidity
function setWithdrawalNetwork(WithdrawalNetwork _withdrawalNetwork) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `WithdrawalNetworkUpdated` event

### Events

#### `MinWithdrawalAmountUpdated`

Emitted when the minimum withdrawal amount for the vault is updated.

```solidity
event MinWithdrawalAmountUpdated(uint256 oldWithdrawalAmount, uint256 newWithdrawalAmount)
```

#### `RecipientUpdated`

Emitted when the fee recipient for this vault is updated.

```solidity
event RecipientUpdated(address oldRecipient, address newRecipient)
```

#### `WithdrawalNetworkUpdated`

Emitted when the withdrawal network for this vault is updated.

```solidity
event WithdrawalNetworkUpdated(WithdrawalNetwork oldWithdrawalNetwork, WithdrawalNetwork newWithdrawalNetwork)
```

### Invariants

- Only `ProxyAdmin.owner()` is allowed to call the setter functions.
- If using the `FeeSplitter`, the withdrawal network MUST be set to `WithdrawalNetwork.L2` and the recipient
  MUST be set to the `FeeSplitter` predeploy address.
- The balance of the vault MUST be preserved between implementation upgrades.

## FeeSplitter

This contract is responsible for splitting the ETH it receives and sending the correct amounts to
the appropriate addresses. It contains two addresses: one for the OP chain runner's share and one
for Optimism's revenue share. The contract will send a portion of the fee to each address according
to their respective percentages.

### Functions

#### `disburseFees`

Initiates the routing flow by withdrawing the fees that each of the fee vaults has collected and sends the shares
to the appropriate addresses according to the configured percentage for each. The function enforces a time-based
cooldown period and will revert if insufficient time has elapsed since the previous call. When executed,
it withdraws the fees from all of the fee vaults and disburses the full balance of the contract, distributing the
appropriate amounts to all configured addresses. Upon successful distribution, the function emits a `FeesDisbursed` event.
In cases where no funds are available for disbursement at the time of the call, the function emits a
`NoFeesCollected` event instead.

#### `setL1FeeWallet`

Sets the address that should receive the chain runner's share of fees.
This function is only callable by `ProxyAdmin.owner()`
and will emit an `L1WalletUpdated` event upon successful execution.

#### `setL1FeeWalletShare`

Sets the share percentage that the chain runner's Wallet should receive, in basis points.
This function is only callable by `ProxyAdmin.owner()` and validates
that the new share percentage does not exceed `BASIS_POINT_SCALE`.
Upon successful execution, it emits an `L1FeeWalletShareUpdated` event.

#### `setOptimismWallet`

Sets the address that should receive the remaining fees after the L1 Fee share has been deducted.
This function is only callable by `ProxyAdmin.owner()` and will emit a `FeeCollectorUpdated`
event upon successful execution.

#### `setFeeDisbursementInterval`

Sets the minimum time that must pass between consecutive calls to `disburseFees`.
This function is only callable by `ProxyAdmin.owner()` and validates that the new interval is at least `MINIMUM_DISBURSE_INTERVAL`.
Upon successful execution, it emits a `FeeDisbursementIntervalUpdated` event.

## Security Considerations

Given that vault recipients can now be updated, it's important to ensure that this can only be done by the appropriate address,
namely `ProxyAdmin.owner()`.
