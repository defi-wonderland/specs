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
    - [`setRecipientA`](#setrecipienta)
    - [`setRecipientAShare`](#setrecipientashare)
    - [`setRecipientB`](#setrecipientb)
    - [`setFeeDisbursementInterval`](#setfeedisbursementinterval)
  - [Events](#events-4)
    - [`FeesDisbursed`](#feesdisbursed)
    - [`NoFeesCollected`](#nofeescollected)
    - [`RecipientAUpdated`](#recipientaupdated)
    - [`RecipientAShareUpdated`](#recipientashareupdated)
    - [`RecipientBUpdated`](#recipientbupdated)
    - [`FeeDisbursementIntervalUpdated`](#feedisbursementintervalupdated)
- [Invariants](#invariants-4)
- [Security Considerations](#security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

| Name        | Address                                    | Introduced | Deprecated | Proxied |
| ----------- | ------------------------------------------ | ---------- | ---------- | ------- |
| FeeSplitter | 0x420000000000000000000000000000000000001C | Jovian     | No         | Yes     |

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

This contract splits the ETH it receives and sends the correct amounts to two designated addresses. It integrates with
the fee vault system by configuring each Fee Vault to use `WithdrawalNetwork.L2` and setting this predeploy as the recipient in EVERY fee vault.

The contract manages two recipients:

- Recipient A
- Recipient B

Based on the configured share percentage for Recipient A, the contract sends the corresponding portion of fees
to Recipient A and sends the remaining amount to Recipient B.

### Functions

#### `disburseFees`

Initiates the routing flow by withdrawing the fees that each of the fee vaults has collected and sends the shares
to the appropriate addresses according to the configured percentage.

```solidity
function disburseFees() external
```

- MUST emit `FeesDisbursed` event upon successful execution.
- MUST emit `NoFeesCollected` event if there are no funds available at the time of the call.
- MUST revert if not enough time has passed since the last successful execution.
- MUST send the appropriate amounts to the recipients.
- The balance of the contract MUST be 0 after a successful execution.

#### `setRecipientA`

Sets the address that should receive the configured share of fees.

```solidity
function setRecipientA(address _recipientA) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit a `RecipientAUpdated` event upon successful execution.

#### `setRecipientAShare`

Sets the share percentage that recipient A should receive, in basis points.

```solidity
function setRecipientAShare(uint256 _shareBasisPoints) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST revert if `_shareBasisPoints` exceeds `BASIS_POINT_SCALE`.
- MUST emit a `RecipientAShareUpdated` event upon successful execution.

#### `setRecipientB`

Sets the address that should receive the remaining fees after the recipient A share has been deducted.

```solidity
function setRecipientB(address _recipientB) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit a `RecipientBUpdated` event upon successful execution.

#### `setFeeDisbursementInterval`

Sets the minimum time, in seconds, that must pass between consecutive calls to `disburseFees`.

```solidity
function setFeeDisbursementInterval(uint32 _newInterval) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit a `FeeDisbursementIntervalUpdated` event upon successful execution.

### Events

#### `FeesDisbursed`

Emitted when fees are successfully withdrawn from fee vaults and distributed to recipients.

```solidity
event FeesDisbursed(
        uint256 indexed disbursementTime, uint256 paidToRecipientA, uint256 paidToRecipientB, uint256 totalFeesDisbursed
    );
```

#### `NoFeesCollected`

Emitted when `disburseFees` is called but there are no funds available in the fee vaults at the time of execution.

```solidity
event NoFeesCollected()
```

#### `RecipientAUpdated`

Emitted when the address of Recipient A is successfully updated.

```solidity
event RecipientAUpdated(address indexed oldRecipientA, address indexed newRecipientA)
```

#### `RecipientAShareUpdated`

Emitted when the share percentage for Recipient A is successfully updated.

```solidity
event RecipientAShareUpdated(uint256 oldShare, uint256 newShare)
```

#### `RecipientBUpdated`

Emitted when the address of Recipient B is successfully updated.

```solidity
event RecipientBUpdated(address indexed oldRecipientB, address indexed newRecipientB)
```

#### `FeeDisbursementIntervalUpdated`

Emitted when the minimum time interval between consecutive fee disbursements is successfully updated.

```solidity
event FeeDisbursementIntervalUpdated(uint256 oldInterval, uint256 newInterval)
```

## Invariants

- Only `ProxyAdmin.owner()` is allowed to call the setter functions.
- The appropriate amounts based on the configured percentage MUST be sent to the correct addresses.
- Upon successful disbursement of fees, the sent amounts must sum to the total balance the contract had
  plus the accumulated fees in the fee vaults.
- The fee share of recipient A must be equal to or less than 10,000 basis points.
- `disburseFees` can only be called after `feeDisbursementInterval` seconds have passed since the last successful execution.

## Security Considerations

Given that vault recipients can now be updated, it's important to ensure that this can only be done by the appropriate address,
namely `ProxyAdmin.owner()`.
