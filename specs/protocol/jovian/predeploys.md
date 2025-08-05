# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Overview](#overview)
- [SequencerFeeVault](#sequencerfeevault)
  - [Functions](#functions)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount)
    - [`setRecipient`](#setrecipient)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork)
- [BaseFeeVault](#basefeevault)
  - [Functions](#functions-1)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount-1)
    - [`setRecipient`](#setrecipient-1)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork-1)
- [L1FeeVault](#l1feevault)
  - [Functions](#functions-2)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount-2)
    - [`setRecipient`](#setrecipient-2)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork-2)
- [OperatorFeeVault](#operatorfeevault)
  - [Functions](#functions-3)
    - [`setMinWithdrawalAmount`](#setminwithdrawalamount-3)
    - [`setRecipient`](#setrecipient-3)
    - [`setWithdrawalNetwork`](#setwithdrawalnetwork-3)
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

A new predeploy contract, `FeeSplitter`, is introduced to manage the distribution of all fees collected on L2.
Instead of having immutable recipients, the `RECIPIENT` address in each of the four `FeeVault` contracts will
be set to the `FeeSplitter` contract on new chains, while existing chains will be able to opt-in to use this new predeploy.

## SequencerFeeVault

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

## BaseFeeVault

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

## L1FeeVault

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

## OperatorFeeVault

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
