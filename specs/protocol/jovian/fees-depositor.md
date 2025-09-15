# FeesDepositor

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Summary](#summary)
- [Functions](#functions)
  - [`receive`](#receive)
  - [`setMinDepositAmount`](#setmindepositamount)
  - [`setRecipient`](#setrecipient)
- [Events](#events)
  - [`DepositInitiated`](#depositinitiated)
  - [`MinDepositAmountUpdated`](#mindepositamountupdated)
  - [`RecipientUpdated`](#recipientupdated)
  - [`FundsReceived`](#fundsreceived)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

An optional periphery contract on L1 that acts as a recipient of the fees sent by the `L1Withdrawer` contract on L2. Its purpose is to perform a deposit transaction to OP Mainnet with those fees via the `OptimismPortal` once it has received enough funds.

## Functions

### `receive`

Initiates the deposit transaction process to OP Mainnet if and only if the contract holds funds equal to or above the `minDepositAmount` threshold.

```solidity
receive() external payable
```

- MUST initiate a deposit transaction to the recipient on OP Mainnet if the `minDepositAmount` threshold is reached.
- MUST emit the `FundsReceived` event with the sender, amount received, and the current balance of the contract.
- MUST emit the `DepositInitiated` event only if the threshold is reached.

### `setMinDepositAmount`

Updates the minimum deposit amount the contract must hold before the deposit process can be initiated.

```solidity
function setMinDepositAmount(uint256 _newMinDepositAmount) external
```

- MUST only be callable by `ProxyAdmin.owner()`.
- MUST emit the `MinDepositAmountUpdated` event.
- MUST update the `minDepositAmount` storage variable.

### `setRecipient`

Updates the address that will receive the funds on OP Mainnet during the deposit process.

```solidity
function setRecipient(address _newRecipient) external
```

- MUST only be callable by `ProxyAdmin.owner()`.
- MUST emit the `RecipientUpdated` event.
- MUST update the `recipient` storage variable.

## Events

### `DepositInitiated`

Emitted when a deposit to OP Mainnet is initiated.

```solidity
event DepositInitiated(address indexed recipient, uint256 amount)
```

### `MinDepositAmountUpdated`

Emitted when the minimum deposit amount before the deposit process can be initiated is updated.

```solidity
event MinDepositAmountUpdated(uint256 oldMinDepositAmount, uint256 newMinDepositAmount)
```

### `RecipientUpdated`

Emitted when the recipient of the funds on OP Mainnet is updated.

```solidity
event RecipientUpdated(address oldRecipient, address newRecipient)
```

### `FundsReceived`

Emitted whenever funds are received.

```solidity
event FundsReceived(address indexed sender, uint256 amount, uint256 newBalance)
```
