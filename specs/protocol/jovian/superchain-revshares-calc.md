# SuperchainRevSharesCalculator

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Summary](#summary)
- [Structs](#structs)
  - [`ShareInfo`](#shareinfo)
- [Functions](#functions)
  - [`getRecipientsAndValues`](#getrecipientsandvalues)
  - [`setShareRecipient`](#setsharerecipient)
  - [`setRemainderRecipient`](#setremainderrecipient)
- [Events](#events)
  - [`ShareRecipientUpdated`](#sharerecipientupdated)
  - [`RemainderRecipientUpdated`](#remainderrecipientupdated)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

We provide a Superchain implementation for the `ISharesCalculator` specification. It pays the greater amount
between 2.5% of gross revenue or 15% of net revenue (gross minus L1 fees) to the configured share recipient.
The second configured recipient receives the full remainder via FeeSplitter's remainder send.

It allows the `ProxyAdmin.owner` to configure the recipient address of the Superchain revenue share and the
recipient of the remainder.

## Structs

### `ShareInfo`

A struct containing both a recipient and the corresponding amount of funds for a transfer. [`getRecipientsAndValues`](#getrecipientsandvalues) uses this struct to calculate the distribution of fees.

```solidity
struct ShareInfo {
    address payable recipient;
    uint256 amount;
}
```

## Functions

### `getRecipientsAndValues`

Calculates the share for each of the two recipients based on the following formula:

```solidity
GrossRevenue = Sum of all vault fee revenues
GrossShare = GrossRevenue × 2.5%
NetShare = (GrossRevenue - L1FeeRevenue) × 15%

ShareRecipientAmount = max(GrossShare, NetShare)
RemainderRecipientAmount = GrossRevenue - ShareRecipientAmount
```

```solidity
function getRecipientsAndValues(uint256 _sequencerFeeRevenue, uint256 _baseFeeRevenue, uint256 _operatorFeeRevenue, uint256 _l1FeeRevenue) external view returns (ShareInfo[] memory shareInfo)
```

- MUST return the correct partition of shares based on the above formula.

### `setShareRecipient`

Sets the recipient of the shares calculated by the fee sharing formula.

```solidity
function setShareRecipient(address payable _shareRecipient) external
```

- MUST only be callable by `ProxyAdmin.owner`.
- MUST emit `ShareRecipientUpdated`.
- MUST update the `shareRecipient` storage variable.

### `setRemainderRecipient`

Sets the recipient of the remainder of the fees once the shares for the share recipient have been calculated.

```solidity
function setRemainderRecipient(address payable _remainderRecipient) external
```

- MUST only be callable by `ProxyAdmin.owner`.
- MUST emit `RemainderRecipientUpdated`.
- MUST update the `remainderRecipient` storage variable.

## Events

### `ShareRecipientUpdated`

Emitted when the recipient for the calculated share of the fees is updated.

```solidity
event ShareRecipientUpdated(address indexed oldShareRecipient, address indexed newShareRecipient);
```

### `RemainderRecipientUpdated`

Emitted when the recipient for the remainder of the fees is updated.

```solidity
event RemainderRecipientUpdated(address indexed oldRemainderRecipient, address indexed newRemainderRecipient);
```
