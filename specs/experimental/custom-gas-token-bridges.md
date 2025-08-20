# Custom Gas Token Bridges

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Overview](#overview)
- [Token Flow](#token-flow)
- [L1CGTBridge](#l1cgtbridge)
  - [Interface](#interface)
  - [Functions](#functions)
    - [`bridgeCGT`](#bridgecgt)
    - [`finalizeBridgeCGT`](#finalizebridgecgt)
  - [Events](#events)
    - [`CGTBridgeInitiated`](#cgtbridgeinitiated)
    - [`CGTBridgeFinalized`](#cgtbridgefinalized)
- [L1CGTBridgeWithLegacyWithdrawal](#l1cgtbridgewithlegacywithdrawal)
  - [Additional Interface](#additional-interface)
  - [Legacy Withdrawal Support](#legacy-withdrawal-support)
  - [Additional Functions](#additional-functions)
    - [`setTrustedStateOnce`](#settrustedstateonce)
    - [`legacyProveWithdrawalTransaction`](#legacyprovewithdrawaltransaction)
    - [`legacyFinalizeWithdrawalTransaction`](#legacyfinalizewithdrawaltransaction)
    - [`enableDeposits` / `disableDeposits`](#enabledeposits--disabledeposits)
    - [`enableWithdrawals` / `disableWithdrawals`](#enablewithdrawals--disablewithdrawals)
    - [`trustedMessagePasserStorageRoot`](#trustedmessagepasserstorageroot)
- [L2CGTBridge](#l2cgtbridge)
  - [Interface](#interface-1)
  - [Functions](#functions-1)
    - [`bridgeCGT`](#bridgecgt-1)
    - [`finalizeBridgeCGT`](#finalizebridgecgt-1)
  - [Events](#events-1)
    - [`CGTBridgeInitiated`](#cgtbridgeinitiated-1)
    - [`CGTBridgeFinalized`](#cgtbridgefinalized-1)
  - [Invariants](#invariants)
- [L1CGTBridgeFactory](#l1cgtbridgefactory)
  - [Interface](#interface-2)
  - [Deployment Process](#deployment-process)
  - [Functions](#functions-2)
    - [`deploy`](#deploy)
  - [Deterministic Addressing](#deterministic-addressing)
  - [Security Considerations](#security-considerations)
- [L2CGTBridgeFactory](#l2cgtbridgefactory)
  - [Interface](#interface-3)
  - [Deployment Process](#deployment-process-1)
  - [Post-Deployment Requirements](#post-deployment-requirements)
- [Bridge Communication](#bridge-communication)
- [Security Considerations](#security-considerations-1)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The Custom Gas Token (CGT) bridges enable bidirectional transfers between a specific L1 ERC20 token (designated as
the chain's custom gas token) and L2 native assets on chains using Custom Gas Token mode. These bridges are purpose-built
to handle the conversion between the chain's designated L1 ERC20 gas token and the L2's native gas-paying asset,
providing seamless bridging functionality for custom gas token chains.

The CGT bridge system consists of:

- **L1CGTBridge**: Deployed on L1 by the L1CGTBridgeFactory, manages locking/unlocking of the designated ERC20 token
- **L2CGTBridge**: Deployed on L2 via L2CGTBridgeFactory triggered by L1CGTBridgeFactory, manages minting/burning of
  native assets
- **L1CGTBridgeFactory**: Factory contract on L1 that orchestrates deployment of both bridges using deterministic
  addressing
- **L2CGTBridgeFactory**: Factory contract deployed on L2 via cross-chain message to deploy the L2CGTBridge

Both bridges are deployed through factory contracts that ensure deterministic addressing and proper configuration,
communicating exclusively through their respective CrossDomainMessenger contracts while maintaining a trusted
relationship where only messages from the counterpart bridge are accepted.

## Token Flow

**L1 → L2 Flow (ERC20 to Native Asset):**

1. User calls `bridgeCGT()` on L1CGTBridge to transfer the specified amount of the designated ERC20 for
   bridging
2. L1CGTBridge locks ERC20 tokens and sends a cross-domain message to L2CGTBridge via
   L1CrossDomainMessenger
3. L2CGTBridge receives the cross-domain message, calls `LiquidityController.mint()` to unlock native
   assets
4. Native assets are sent to recipient address on L2

**L2 → L1 Flow (Native Asset to ERC20):**

1. User calls `bridgeCGT()` on L2CGTBridge with native assets (msg.value)
2. L2CGTBridge calls `LiquidityController.burn()` to deposit native assets into
   NativeAssetLiquidity
3. L2CGTBridge sends a cross-domain message to L1CGTBridge via
   L2CrossDomainMessenger
4. L1CGTBridge receives the cross-domain message and unlocks ERC20 tokens to recipient
   address on L1

## L1CGTBridge

The L1CGTBridge contract is deployed individually for each L2 chain using Custom Gas Token mode. It manages the L1
side of CGT bridging operations.

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

The `L1CGTBridgeWithLegacyWithdrawal` is an extension of the `L1CGTBridge` contract that includes additional
functionality for handling legacy withdrawals from before CGT migration, bridge management controls, and trusted state
management. This extension is deployed when legacy withdrawal support is required.

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

The L1CGTBridgeWithLegacyWithdrawal provides functionality to handle legacy withdrawals initiated before CGT
migration. These functions use a trusted storage root to prove and finalize withdrawals containing native asset
transfers (`_tx.value > 0`) that cannot be processed through the standard OptimismPortal after migration.

### Additional Functions

#### `setTrustedStateOnce`

Sets the trusted L2ToL1MessagePasser storage root for legacy withdrawal verification (migration
only).

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

The `L2CGTBridge` contract is deployed on L2 via a factory deployment initiated from L1. It handles the L2 side of
Custom Gas Token bridging operations, enabling bidirectional transfers between L1 ERC20 tokens and L2 native assets.
This contract is deployed on chains using Custom Gas Token mode and has minting authorization from the
`LiquidityController` to manage native asset supply.

The bridge maintains a trusted relationship with its L1 counterpart (`L1CGTBridge`) and only accepts cross-domain
messages from the designated L1 bridge address through the `L2CrossDomainMessenger`.

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
- MUST revert if `LiquidityController.mint()` operation fails (this will occur if L2CGTBridge is not authorized as minter)

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
- The L2CGTBridge MUST be authorized as a minter in the `LiquidityController` before any L1→L2 bridging operations can succeed
- Authorization is granted by the LiquidityController owner via `LiquidityController.authorizeMinter(l2BridgeAddress)`
- All L2→L1 transfers immediately burn native assets by depositing them into `NativeAssetLiquidity`
- All L1→L2 transfers mint native assets by withdrawing them from `NativeAssetLiquidity` via `LiquidityController`
- Native asset supply on L2 is backed 1:1 by locked ERC20 tokens on L1
- Cross-domain message authentication ensures only trusted L1CGTBridge can trigger finalization

## L1CGTBridgeFactory

The `L1CGTBridgeFactory` is a factory contract deployed on L1 that handles the deployment of both L1CGTBridge and
L2CGTBridge contracts across L1 and L2. It uses Optimism's cross-chain deployment infrastructure to ensure
deterministic addressing and proper configuration of both bridges.

### Interface

```solidity
interface IL1CGTBridgeFactory {
    /// @notice Struct containing L2 deployment parameters
    struct L2Deployments {
        address l2BridgeOwner;           // Owner of the L2CGTBridge
        uint32 minGasLimitDeploy;        // Minimum gas limit for L2 factory deployment
    }

    /// @notice The L2 Create2Deployer address used by Optimism
    function L2_CREATE2_DEPLOYER() external view returns (address);

    /// @notice Counter to ensure unique salts for each deployment
    function deploymentsSaltCounter() external view returns (uint256);

    /// @notice Deploys L1CGTBridge and triggers L2CGTBridge deployment
    /// @param _l1Messenger The L1CrossDomainMessenger address
    /// @param _cgtToken The ERC20 token address for custom gas token
    /// @param _l1BridgeOwner The owner of the L1CGTBridge
    /// @param _l2Deployments Parameters for L2 deployment
    /// @return _l1Bridge The deployed L1CGTBridge address
    /// @return _l2Factory The L2 factory address
    /// @return _l2Bridge The precalculated L2CGTBridge address
    function deploy(
        address _l1Messenger,
        address _cgtToken,
        address _l1BridgeOwner,
        L2Deployments calldata _l2Deployments
    ) external returns (address _l1Bridge, address _l2Factory, address _l2Bridge);

    /// @notice Emitted when bridges are deployed
    event BridgesDeployed(address indexed l1Bridge, address indexed l2Factory, address indexed l2Bridge);
}
```

### Deployment Process

The factory follows this deployment sequence:

1. **Salt Generation**: Increments `deploymentsSaltCounter` to ensure unique deployment addresses
2. **Address Precalculation**: Uses CREATE nonces to precalculate L1 and L2 bridge addresses
3. **L1 Bridge Deployment**: Deploys L1CGTBridge implementation and proxy with precalculated L2 bridge address
4. **L2 Factory Deployment**: Sends cross-domain message to deploy L2CGTBridgeFactory on L2
5. **L2 Bridge Deployment**: L2 factory automatically deploys L2CGTBridge with precalculated L1 bridge address

### Functions

#### `deploy`

Deploys both L1CGTBridge and triggers L2CGTBridge deployment via cross-chain message.

```solidity
function deploy(
    address _l1Messenger,
    address _cgtToken,
    address _l1BridgeOwner,
    L2Deployments calldata _l2Deployments
) external returns (address _l1Bridge, address _l2Factory, address _l2Bridge)
```

- MUST increment `deploymentsSaltCounter` by 2 to ensure unique salt
- MUST precalculate L1CGTBridge address using CREATE nonce
- MUST precalculate L2CGTBridge address using L2 factory nonce
- MUST deploy L1CGTBridge implementation and proxy with correct configuration
- MUST send cross-domain message to deploy L2CGTBridgeFactory via L1CrossDomainMessenger
- MUST emit `BridgesDeployed` event
- MUST revert if `_cgtToken` is zero address
- MUST revert if `_l1Messenger` is zero address

### Deterministic Addressing

The factory ensures deterministic addressing through:

**L1CGTBridge Address**: Calculated using `CREATE` with factory address and current nonce
**L2CGTBridge Address**: Calculated using `CREATE` with L2 factory address and deployment nonce
**Salt Uniqueness**: Each deployment uses incremented counter as salt to ensure unique L2 factory addresses

### Security Considerations

- Factory contract SHOULD be deployed with proper access controls if needed
- Salt counter prevents address collisions across different deployments
- Cross-domain message failure can be replayed through standard Optimism retry mechanisms
- L1CGTBridge configuration includes precalculated L2CGTBridge address for trusted communication

## L2CGTBridgeFactory

The `L2CGTBridgeFactory` is deployed on L2 by the L1CGTBridgeFactory via cross-chain message. It handles the
deployment of the L2CGTBridge contract and ensures proper initialization.

### Interface

```solidity
interface IL2CGTBridgeFactory {
    /// @notice Deploys the L2CGTBridge contract
    /// @param _l1Bridge The L1CGTBridge address
    /// @param _l2BridgeOwner The owner of the L2CGTBridge
    /// @param _liquidityController The LiquidityController predeploy address
    /// @return _l2Bridge The deployed L2CGTBridge address
    function deploy(
        address _l1Bridge,
        address _l2BridgeOwner,
        address _liquidityController
    ) external returns (address _l2Bridge);

    /// @notice Emitted when L2CGTBridge is deployed
    event L2BridgeDeployed(address indexed l2Bridge, address indexed l1Bridge);
}
```

### Deployment Process

1. **Automatic Execution**: Deployed and executed automatically by L1CGTBridgeFactory cross-chain
   message
2. **L2CGTBridge Deployment**: Creates L2CGTBridge implementation and proxy
3. **Configuration**: Initializes L2CGTBridge with L1 bridge address and owner
4. **Authorization Required**: L2CGTBridge MUST be separately authorized as minter in LiquidityController via
   `authorizeMinter()` before bridging operations can function

### Post-Deployment Requirements

After the L2CGTBridge is deployed, the following authorization step is
required:

- The LiquidityController owner (L1 ProxyAdmin owner) MUST call
  `LiquidityController.authorizeMinter(l2BridgeAddress)` to grant minting permissions
- Without this authorization, all L1→L2 bridging operations will fail when `finalizeBridgeCGT()` attempts to
  call `LiquidityController.mint()`
- The L2CGTBridge address can be obtained from the factory deployment event or precalculated using the factory
  deployment parameters

## Bridge Communication

Both bridges maintain a trusted relationship and can only accept finalization messages from their designated counterpart:

**L1CGTBridge Configuration:**

- `otherBridge`: Address of the corresponding L2CGTBridge (precalculated by L1CGTBridgeFactory during deployment)
- `messenger`: Address of the L1CrossDomainMessenger
- `cgtToken`: Address of the ERC20 token being bridged (immutable, set at factory deployment)

**L2CGTBridge Configuration:**

- `otherBridge`: Address of the corresponding L1CGTBridge (provided by L2CGTBridgeFactory during deployment)
- `messenger`: Address of the L2CrossDomainMessenger
- `liquidityController`: Address of the LiquidityController predeploy

The bridges enforce cross-domain message authentication by:

1. Verifying calls to `finalizeBridgeCGT()` come from the CrossDomainMessenger
2. Verifying the `xDomainMessageSender()` matches the expected counterpart bridge
   address
3. Only processing messages that originate from the trusted counterpart bridge

## Security Considerations

**Access Control:**

- L2CGTBridge MUST be authorized as a minter in the LiquidityController via `authorizeMinter()` before any L1→L2
  transfers can be completed
- This authorization must be performed by the LiquidityController owner (L1 ProxyAdmin owner) after bridge
  deployment
- Failure to authorize the L2CGTBridge will cause all `finalizeBridgeCGT()` calls to revert when attempting to
  mint native assets
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

- `setTrustedStateOnce()` MUST revert if called by any address other than contract owner and MUST only be callable
  once
- `trustedMessagePasserStorageRoot` MUST represent valid post-migration L2ToL1MessagePasser state
- Legacy finalization MUST check both OptimismPortal and L1CGTBridge to prevent replays
- Legacy finalization MUST reject `_tx.target == cgtToken` to prevent approval attacks
- Legacy withdrawals MUST only process transactions with `_tx.value > 0`
- Proof verification MUST use SecureMerkleTrie with same standards as OptimismPortal
