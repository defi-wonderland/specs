# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

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
    - [Constants](#constants)
    - [Functions](#functions-1)
      - [`initialize`](#initialize)
      - [`disburseFees`](#disbursefees)
      - [`receive`](#receive)
      - [`setRevenueShareRecipient`](#setrevenuesharerecipient)
      - [`setNetFeeShareBP`](#setnetfeesharebp)
      - [`setGrossFeeShareBP`](#setgrossfeesharebp)
      - [`setRevenueRemainderRecipient`](#setrevenueremainderrecipient)
      - [`setFeeDisbursementInterval`](#setfeedisbursementinterval)
    - [Events](#events-1)
      - [`FeesDisbursed`](#feesdisbursed)
      - [`NoFeesCollected`](#nofeescollected)
      - [`FeesReceived`](#feesreceived)
      - [`NetFeeShareBPUpdated`](#netfeesharebpupdated)
      - [`GrossFeeShareBPUpdated`](#grossfeesharebpupdated)
      - [`Initialized`](#initialized)
      - [`RevenueShareRecipientUpdated`](#revenuesharerecipientupdated)
      - [`RevenueRemainderRecipientUpdated`](#revenueremainderrecipientupdated)
      - [`FeeDisbursementIntervalUpdated`](#feedisbursementintervalupdated)
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
fee recipient. All chains can optionally adopt it or not.

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
function setMinWithdrawalAmount(uint256 _newMinWithdrawalAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `MinWithdrawalAmountUpdated` event

#### `setRecipient`

Updates the recipient of sequencer fees when they are withdrawn from the vault.

```solidity
function setRecipient(address _newRecipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit the `RecipientUpdated` event

#### `setWithdrawalNetwork`

Updates the network to which vault collected fees will be withdrawn.
This can be either `WithdrawalNetwork.L1` to withdraw them to an address on L1 by using the `L2ToL1MessagePasser`
predeploy, or `WithdrawalNetwork.L2` to withdraw them to an address on the same chain.

```solidity
function setWithdrawalNetwork(WithdrawalNetwork _newWithdrawalNetwork) external
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

And it has two ways of dividing the revenue (considered as the total amount of ETH received by the contract since the last disbursement):

- `grossRevenue`: The whole balance received by the contract.
- `netRevenue`: Only the fees collected by the `SequencerFeeVault`, `BaseFeeVault` and `OperatorFeeVault`.

Their percentages for each kind of revenue are managed separately.
The contract will send the maximum amount between calculating the `grossRevenueShare` and `netRevenueShare` with their respective percentages to the `revenueShareRecipient`, and the remaining amount will be sent to the `revenueRemainderRecipient`.

The `FeeSplitter` MUST be proxied and initializable only by the `ProxyAdmin.owner()`.

### Constants

| Name                            | Value    |
| ------------------------------- | -------- |
| `BASIS_POINT_SCALE`             | 10000    |
| `MIN_FEE_DISBURSEMENT_INTERVAL` | 24 hours |

### Functions

#### `initialize`

Initializes the contract with the initial recipients, fee shares, and disbursement interval.

```solidity
function initialize(
        address payable _revenueShareRecipient,
        address payable _revenueRemainderRecipient,
        uint40 _feeDisbursementInterval,
        uint16 _netFeeShareBP,
        uint16 _grossFeeShareBP
    ) external
```

- MUST only be callable once.
- MUST only be callable by `ProxyAdmin.owner()`.
- MUST revert if `_revenueShareRecipient` is the zero address.
- MUST revert if `_revenueRemainderRecipient` is the zero address.
- MUST revert if `_feeDisbursementInterval` is less than `MIN_FEE_DISBURSEMENT_INTERVAL`.
- MUST revert if `_netFeeShareBP` exceeds `BASIS_POINT_SCALE`.
- MUST revert if `_grossFeeShareBP` exceeds `BASIS_POINT_SCALE`.
- MUST set `revenueShareRecipient` to `_revenueShareRecipient`.
- MUST set `revenueRemainderRecipient` to `_revenueRemainderRecipient`.
- MUST set `feeDisbursementInterval` to `_feeDisbursementInterval`.
- MUST set `netFeeShareBP` to `_netFeeShareBP`.
- MUST set `grossFeeShareBP` to `_grossFeeShareBP`.
- MUST emit an `Initialized` event with the provided parameters.

#### `disburseFees`

Initiates the routing flow by withdrawing the fees that each of the fee vaults has collected and sends the shares
to the appropriate addresses according to the configured percentage.
The function MUST revert if the withdrawal is not set to `WithdrawalNetwork.L2`, or if the recipient set is not the `FeeSplitter`.
The function MUST withdraw only if the vault balance is greater than or equal to its minimum withdrawal amount.

When attempting to withdraw from the vaults, it will check that the withdrawal network is set to `WithdrawalNetwork.L2`, and that the recipient of the vault is the `FeeSplitter`. It MUST revert if any of these conditions is not met.
It MUST only withdraw if the vault balance is greater than or equal to its minimum withdrawal amount.

```solidity
function disburseFees() external
```

- MUST revert if not enough time has passed since the last successful execution.
- MUST revert if any vault has a recipient different from this contract.
- MUST revert if any vault has a withdrawal network different from `WithdrawalNetwork.L2`.
- MUST withdraw vault's fees balance if the vault's balance is equal or greater than the min amount set.
- MUST set the `lastDisbursementTime` to the current block timestamp.
- MUST reset the `netRevenueShare` state variable.
- MUST send the max between `grossRevenueShare` and `netRevenueShare` to the `revenueShareRecipient`.
- MUST send the `grossRevenue` minus the amount sent to the `revenueShareRecipient` to the `revenueRemainderRecipient`.
- MUST emit `NoFeesCollected` event if there are no funds available in the contract after the vaults have been withdrawn.
- MUST emit `FeesDisbursed` event if the funds were disbursed.
- The balance of the contract MUST be 0 after a successful execution.

#### `receive`

Receives ETH from any sender, but only accounts for `netRevenueShare` if the sender is either the `SequencerFeeVault`, `BaseFeeVault` or `OperatorFeeVault`.

This function is virtual to allow for overrides in the derived contracts, in case some other custom logic is needed for receiving or accounting the fees.

```solidity
function receive() external payable virtual
```

- MUST add the received amount to the `netRevenueShare` balance if the sender is either the `SequencerFeeVault`, `BaseFeeVault` or `OperatorFeeVault`.
- MUST accept ETH from any sender.
- MUST emit a `FeesReceived` event upon successful execution.

#### `setRevenueShareRecipient`

Sets the address that should receive the configured share of fees.

```solidity
function setRevenueShareRecipient(address _newRevenueShareRecipient) external
```

- MUST revert if `_newRevenueShareRecipient` is the zero address.
- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit a `RevenueShareRecipientUpdated` event upon successful execution.

#### `setNetFeeShareBP`

Sets the share percentage that the net revenue share recipient should receive, in case the `netRevenueShare` is greater than the `grossRevenueShare`.

```solidity
function setNetFeeShareBP(uint256 _newNetFeeShareBP) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST revert if `_newNetFeeShareBP` exceeds `BASIS_POINT_SCALE`.
- MUST emit a `NetFeeShareBPUpdated` event upon successful execution.

#### `setGrossFeeShareBP`

Sets the share percentage that revenue remainder recipient should receive, in case the `grossRevenueShare` is greater than the `netRevenueShare`.

```solidity
function setGrossFeeShareBP(uint256 _newGrossFeeShareBP) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST revert if `_newGrossFeeShareBP` exceeds `BASIS_POINT_SCALE`.
- MUST emit a `GrossFeeShareBPUpdated` event upon successful execution.

#### `setRevenueRemainderRecipient`

Sets the address that should receive the remaining fees.

```solidity
function setRevenueRemainderRecipient(address _newRevenueRemainderRecipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`
- MUST revert if `_newRevenueRemainderRecipient` is the zero address
- MUST emit a `RevenueRemainderRecipientUpdated` event upon successful execution.

#### `setFeeDisbursementInterval`

Sets the minimum time, in seconds, that must pass between consecutive calls to `disburseFees`.

```solidity
function setFeeDisbursementInterval(uint40 _newInterval) external
```

- MUST revert if `_newInterval` is less than `MIN_FEE_DISBURSEMENT_INTERVAL`.
- MUST only be callable by `ProxyAdmin.owner()`
- MUST emit a `FeeDisbursementIntervalUpdated` event upon successful execution.

### Events

#### `FeesDisbursed`

Emitted when fees are successfully withdrawn from fee vaults and distributed to recipients.

```solidity
event FeesDisbursed(
        uint256 disbursementTime, uint256 revenueShareRecipientAmount, uint256 revenueRemainderRecipientAmount, uint256 totalFeesDisbursed
    );
```

#### `NoFeesCollected`

Emitted when `disburseFees` is called and, after attempting eligible vault withdrawals, there are no funds available to withdraw or distribute. This can occur when all vaults lack withdrawable funds or when the contract holds no ETH from any other sender.

```solidity
event NoFeesCollected()
```

#### `FeesReceived`

Emitted when the contract receives balance.

```solidity
event FeesReceived(address indexed sender, uint256 amount)
```

#### `NetFeeShareBPUpdated`

Emitted when the net fee share basis points are updated.

```solidity
event NetFeeShareBPUpdated(uint16 oldNetFeeShareBP, uint16 newNetFeeShareBP)
```

#### `GrossFeeShareBPUpdated`

Emitted when the gross fee share basis points are updated.

```solidity
event GrossFeeShareBPUpdated(uint16 oldGrossFeeShareBP, uint16 newGrossFeeShareBP)
```

#### `Initialized`

Emitted when the contract is initialized with its initial configuration.

```solidity
event Initialized(
        address revenueShareRecipient,
        address revenueRemainderRecipient,
        uint40 feeDisbursementInterval,
        uint16 netFeeShareBP,
        uint16 grossFeeShareBP
    )
```

#### `RevenueShareRecipientUpdated`

Emitted when the revenue share recipient is successfully updated.

```solidity
event RevenueShareRecipientUpdated(address indexed oldRevenueShareRecipient, address indexed newRevenueShareRecipient)
```

#### `RevenueRemainderRecipientUpdated`

Emitted when the revenue remainder recipient is successfully updated.

```solidity
event RevenueRemainderRecipientUpdated(address indexed oldRevenueRemainderRecipient, address indexed newRevenueRemainderRecipient)
```

#### `FeeDisbursementIntervalUpdated`

Emitted when the minimum time interval between consecutive fee disbursements is successfully updated.

```solidity
event FeeDisbursementIntervalUpdated(uint256 oldFeeDisbursementInterval, uint256 newFeeDisbursementInterval)
```

## Security Considerations

Given that vault recipients can now be updated, it's important to ensure that this can only be done by the appropriate address,
namely `ProxyAdmin.owner()`.
