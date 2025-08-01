# Cross Domain Messengers

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Message Passing](#message-passing)
- [Upgradability](#upgradability)
- [Message Versioning](#message-versioning)
  - [Message Version 0](#message-version-0)
  - [Message Version 1](#message-version-1)
- [Backwards Compatibility Notes](#backwards-compatibility-notes)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The cross domain messengers are responsible for providing a higher level API for
developers who are interested in sending cross domain messages. They allow for
the ability to replay cross domain messages and sit directly on top of the lower
level system contracts responsible for cross domain messaging on L1 and L2.

The `CrossDomainMessenger` is extended to create both an
`L1CrossDomainMessenger` as well as a `L2CrossDomainMessenger`.
These contracts are then extended with their legacy APIs to provide backwards
compatibility for applications that integrated before the Bedrock system
upgrade.

The `L2CrossDomainMessenger` is a predeploy contract located at
`0x4200000000000000000000000000000000000007`.

```solidity
interface CrossDomainMessenger {
    event FailedRelayedMessage(bytes32 indexed msgHash);
    event RelayedMessage(bytes32 indexed msgHash);
    event SentMessage(address indexed target, address sender, bytes message, uint256 messageNonce, uint256 gasLimit);

    function MESSAGE_VERSION() external view returns (uint16);
    function MIN_GAS_CALLDATA_OVERHEAD() external view returns (uint64);
    function MIN_GAS_DYNAMIC_OVERHEAD_DENOMINATOR() external view returns (uint64);
    function MIN_GAS_DYNAMIC_OVERHEAD_NUMERATOR() external view returns (uint64);
    function OTHER_MESSENGER() external view returns (address);
    function RELAY_CALL_OVERHEAD() external view returns (uint64);
    function RELAY_CONSTANT_OVERHEAD() external view returns (uint64);
    function RELAY_GAS_CHECK_BUFFER() external view returns (uint64);
    function RELAY_RESERVED_GAS() external view returns (uint64);
    function baseGas(bytes calldata _message, uint32 _minGasLimit) external pure returns (uint64);
    function failedMessages(bytes32) external view returns (bool);
    function messageNonce() external view returns (uint256);
    function relayMessage(uint256 _nonce, address _sender, address _target, uint256 _value, uint256 _minGasLimit, bytes calldata _message) external payable;
    function sendMessage(address _target, bytes calldata _message, uint32 _minGasLimit) external payable;
    function successfulMessages(bytes32) external view returns (bool);
    function xDomainMessageSender() external view returns (address);
}
```

## Message Passing

L1 `CrossDomainMessenger` messages are implemented as
L1 to L2 transactions by way of the `OptimismPortal`, and L2
`CrossDomainMessenger` messages are implemented as withdrawals.

Successful messages have their hash stored in the `successfulMessages` mapping
while unsuccessful messages have their hash stored in the `failedMessages`
mapping.

The `sendMessage` function MUST revert when Custom Gas Token mode is enabled and `msg.value > 0`.

The user experience when sending from L1 to L2 is a bit different than when
sending a transaction from L2 to L1. When going from L1 into L2, the user does
not need to call `relayMessage` on L2 themselves. The user pays for L2 gas on L1
and the transaction is automatically executed on L2 on the user's behalf.
However if the gas amount paid is too low, the transaction will fail on L2, but
in the case that it fails, it can be replayed.

In the case where a user is sending a transaction from L2 to L1, they still
need to call `relayMessage` on L1 after the fault proof period has ended.

## Upgradability

The cross domain messenger should be deployed behind upgradable proxies.

## Message Versioning

The cross domain messengers use a version to determine the serialization format
for sending messages. The serialization formats are used to ensure backwards
compatibility as the schemas are updated.

### Message Version 0

The first version of the messages that were sent used the following serialization
format:

```solidity
abi.encodeWithSignature(
    "relayMessage(address,address,bytes,uint256)",
    _target,
    _sender,
    _message,
    _nonce
);
```

### Message Version 1

The second version of the messages use the following serialization format:

```solidity
abi.encodeWithSignature(
    "relayMessage(uint256,address,address,uint256,uint256,bytes)",
    _nonce,
    _sender,
    _target,
    _value,
    _minGasLimit,
    _message
);
```

The version 1 schema adds value to allow for native asset to be sent along
with the message and a minimum gas limit to enforce that a minimum amount
of gas is provided for the call to be successful.

## Backwards Compatibility Notes

The `CrossDomainMessenger` is extended with legacy methods to support backwards
compatibility. Legacy methods should not be used and instead should only exist
to support applications that integrated with the system before the Bedrock
upgrade.

The legacy methods are:

- `xDomainMessageSender()`
