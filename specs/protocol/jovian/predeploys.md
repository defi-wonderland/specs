# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Predeploys](#predeploys)
  - [Overview](#overview)
  - [Fee Vaults (SequencerFeeVault, L1FeeVault, BaseFeeVault, OperatorFeeVault)](#fee-vaults-sequencerfeevault-l1feevault-basefeevault-operatorfeevault)
    - [Functions](#functions)
      - [`setMinWithdrawalAmount`](#setminwithdrawalamount)
      - [`setRecipient`](#setrecipient)
      - [`setWithdrawalNetwork`](#setwithdrawalnetwork)
    - [Events](#events)
      - [`MinWithdrawalAmountUpdated`](#minwithdrawalamountupdated)
      - [`RecipientUpdated`](#recipientupdated)
      - [`WithdrawalNetworkUpdated`](#withdrawalnetworkupdated)
    - [Invariants](#invariants)
  - [FeeSplitter](#feesplitter)
    - [Functions](#functions-1)
      - [`disburseFees`](#disbursefees)
      - [`setRecipientA`](#setrecipienta)
      - [`setRecipientAShare`](#setrecipientashare)
      - [`setRecipientB`](#setrecipientb)
      - [`setFeeDisbursementInterval`](#setfeedisbursementinterval)
    - [Events](#events-1)
      - [`FeesDisbursed`](#feesdisbursed)
      - [`NoFeesCollected`](#nofeescollected)
      - [`RecipientAUpdated`](#recipientaupdated)
      - [`RecipientAShareUpdated`](#recipientashareupdated)
      - [`RecipientBUpdated`](#recipientbupdated)
      - [`FeeDisbursementIntervalUpdated`](#feedisbursementintervalupdated)
  - [Invariants](#invariants-1)
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

## Fee Vaults (SequencerFeeVault, L1FeeVault, BaseFeeVault, OperatorFeeVault)

These contracts' configuration includes the withdrawal network and the recipient to which the fees will be sent.

- **WithdrawalNetwork.L1**: Funds are withdrawn to an L1 address (default behavior)
- **WithdrawalNetwork.L2**: Funds are withdrawn to an L2 address

For existing chains that choose to use the `FeeSplitter` predeploy, `WithdrawalNetwork.L2` as the
withdrawal network and the `FeeSplitter` as the recipient MUST be set using the setter functions.

New chain deployments that choose to use the `FeeSplitter` MUST use the default configuration.

### Functions

#### `setMinWithdrawalAmount`

Updates the minimum amount of funds the vault contract must hold before they can be withdrawn.

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

Updates the network to which vault collected fees will be withdrawn.
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
the fee vault system by configuring each Fee Vault to use `WithdrawalNetwork.L2` and setting this predeploy as the
recipient in EVERY fee vault.

The contract manages two recipients:

- Revenue share recipient
- Remainder recipient

Based on the configured share percentage for the revenue share recipient, the contract sends the corresponding
portion of fees to that recipient and sends the remainder to the remainder recipient.

### Functions

#### `disburseFees`

Initiates the routing flow by withdrawing the fees that each of the fee vaults has collected and sends the shares
to the appropriate addresses according to the configured percentage.
The function MUST revert if the withdrawal is not set to `WithdrawalNetwork.L2`, or if the recipient set is not the `FeeSplitter`.
The function MUST withdraw the vault's fees balance only if this has a balance equal or greater than the set minimun amount.

When attempting to withdraw from the vaults, it will check that they all have a balance equal to or larger than
their minimum withdrawal amount, that the withdrawal network is set to `WithdrawalNetwork.L2`, and that the recipient
of the vault is the `FeeSplitter`. The function MUST revert if any of these conditions is not met.
The function MUST emit the `NoFeesCollected` event if the contract doesn't have any funds after the vaults have been withdrawn.

```solidity
function disburseFees() external
```

- MUST revert if not enough time has passed since the last successful execution.
- MUST revert if any vault has a recipient different from this contract.
- MUST revert if any vault has a withdrawal network different from `WithdrawalNetwork.L2`.
- MUST send the appropriate amounts to the recipients.
- MUST withdraw vault's fees balance if the vault's balance is equal or greater than the min amount set.
- MUST emit `NoFeesCollected` event if there are no funds available in the contract after the vaults have been withdrawn.
- MUST emit `FeesDisbursed` event if the funds were dibursed.
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

Emitted when `disburseFees` is called but there are no funds available in the contract _after_ the fee vaults
have been withdrawn. This can happen in scenarios where the fee vaults' minimum withdrawal amount is
configured to `0`.

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
