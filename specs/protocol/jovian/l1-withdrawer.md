# L1Withdrawer

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Summary](#summary)
- [Functions](#functions)
  - [`receive`](#receive)
  - [`setMinWithdrawalAmount`](#setminwithdrawalamount)
  - [`setRecipient`](#setrecipient)
  - [`setWithdrawalGasLimit`](#setwithdrawalgaslimit)
  - [`setWithdrawalData`](#setwithdrawaldata)
- [Events](#events)
  - [`WithdrawalInitiated`](#withdrawalinitiated)
  - [`MinWithdrawalAmountUpdated`](#minwithdrawalamountupdated)
  - [`RecipientUpdated`](#recipientupdated)
  - [`WithdrawalGasLimitUpdated`](#withdrawalgaslimitupdated)
  - [`WithdrawalDataUpdated`](#withdrawaldataupdated)
  - [`FundsReceived`](#fundsreceived)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

An optional periphery contract designed to be used as a recipient for a portion of the shares sent
by the `FeeSplitter`. Its sole purpose is to initiate a withdrawal to L1 via `L2ToL1MessagePasser.initiateWithdrawal`
once it has received enough funds.

## Functions

### `receive`

Initiates the withdrawal process to L1 once the contract holds funds equal to or above the `minWithdrawalAmount` threshold.

```solidity
receive() external payable
```

- MUST start the withdrawal process ONLY if the funds in the contract are equal to or above the minimum threshold.
- MUST emit the `WithdrawalInitiated` event.
- MUST emit the `FundsReceived` event if funds are received but no withdrawal is initiated.

### `setMinWithdrawalAmount`

Updates the minimum withdrawal amount the contract must hold before the withdrawal process can be initiated.

```solidity
function setMinWithdrawalAmount(uint256 _newMinWithdrawalAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`.
- MUST emit the `MinWithdrawalAmountUpdated` event.
- MUST update the `minWithdrawalAmount` storage variable.

### `setRecipient`

Updates the address that will receive the funds on L1 during the withdrawal process.

```solidity
function setRecipient(address _newRecipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`.
- MUST emit the `RecipientUpdated` event.
- MUST update the `recipient` storage variable.

### `setWithdrawalGasLimit`

Updates the gas limit for the withdrawal on L1.

```solidity
function setWithdrawalGasLimit(uint256 _newWithdrawalGasLimit) external
```

- MUST only be callable by `ProxyAdmin.owner()`.
- MUST emit the `WithdrawalGasLimitUpdated` event.
- MUST update the `withdrawalGasLimit` storage variable.

### `setWithdrawalData`

Updates the additional calldata sent to the funds recipient on L1.

```solidity
function setWithdrawalData(bytes memory _newWithdrawalData) external
```

- MUST only be callable by `ProxyAdmin.owner()`.
- MUST emit the `WithdrawalDataUpdated` event.
- MUST update the `withdrawalData` storage variable.

## Events

### `WithdrawalInitiated`

Emitted when a withdrawal to L1 is initiated.

```solidity
event WithdrawalInitiated(address indexed recipient, uint256 amount)
```

### `MinWithdrawalAmountUpdated`

Emitted when the minimum withdrawal amount before the withdrawal can be initiated is updated.

```solidity
event MinWithdrawalAmountUpdated(uint256 oldMinWithdrawalAmount, uint256 newMinWithdrawalAmount)
```

### `RecipientUpdated`

Emitted when the recipient of the funds on L1 is updated.

```solidity
event RecipientUpdated(address oldRecipient, address newRecipient)
```

### `WithdrawalGasLimitUpdated`

Emitted when the withdrawal gas limit on L1 is updated.

```solidity
event WithdrawalGasLimitUpdated(uint256 oldWithdrawalGasLimit, uint256 newWithdrawalGasLimit)
```

### `WithdrawalDataUpdated`

Emitted when the additional calldata sent as part of the withdrawal is updated.

```solidity
event WithdrawalDataUpdated(bytes oldWithdrawalData, bytes newWithdrawalData)
```

### `FundsReceived`

Emitted whenever funds are received but the balance in the contract is below the withdrawal threshold.

```solidity
event FundsReceived(address indexed sender, uint256 amount, uint256 newBalance)
```
