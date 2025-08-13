# Custom Gas Token Bridges

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Overview](#overview)
- [Token Flow](#token-flow)
- [L1CGTBridge](#l1cgtbridge)
  - [Interface](#interface)
  - [Functions](#functions)
  - [Events](#events)
- [L1CGTBridgeWithLegacyWithdrawal](#l1cgtbridgewithlegacywithdrawal)
  - [Additional Interface](#additional-interface)
  - [Legacy Withdrawal Support](#legacy-withdrawal-support)
  - [Additional Functions](#additional-functions)
- [L2CGTBridge](#l2cgtbridge)
  - [Functions](#functions-1)
  - [Events](#events-1)
  - [Invariants](#invariants)
- [Bridge Communication](#bridge-communication)
- [Security Considerations](#security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The Custom Gas Token (CGT) bridges enable bidirectional transfers between a specific L1 ERC20 token (designated as the chain's custom gas token) and L2 native assets on chains using Custom Gas Token mode. These bridges are purpose-built to handle the conversion between the chain's designated L1 ERC20 gas token and the L2's native gas-paying asset, providing seamless bridging functionality for custom gas token chains.

The CGT bridge system consists of:

- **L1CGTBridge**: Deployed on L1, manages locking/unlocking of the designated ERC20 token
- **L2CGTBridge**: Deployed on L2 via factory deployment from L1, manages minting/burning of native assets

Both bridges are deployed from a factory on L1 and communicate exclusively through their respective CrossDomainMessenger contracts, maintaining a trusted relationship where only messages from the counterpart bridge are accepted.

## Token Flow

**L1 → L2 Flow (ERC20 to Native Asset):**

1. User calls `bridgeCGT()` on L1CGTBridge to transfer the specified amount of the designated ERC20 for bridging
2. L1CGTBridge locks ERC20 tokens and sends a cross-domain message to L2CGTBridge via L1CrossDomainMessenger
3. L2CGTBridge receives the cross-domain message, calls `LiquidityController.mint()` to unlock native assets
4. Native assets are sent to recipient address on L2

**L2 → L1 Flow (Native Asset to ERC20):**

1. User calls `bridgeCGT()` on L2CGTBridge with native assets (msg.value)
2. L2CGTBridge calls `LiquidityController.burn()` to deposit native assets into NativeAssetLiquidity
3. L2CGTBridge sends a cross-domain message to L1CGTBridge via L2CrossDomainMessenger
4. L1CGTBridge receives the cross-domain message and unlocks ERC20 tokens to recipient address on L1

## L1CGTBridge

The L1CGTBridge contract is deployed individually for each L2 chain using Custom Gas Token mode. It manages the L1 side of CGT bridging operations.

### Interface

```solidity
interface IL1CGTBridge {
    // Events
    event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount);
    event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount);

    // Core bridge functions
    function bridgeCGT(address _to, uint256 _amount, uint32 _minGasLimit) external;
    function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external;

    // Configuration getters
    function cgtToken() external view returns (address);
    function otherBridge() external view returns (address);
    function messenger() external view returns (address);
}
```

### Functions

#### `bridgeCGT`

Initiates a CGT transfer from L1 to L2 to a specified recipient.

```solidity
function bridgeCGT(address _to, uint256 _amount, uint32 _minGasLimit) external
```

- MUST transfer `_amount` of CGT ERC20 tokens from `msg.sender` to the bridge contract
- MUST send a message to L2CGTBridge via CrossDomainMessenger to mint equivalent native assets to `_to`
- MUST emit `CGTBridgeInitiated` event
- MUST revert if `_to` is zero address
- MUST revert if `_amount` is zero
- MUST revert if caller has insufficient token balance or allowance

#### `finalizeBridgeCGT`

Finalizes a CGT transfer from L2 to L1 by unlocking tokens.

```solidity
function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external
```

- MUST transfer `_amount` of CGT ERC20 tokens from bridge contract to `_to` address
- MUST emit `CGTBridgeFinalized` event
- MUST revert if called by any address other than the CrossDomainMessenger
- MUST revert if the CrossDomainMessenger's `xDomainMessageSender()` is not the L2CGTBridge address
- MUST revert if bridge has insufficient token balance

### Events

#### `CGTBridgeInitiated`

Emitted when a CGT bridge transfer is initiated from L1 to L2.

```solidity
event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount)
```

#### `CGTBridgeFinalized`

Emitted when a CGT bridge transfer from L2 to L1 is finalized on L1.

```solidity
event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount)
```


## L1CGTBridgeWithLegacyWithdrawal

The `L1CGTBridgeWithLegacyWithdrawal` is an extension of the `L1CGTBridge` contract that includes additional functionality for handling legacy withdrawals from before CGT migration, bridge management controls, and trusted state management. This extension is deployed when legacy withdrawal support is required.

### Additional Interface

```solidity
interface IL1CGTBridgeWithLegacyWithdrawal is IL1CGTBridge {
    // Additional events
    event WithdrawalProven(bytes32 indexed withdrawalHash, address indexed from, address indexed to);
    event WithdrawalProvenExtension1(bytes32 indexed withdrawalHash, address indexed proofSubmitter);
    event WithdrawalFinalized(bytes32 indexed withdrawalHash, bool success);

    // Legacy withdrawal functions (for CGT migration)
    function legacyProveWithdrawalTransaction(Types.WithdrawalTransaction memory _tx, bytes[] calldata _withdrawalProof) external;
    function legacyFinalizeWithdrawalTransaction(Types.WithdrawalTransaction memory _tx) external;

    // Trusted state management (migration only)
    function setTrustedStateOnce(bytes32 trustedMessagePasserStorageRoot) external;

    // Bridge management
    function enableDeposits() external;
    function disableDeposits() external;
    function enableWithdrawals() external;
    function disableWithdrawals() external;

    // Configuration getter
    function trustedMessagePasserStorageRoot() external view returns (bytes32);
}
```

### Legacy Withdrawal Support

The L1CGTBridgeWithLegacyWithdrawal provides functionality to handle legacy withdrawals initiated before CGT migration. These functions use a trusted storage root to prove and finalize withdrawals containing native asset transfers (`_tx.value > 0`) that cannot be processed through the standard OptimismPortal after migration.

### Additional Functions

#### `setTrustedStateOnce`

Sets the trusted L2ToL1MessagePasser storage root for legacy withdrawal verification (migration only).

```solidity
function setTrustedStateOnce(bytes32 trustedMessagePasserStorageRoot) external
```

- MUST revert if called by any address other than the contract owner
- MUST only be callable once (subsequent calls MUST revert)
- MUST store the provided storage root for legacy withdrawal proofs
- MUST be set to a valid L2ToL1MessagePasser storage root from after the CGT migration
- MUST emit appropriate event when set

#### `legacyProveWithdrawalTransaction`

Proves a legacy withdrawal transaction using the trusted storage root (migration only).

```solidity
function legacyProveWithdrawalTransaction(Types.WithdrawalTransaction memory _tx, bytes[] calldata _withdrawalProof) external
```

- MUST revert if `trustedMessagePasserStorageRoot` has not been set
- MUST compute withdrawal hash using `Hashing.hashWithdrawal(_tx)`
- MUST verify inclusion proof using `SecureMerkleTrie.verifyInclusionProof()` against trusted root
- MUST store proven withdrawal in `provenLegacyWithdrawals[withdrawalHash][msg.sender]`
- MUST emit `WithdrawalProven` and `WithdrawalProvenExtension1` events
- MUST revert if proof is invalid

#### `legacyFinalizeWithdrawalTransaction`

Finalizes a proven legacy withdrawal transaction (migration only).

```solidity
function legacyFinalizeWithdrawalTransaction(Types.WithdrawalTransaction memory _tx) external
```

- MUST revert if withdrawal has not been previously proven
- MUST verify withdrawal has not been finalized in OptimismPortal or L1CGTBridge
- MUST only finalize withdrawals with `_tx.value > 0` (native asset withdrawals)
- MUST transfer ERC20 tokens equivalent to `_tx.value` to `_tx.target`
- MUST execute `_tx.data` if present (with gas limit checks)
- MUST mark withdrawal as finalized to prevent replay
- MUST revert if `_tx.target` is the CGT token contract address
- MUST emit `WithdrawalFinalized` event

#### `enableDeposits` / `disableDeposits`

Enables or disables deposit functionality for emergency control.

```solidity
function enableDeposits() external
function disableDeposits() external
```

- MUST revert if called by any address other than the contract owner
- MUST pause/unpause deposit functions (`bridgeCGT`)
- MUST emit appropriate events when state changes

#### `enableWithdrawals` / `disableWithdrawals`

Enables or disables withdrawal functionality for emergency control.

```solidity
function enableWithdrawals() external
function disableWithdrawals() external
```

- MUST revert if called by any address other than the contract owner
- MUST pause/unpause withdrawal functions (`finalizeBridgeCGT`, legacy functions)
- MUST emit appropriate events when state changes

#### `trustedMessagePasserStorageRoot`

Returns the trusted L2ToL1MessagePasser storage root used for legacy withdrawal verification.

```solidity
function trustedMessagePasserStorageRoot() external view returns (bytes32)
```

- MUST return the stored trusted storage root
- MUST return zero if not set

## L2CGTBridge

The `L2CGTBridge` contract is deployed on L2 via a factory deployment initiated from L1. It handles the L2 side of Custom Gas Token bridging operations, enabling bidirectional transfers between L1 ERC20 tokens and L2 native assets. This contract is deployed on chains using Custom Gas Token mode and has minting authorization from the `LiquidityController` to manage native asset supply.

The bridge maintains a trusted relationship with its L1 counterpart (`L1CGTBridge`) and only accepts cross-domain messages from the designated L1 bridge address through the `L2CrossDomainMessenger`.

### Interface

```solidity
interface IL2CGTBridge {
    // Events
    event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount);
    event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount);

    // Core bridge functions
    function bridgeCGT(address _to, uint32 _minGasLimit) external payable;
    function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external;

    // Configuration getters
    function otherBridge() external view returns (address);
    function messenger() external view returns (address);
    function liquidityController() external view returns (address);
}
```

### Functions

#### `bridgeCGT`

Initiates a CGT transfer from L2 to L1 to a specified recipient address.

```solidity
function bridgeCGT(address _to, uint32 _minGasLimit) external payable
```

- MUST accept native assets via `msg.value` (the amount being bridged)
- MUST call `LiquidityController.burn{value: msg.value}()` to deposit native assets into NativeAssetLiquidity contract
- MUST send cross-domain message to L1CGTBridge via `L2CrossDomainMessenger` to unlock equivalent ERC20 tokens to `_to`
- MUST pass `msg.value` as the `_amount` parameter in the cross-domain message for L1 finalization
- MUST emit `CGTBridgeInitiated` event with `msg.sender`, `_to`, and `msg.value`
- MUST revert if `_to` is zero address
- MUST revert if `msg.value` is zero

#### `finalizeBridgeCGT`

Finalizes a CGT transfer from L1 to L2 by minting native assets to the recipient.

```solidity
function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external
```

- MUST call `LiquidityController.mint(_to, _amount)` to mint native assets to recipient
- MUST emit `CGTBridgeFinalized` event
- MUST revert if called by any address other than the `L2CrossDomainMessenger`
- MUST revert if the `CrossDomainMessenger.xDomainMessageSender()` is not the authorized L1CGTBridge address
- MUST revert if `LiquidityController.mint()` operation fails

### Events

#### `CGTBridgeInitiated`

Emitted when a CGT bridge transfer is initiated from L2 to L1.

```solidity
event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount)
```

Where `from` is the L2 sender, `to` is the L1 recipient, and `amount` is the native asset amount being bridged.

#### `CGTBridgeFinalized`

Emitted when a CGT bridge transfer from L1 to L2 is finalized on L2.

```solidity
event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount)
```

Where `from` is the L1 sender, `to` is the L2 recipient, and `amount` is the native asset amount minted.

### Invariants

- Only the `L2CrossDomainMessenger` can call `finalizeBridgeCGT()` when relaying messages from the authorized L1CGTBridge
- The L2CGTBridge MUST be authorized as a minter in the `LiquidityController` before any bridging operations
- All L2→L1 transfers immediately burn native assets by depositing them into `NativeAssetLiquidity`
- All L1→L2 transfers mint native assets by withdrawing them from `NativeAssetLiquidity` via `LiquidityController`
- Native asset supply on L2 is backed 1:1 by locked ERC20 tokens on L1
- Cross-domain message authentication ensures only trusted L1CGTBridge can trigger finalization

## Bridge Communication

Both bridges maintain a trusted relationship and can only accept finalization messages from their designated counterpart:

**L1CGTBridge Configuration:**

- `otherBridge`: Address of the corresponding L2CGTBridge (deployed via factory with deterministic addressing)
- `messenger`: Address of the L1CrossDomainMessenger
- `cgtToken`: Address of the ERC20 token being bridged (immutable, set at deployment)

**L2CGTBridge Configuration:**

- `otherBridge`: Address of the corresponding L1CGTBridge (deployed via factory with deterministic addressing)
- `messenger`: Address of the L2CrossDomainMessenger
- `liquidityController`: Address of the LiquidityController predeploy

The bridges enforce cross-domain message authentication by:

1. Verifying calls to `finalizeBridgeCGT()` come from the CrossDomainMessenger
2. Verifying the `xDomainMessageSender()` matches the expected counterpart bridge address
3. Only processing messages that originate from the trusted counterpart bridge

## Security Considerations

**Access Control:**

- L2CGTBridge MUST be authorized as a minter in the LiquidityController before any L1→L2 transfers can be completed
- Only the designated L1CGTBridge can send finalization messages to L2CGTBridge
- Only the L2CGTBridge can send finalization messages to L1CGTBridge

**Token Safety:**

- L1CGTBridge holds locked ERC20 tokens as collateral for minted L2 native assets
- Native assets on L2 are backed 1:1 by locked ERC20 tokens on L1
- Burns on L2 immediately deposit assets into NativeAssetLiquidity, reducing circulating supply

**Bridge Integrity:**

- Failed cross-domain messages can be replayed through the standard message relay mechanisms
- Bridge contracts SHOULD implement pausability for emergency situations
- Both bridges MUST validate all message parameters to prevent invalid minting or unlocking operations

**Legacy Withdrawal Security (L1CGTBridgeWithLegacyWithdrawal only):**

- `setTrustedStateOnce()` MUST revert if called by any address other than contract owner and MUST only be callable once
- `trustedMessagePasserStorageRoot` MUST represent valid post-migration L2ToL1MessagePasser state
- Legacy finalization MUST check both OptimismPortal and L1CGTBridge to prevent replays
- Legacy finalization MUST reject `_tx.target == cgtToken` to prevent approval attacks
- Legacy withdrawals MUST only process transactions with `_tx.value > 0`
- Proof verification MUST use SecureMerkleTrie with same standards as OptimismPortal