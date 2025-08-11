# Bridges

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Overview](#overview)
- [Custom Gas Token Bridges](#custom-gas-token-bridges)
- [Token Flow](#token-flow)
- [L1CGTBridge](#l1cgtbridge)
  - [Interface](#interface)
  - [Functions](#functions)
  - [Events](#events)
- [L2CGTBridge](#l2cgtbridge)
- [Bridge Communication](#bridge-communication)
- [Security Considerations](#security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

ETH bridging functions MUST revert when Custom Gas Token mode is enabled and the function involves ETH transfers. This revert behavior is necessary because when a chain operates in Custom Gas Token mode, ETH is no longer the native asset used for gas fees and transactions. The chain has shifted to using a different native asset entirely. Allowing ETH transfers could create confusion about which asset serves as the native currency, potentially leading to user errors and lost funds. Additionally, the custom gas token's supply is managed independently through dedicated contracts (`NativeAssetLiquidity` and `LiquidityController`), and combining ETH bridging with custom gas token operations introduces additional complexity to supply management and accounting.

## Custom Gas Token Bridges

The Custom Gas Token (CGT) bridges enable bidirectional transfers between L1 ERC20 tokens and L2 native assets on chains using Custom Gas Token mode. Unlike standard bridges that handle ETH and generic ERC20 tokens, CGT bridges are specifically designed to convert between a designated L1 ERC20 token and the L2's native gas-paying asset.

The CGT bridge system consists of:

- **L1CGTBridge**: Deployed on L1, manages locking/unlocking of the designated ERC20 token
- **L2CGTBridge**: Predeploy on L2 at `0x420000000000000000000000000000000000002B`, manages minting/burning of native assets

Both bridges communicate exclusively through their respective CrossDomainMessenger contracts and maintain a trusted relationship where only messages from the counterpart bridge are accepted.

## Token Flow

**L1 → L2 Flow (ERC20 to Native Asset):**

1. User calls `bridgeCGTTo()` on L1CGTBridge with ERC20 tokens
2. L1CGTBridge locks ERC20 tokens and sends message to L2CGTBridge via L1CrossDomainMessenger
3. L2CGTBridge receives message, calls `LiquidityController.mint()` to unlock native assets
4. Native assets are sent to recipient address on L2

**L2 → L1 Flow (Native Asset to ERC20):**

1. User calls `bridgeCGTTo()` on L2CGTBridge with native assets (msg.value)
2. L2CGTBridge calls `LiquidityController.burn()` to deposit native assets into NativeAssetLiquidity
3. L2CGTBridge sends message to L1CGTBridge via L2CrossDomainMessenger
4. L1CGTBridge receives message and unlocks ERC20 tokens to recipient address on L1

## Legacy Withdrawal Support

The L1CGTBridge provides functionality to handle legacy withdrawals initiated before CGT migration. These functions use a trusted storage root to prove and finalize withdrawals containing native asset transfers (`_tx.value > 0`) that cannot be processed through the standard OptimismPortal after migration.

## L1CGTBridge

The L1CGTBridge contract is deployed individually for each L2 chain using Custom Gas Token mode. It manages the L1 side of CGT bridging operations.

### Interface

```solidity
interface IL1CGTBridge {
    // Events
    event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount, bytes extraData);
    event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount, bytes extraData);
    event WithdrawalProven(bytes32 indexed withdrawalHash, address indexed from, address indexed to);
    event WithdrawalProvenExtension1(bytes32 indexed withdrawalHash, address indexed proofSubmitter);
    event WithdrawalFinalized(bytes32 indexed withdrawalHash, bool success);

    // Core bridge functions
    function bridgeCGT(uint256 _amount, uint32 _minGasLimit, bytes calldata _extraData) external;
    function bridgeCGTTo(address _to, uint256 _amount, uint32 _minGasLimit, bytes calldata _extraData) external;
    function finalizeBridgeCGT(address _from, address _to, uint256 _amount, bytes calldata _extraData) external;

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

    // Configuration getters
    function cgtToken() external view returns (address);
    function otherBridge() external view returns (address);
    function messenger() external view returns (address);
    function trustedMessagePasserStorageRoot() external view returns (bytes32);
}
```

### Functions

#### `bridgeCGT`

Initiates a CGT transfer from L1 to L2 for the caller.

```solidity
function bridgeCGT(uint256 _amount, uint32 _minGasLimit, bytes calldata _extraData) external
```

- MUST transfer `_amount` of CGT ERC20 tokens from `msg.sender` to the bridge contract
- MUST send a message to L2CGTBridge via CrossDomainMessenger to mint equivalent native assets to `msg.sender`
- MUST emit `CGTBridgeInitiated` event
- MUST revert if `_amount` is zero
- MUST revert if caller has insufficient token balance or allowance

#### `bridgeCGTTo`

Initiates a CGT transfer from L1 to L2 to a specified recipient.

```solidity
function bridgeCGTTo(address _to, uint256 _amount, uint32 _minGasLimit, bytes calldata _extraData) external
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
function finalizeBridgeCGT(address _from, address _to, uint256 _amount, bytes calldata _extraData) external
```

- MUST only be callable by the CrossDomainMessenger when relaying a message from L2CGTBridge
- MUST transfer `_amount` of CGT ERC20 tokens from bridge contract to `_to` address
- MUST emit `CGTBridgeFinalized` event
- MUST revert if called by any address other than the CrossDomainMessenger
- MUST revert if the CrossDomainMessenger's `xDomainMessageSender()` is not the L2CGTBridge address
- MUST revert if bridge has insufficient token balance

#### `setTrustedStateOnce`

Sets the trusted L2ToL1MessagePasser storage root for legacy withdrawal verification (migration only).

```solidity
function setTrustedStateOnce(bytes32 trustedMessagePasserStorageRoot) external
```

- MUST only be callable by the contract owner
- MUST only be callable once (subsequent calls MUST revert)
- MUST store the provided storage root for legacy withdrawal proofs
- MUST be set to a valid L2ToL1MessagePasser storage root from after the CGT migration
- MUST emit appropriate event when set

#### `legacyProveWithdrawalTransaction`

Proves a legacy withdrawal transaction using the trusted storage root (migration only).

```solidity
function legacyProveWithdrawalTransaction(Types.WithdrawalTransaction memory _tx, bytes[] calldata _withdrawalProof) external
```

- MUST only be callable when `trustedMessagePasserStorageRoot` has been set
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

- MUST only be callable for previously proven withdrawals
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

- MUST only be callable by the contract owner
- MUST pause/unpause deposit functions (`bridgeCGT`, `bridgeCGTTo`)
- MUST emit appropriate events when state changes

#### `enableWithdrawals` / `disableWithdrawals`

Enables or disables withdrawal functionality for emergency control.

```solidity
function enableWithdrawals() external
function disableWithdrawals() external
```

- MUST only be callable by the contract owner
- MUST pause/unpause withdrawal functions (`finalizeBridgeCGT`, legacy functions)
- MUST emit appropriate events when state changes

### Events

#### `CGTBridgeInitiated`

Emitted when a CGT bridge transfer is initiated from L1 to L2.

```solidity
event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount, bytes extraData)
```

#### `CGTBridgeFinalized`

Emitted when a CGT bridge transfer from L2 to L1 is finalized on L1.

```solidity
event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount, bytes extraData)
```

#### `WithdrawalProven`

Emitted when a legacy withdrawal is successfully proven.

```solidity
event WithdrawalProven(bytes32 indexed withdrawalHash, address indexed from, address indexed to)
```

#### `WithdrawalProvenExtension1`

Emitted when a legacy withdrawal is proven, includes proof submitter information.

```solidity
event WithdrawalProvenExtension1(bytes32 indexed withdrawalHash, address indexed proofSubmitter)
```

#### `WithdrawalFinalized`

Emitted when a legacy withdrawal is finalized.

```solidity
event WithdrawalFinalized(bytes32 indexed withdrawalHash, bool success)
```

## L2CGTBridge

The L2CGTBridge is a predeploy contract that handles the L2 side of CGT bridging operations. See the [L2CGTBridge predeploy specification](predeploys.md#l2cgtbridge) for complete interface and function definitions.

Address: `0x420000000000000000000000000000000000002B`

## Bridge Communication

Both bridges maintain a trusted relationship and can only accept finalization messages from their designated counterpart:

**L1CGTBridge Configuration:**

- `otherBridge`: Address of the L2CGTBridge predeploy (`0x420000000000000000000000000000000000002B`)
- `messenger`: Address of the L1CrossDomainMessenger
- `cgtToken`: Address of the ERC20 token being bridged (immutable, set at deployment)

**L2CGTBridge Configuration:**

- `otherBridge`: Address of the corresponding L1CGTBridge (set during initialization)
- `messenger`: Address of the L2CrossDomainMessenger predeploy
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

**Legacy Withdrawal Security:**

- `setTrustedStateOnce()` MUST only be callable once by contract owner
- `trustedMessagePasserStorageRoot` MUST represent valid post-migration L2ToL1MessagePasser state
- Legacy finalization MUST check both OptimismPortal and L1CGTBridge to prevent replays
- Legacy finalization MUST reject `_tx.target == cgtToken` to prevent approval attacks
- Legacy withdrawals MUST only process transactions with `_tx.value > 0`
- Proof verification MUST use SecureMerkleTrie with same standards as OptimismPortal
