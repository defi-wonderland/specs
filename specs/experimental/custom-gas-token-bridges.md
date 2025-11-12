# Custom Gas Token Bridges Specification

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
  - [Pause Control](#pause-control)
- [L1CGTBridgeWithLegacyWithdrawal](#l1cgtbridgewithlegacywithdrawal)
  - [Additional Interface](#additional-interface)
  - [Legacy Withdrawal Support](#legacy-withdrawal-support)
  - [Additional Functions](#additional-functions)
    - [`setTrustedStateOnce`](#settrustedstateonce)
    - [`legacyProveWithdrawalTransaction`](#legacyprovewithdrawaltransaction)
    - [`legacyFinalizeWithdrawalTransaction`](#legacyfinalizewithdrawaltransaction)
    - [`enableDeposits` / `disableDeposits`](#enabledeposits--disabledeposits)
    - [`enableWithdrawals` / `disableWithdrawals`](#enablewithdrawals--disablewithdrawals)
  - [Trusted State Management](#trusted-state-management)
- [L2CGTBridge](#l2cgtbridge)
  - [Interface](#interface-1)
  - [Functions](#functions-1)
    - [`bridgeCGT`](#bridgecgt-1)
    - [`finalizeBridgeCGT`](#finalizebridgecgt-1)
  - [Events](#events-1)
    - [`CGTBridgeInitiated`](#cgtbridgeinitiated-1)
    - [`CGTBridgeFinalized`](#cgtbridgefinalized-1)
  - [Security Validations](#security-validations)
  - [Invariants](#invariants)
- [LiquidityController](#liquiditycontroller)
- [L1CGTBridgeFactory](#l1cgtbridgefactory)
  - [Interface](#interface-2)
  - [Deployment Process](#deployment-process)
    - [`deploy`](#deploy)
  - [Deterministic Addressing](#deterministic-addressing)
  - [Security Considerations](#security-considerations)
- [L2CGTBridgeFactory](#l2cgtbridgefactory)
  - [Interface](#interface-3)
  - [Deployment Process](#deployment-process-1)
  - [Post-Deployment Requirements](#post-deployment-requirements)
  - [Functions](#functions-2)
    - [`authorizeMinter`](#authorizeminter)
    - [`mint`](#mint)
    - [`burn`](#burn)
  - [Authorization Control](#authorization-control)
- [Bridge Communication](#bridge-communication)
- [Security Considerations](#security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

The Custom Gas Token (CGT) bridges enable bidirectional transfers between a specific L1 ERC20 token (designated as the chain's custom gas token) and L2 native assets on chains using Custom Gas Token mode. These bridges are purpose-built to handle the conversion between the chain's designated L1 ERC20 gas token and the L2's native gas-paying asset.

The CGT bridge system consists of:

- **L1CGTBridge**: Deployed on L1 by the L1CGTBridgeFactory, manages locking/unlocking of the designated ERC20 token
- **L2CGTBridge**: Deployed on L2 via L2CGTBridgeFactory triggered by L1CGTBridgeFactory, manages minting/burning of native assets through the LiquidityController
- **LiquidityController**: L2 predeploy contract that controls native asset liquidity
- **L1CGTBridgeWithLegacyWithdrawal**: Extension of L1CGTBridge with legacy withdrawal support
- **L1CGTBridgeFactory**: Factory contract on L1 that orchestrates deployment of both bridges using deterministic
  addressing
- **L2CGTBridgeFactory**: Factory contract deployed on L2 via cross-chain message to deploy the L2CGTBridge

Both bridges are deployed through factory contracts that ensure deterministic addressing and proper configuration,
communicating exclusively through their respective CrossDomainMessenger contracts while maintaining a trusted
relationship where only messages from the counterpart bridge are accepted.

## Token Flow

**L1 → L2 Flow (ERC20 to Native Asset):**

1. User calls `bridgeCGT()` on L1CGTBridge to transfer the specified amount of the designated ERC20
2. L1CGTBridge locks ERC20 tokens and sends a cross-domain message to L2CGTBridge via L1CrossDomainMessenger
3. L2CGTBridge receives the cross-domain message, calls `LiquidityController.mint()` to mint native assets
4. Native assets are sent to recipient address on L2

**L2 → L1 Flow (Native Asset to ERC20):**

1. User calls `bridgeCGT()` on L2CGTBridge with native assets (msg.value)
2. L2CGTBridge calls `LiquidityController.burn()` to deposit native assets into NativeAssetLiquidity
3. L2CGTBridge sends a cross-domain message to L1CGTBridge via L2CrossDomainMessenger
4. L1CGTBridge receives the cross-domain message and unlocks ERC20 tokens to recipient address on L1

## L1CGTBridge

The L1CGTBridge contract manages the L1 side of CGT bridging operations. It extends ProxyAdminOwnedBase for ownership control and ReinitializableBase for upgrade capabilities.

### Interface

```solidity
interface IL1CGTBridge {
    // Errors
    error Paused();
    error OnlyL2CGTBridge();

    // Events
    event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount);
    event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount);

    // Core bridge functions
    function bridgeCGT(address _to, uint256 _amount, uint32 _minGasLimit) external;
    function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external;

    // Initialization
    function initialize(
        ICrossDomainMessenger _messenger,
        ISuperchainConfig _superchainConfig,
        IERC20 _cgtToken,
        address _l2CGTBridge
    ) external;

    // Configuration getters
    function l2CGTBridge() external view returns (address);
    function cgtToken() external view returns (IERC20);
    function messenger() external view returns (ICrossDomainMessenger);
    function superchainConfig() external view returns (ISuperchainConfig);
    function version() external view returns (string memory);
}
```

### Functions

#### `bridgeCGT`

Initiates a CGT transfer from L1 to L2 to a specified recipient.

```solidity
function bridgeCGT(address _to, uint256 _amount, uint32 _minGasLimit) external virtual
```

- MUST check that the system is not paused via `superchainConfig.paused(address(this))`
- MUST transfer `_amount` of CGT ERC20 tokens from `msg.sender` to the bridge contract using `SafeERC20.safeTransferFrom`
- MUST send a message to L2CGTBridge via CrossDomainMessenger to mint equivalent native assets to `_to`
- MUST emit `CGTBridgeInitiated` event
- MUST revert if the system is paused
- MUST revert if caller has insufficient token balance or allowance

#### `finalizeBridgeCGT`

Finalizes a CGT transfer from L2 to L1 by unlocking tokens.

```solidity
function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external virtual
```

- MUST check that the system is not paused via `superchainConfig.paused(address(this))`
- MUST verify that the caller is the authorized CrossDomainMessenger
- MUST verify that `messenger.xDomainMessageSender()` is the L2CGTBridge address
- MUST transfer `_amount` of CGT ERC20 tokens from bridge contract to `_to` address using `SafeERC20.safeTransfer`
- MUST emit `CGTBridgeFinalized` event
- MUST revert if not called by the authorized messenger or L2CGTBridge
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

### Pause Control

The `L1CGTBridge` integrates with SuperchainConfig for pause control:

- Uses `superchainConfig.paused(address(this))` to check pause state
- Both `bridgeCGT` and `finalizeBridgeCGT` functions check pause state before proceeding
- Only the SuperchainConfig Guardian can pause/unpause the bridge
- Pauses have automatic expiration after 3 months (7,884,000 seconds)

## L1CGTBridgeWithLegacyWithdrawal

The `L1CGTBridgeWithLegacyWithdrawal` is an extension of the `L1CGTBridge` that includes additional functionality for handling legacy withdrawals from before CGT migration, bridge management controls, and trusted state management.

### Additional Interface

```solidity
interface IL1CGTBridgeWithLegacyWithdrawal is IL1CGTBridge {
    // Additional errors
    error TrustedStateAlreadySet();
    error TrustedStateNotSet();
    error DepositsDisabled();
    error WithdrawalsDisabled();
    error WithdrawalAlreadyFinalized();
    error WithdrawalNotProven();
    error InvalidWithdrawalValue();
    error InvalidTrustedRoot();

    // Additional events
    event WithdrawalProven(bytes32 indexed withdrawalHash, address indexed from, address indexed to);
    event WithdrawalProvenExtension1(bytes32 indexed withdrawalHash, address indexed proofSubmitter);
    event WithdrawalFinalized(bytes32 indexed withdrawalHash, bool success);
    event DepositsToggled(bool enabled);
    event WithdrawalsToggled(bool enabled);
    event TrustedStateSet(bytes32 indexed trustedRoot);

    // Legacy withdrawal functions
    function legacyProveWithdrawalTransaction(Types.WithdrawalTransaction memory _tx, bytes[] calldata _withdrawalProof) external;
    function legacyFinalizeWithdrawalTransaction(Types.WithdrawalTransaction memory _tx) external;

    // Trusted state management
    function setTrustedStateOnce(bytes32 trustedMessagePasserStorageRoot) external;

    // Bridge management
    function enableDeposits() external;
    function disableDeposits() external;
    function enableWithdrawals() external;
    function disableWithdrawals() external;

    // Additional getters
    function trustedMessagePasserStorageRoot() external view returns (bytes32);
    function depositsEnabled() external view returns (bool);
    function withdrawalsEnabled() external view returns (bool);
}
```

### Legacy Withdrawal Support

The `L1CGTBridgeWithLegacyWithdrawal` provides functionality to handle legacy withdrawals initiated before CGT migration. These functions use a trusted storage root to prove and finalize withdrawals containing native asset transfers (`_tx.value > 0`) that cannot be processed through the standard OptimismPortal after migration.

### Additional Functions

#### `setTrustedStateOnce`

Sets the trusted L2ToL1MessagePasser storage root for legacy withdrawal verification (migration only).

```solidity
function setTrustedStateOnce(bytes32 _trustedRoot) external
```

- MUST revert if not called by ProxyAdmin or ProxyAdmin owner
- MUST only be callable once (subsequent calls MUST revert)
- MUST store the provided storage root for legacy withdrawal proofs
- MUST be set to a valid L2ToL1MessagePasser storage root from after the CGT migration
- MUST emit `TrustedStateSet` event when set
- MUST revert if `_trustedRoot` is bytes32(0)

#### `legacyProveWithdrawalTransaction`

Proves a legacy withdrawal transaction using the trusted storage root.

```solidity
function legacyProveWithdrawalTransaction(
    Types.WithdrawalTransaction memory _tx,
    bytes[] calldata _withdrawalProof
) external
```

- MUST revert if `trustedMessagePasserStorageRoot` has not been set
- MUST only process withdrawals with `_tx.value > 0` (CGT conversions)
- MUST compute withdrawal hash using `Hashing.hashWithdrawal(_tx)`
- MUST verify inclusion proof using `SecureMerkleTrie.verifyInclusionProof()` against trusted root
- MUST store proven withdrawal in `provenLegacyWithdrawals[withdrawalHash][msg.sender]`
- MUST emit `WithdrawalProven` and `WithdrawalProvenExtension1` events
- MUST revert if proof is invalid

#### `legacyFinalizeWithdrawalTransaction`

Finalizes a proven legacy withdrawal transaction.

```solidity
function legacyFinalizeWithdrawalTransaction(Types.WithdrawalTransaction memory _tx) external
```

- MUST revert if withdrawal has not been previously proven
- MUST verify withdrawal has not been finalized in OptimismPortal or L1CGTBridge
- MUST only finalize withdrawals with `_tx.value > 0` (native asset withdrawals)
- MUST transfer ERC20 tokens equivalent to `_tx.value` to `_tx.target`
- MUST execute `_tx.data` if present (with gas limit checks)
- MUST mark withdrawal as finalized to prevent replay
- MUST emit `WithdrawalFinalized` event

#### `enableDeposits` / `disableDeposits`

Enables or disables deposit functionality for emergency control.

```solidity
function enableDeposits() external
function disableDeposits() external
```

- MUST revert if not called by ProxyAdmin or ProxyAdmin owner
- MUST pause/unpause deposit functions (`bridgeCGT`)
- MUST emit `DepositsToggled` event when state changes

#### `enableWithdrawals` / `disableWithdrawals`

Enables or disables withdrawal functionality for emergency control.

```solidity
function enableWithdrawals() external
function disableWithdrawals() external
```

- MUST revert if not called by ProxyAdmin or ProxyAdmin owner
- MUST pause/unpause withdrawal functions (`finalizeBridgeCGT`, legacy functions)
- MUST emit `WithdrawalsToggled` event when state changes

### Trusted State Management

The trusted state for legacy withdrawals is managed through:

- `_trustedMessagePasserStorageRoot`: Storage root set once for legacy verification
- `_trustedStateSet`: Boolean flag preventing re-setting
- `provenLegacyWithdrawals`: Mapping of withdrawal hashes to proof submitters to finalization status
- `finalizedLegacyWithdrawals`: Mapping of withdrawal hashes to finalization status

## L2CGTBridge

The `L2CGTBridge` contract handles the L2 side of Custom Gas Token bridging operations, enabling bidirectional transfers between L1 ERC20 tokens and L2 native assets. This contract is deployed on chains using Custom Gas Token mode and has minting authorization from the LiquidityController.

### Interface

```solidity
interface IL2CGTBridge {
    // Errors
    error OnlyL1CGTBridge();
    error InvalidRecipient();

    // Events
    event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount);
    event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount);

    // Core bridge functions
    function bridgeCGT(address _to, uint32 _minGasLimit) external payable;
    function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external;

    // Initialization
    function initialize(
        ICrossDomainMessenger _messenger,
        ILiquidityController _liquidityController,
        address _l1CGTBridge
    ) external;

    // Configuration getters
    function l1CGTBridge() external view returns (address);
    function messenger() external view returns (ICrossDomainMessenger);
    function liquidityController() external view returns (ILiquidityController);
    function version() external view returns (string memory);
}
```

### Functions

#### `bridgeCGT`

Initiates a CGT transfer from L2 to L1 to a specified recipient address.

```solidity
function bridgeCGT(address _to, uint32 _minGasLimit) external payable virtual
```

- MUST accept native assets via `msg.value` (the amount being bridged)
- MUST call `LiquidityController.burn{value: msg.value}()` to deposit native assets into NativeAssetLiquidity contract
- MUST send cross-domain message to L1CGTBridge via L2CrossDomainMessenger to unlock equivalent ERC20 tokens to `_to`
- MUST pass `msg.value` as the `_amount` parameter in the cross-domain message for L1 finalization
- MUST emit `CGTBridgeInitiated` event with `msg.sender`, `_to`, and `msg.value`
- MUST revert if `_to` is zero address
- MUST revert if `msg.value` is zero

#### `finalizeBridgeCGT`

Finalizes a CGT transfer from L1 to L2 by minting native assets to the recipient.

```solidity
function finalizeBridgeCGT(address _from, address _to, uint256 _amount) external virtual
```

- MUST verify that the caller is the authorized L2CrossDomainMessenger
- MUST verify that `CrossDomainMessenger.xDomainMessageSender()` is the authorized L1CGTBridge address
- MUST validate that `_to` is not an invalid address (bridge or messenger)
- MUST call `LiquidityController.mint(_to, _amount)` to mint native assets to recipient
- MUST emit `CGTBridgeFinalized` event
- MUST revert if not called by the authorized messenger or L1CGTBridge
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

### Security Validations

The L2CGTBridge implements multiple security validations:

**Cross-Domain Message Authentication:**

```solidity
if (msg.sender != address(messenger) || messenger.xDomainMessageSender() != l1CGTBridge) {
    revert OnlyL1CGTBridge();
}
```

**Recipient Validation:**

```solidity
if (_to == address(this) || _to == address(messenger)) {
    revert InvalidRecipient();
}
```

### Invariants

- Only the L2CrossDomainMessenger can call `finalizeBridgeCGT()` when relaying messages from the authorized L1CGTBridge
- The L2CGTBridge MUST be authorized as a minter in the LiquidityController before any L1→L2 bridging operations can succeed
- Authorization is granted by the LiquidityController owner via `LiquidityController.authorizeMinter(l2BridgeAddress)`
- All L2→L1 transfers immediately burn native assets by depositing them into NativeAssetLiquidity
- All L1→L2 transfers mint native assets by withdrawing them from NativeAssetLiquidity via LiquidityController
- Native asset supply on L2 is backed 1:1 by locked ERC20 tokens on L1

## LiquidityController

The LiquidityController is a predeploy contract on L2 that controls the liquidity of native assets on the L2 chain. It handles the minting and burning of native assets for CGT bridging operations.

### Interface

```solidity
interface ILiquidityController {
    // Errors
    error Unauthorized();

    // Events
    event MinterAuthorized(address indexed minter);
    event LiquidityMinted(address indexed minter, address indexed to, uint256 amount);
    event LiquidityBurned(address indexed minter, uint256 amount);

    // Authorization functions
    function authorizeMinter(address _minter) external;

    // Liquidity functions
    function mint(address _to, uint256 _amount) external;
    function burn() external payable;

    // Getters
    function minters(address) external view returns (bool);
    function gasPayingTokenName() external view returns (string memory);
    function gasPayingTokenSymbol() external view returns (string memory);

    // Initialization
    function initialize(string memory _gasPayingTokenName, string memory _gasPayingTokenSymbol) external;
}
```

### Functions

#### `authorizeMinter`

Authorizes an address to perform liquidity control operations.

```solidity
function authorizeMinter(address _minter) external
```

- MUST revert if not called by the ProxyAdmin owner (L1 system owner)
- MUST set `minters[_minter] = true`
- MUST emit `MinterAuthorized` event

#### `mint`

Mints native asset liquidity and sends it to a specified address.

```solidity
function mint(address _to, uint256 _amount) external
```

- MUST revert if `msg.sender` is not authorized as a minter
- MUST call `INativeAssetLiquidity(Predeploys.NATIVE_ASSET_LIQUIDITY).withdraw(_amount)`
- MUST send native assets to `_to` using forced SafeSend
- MUST emit `LiquidityMinted` event

#### `burn`

Burns native asset liquidity by sending ETH to the contract.

```solidity
function burn() external payable
```

- MUST revert if `msg.sender` is not authorized as a minter
- MUST call `INativeAssetLiquidity(Predeploys.NATIVE_ASSET_LIQUIDITY).deposit{value: msg.value}()`
- MUST emit `LiquidityBurned` event

### Authorization Control

Authorization control in LiquidityController is critical for security:

- Only the ProxyAdmin owner (which is the L1 system owner) can authorize minters
- Authorized minters can call `mint()` and `burn()`
- The L2CGTBridge MUST be authorized as a minter before bridging operations work
- Authorization is checked on every mint/burn operation

## L1CGTBridgeFactory

The `L1CGTBridgeFactory` is the main contract responsible for orchestrating the complete deployment of the CGT bridge system. It is deployed on L1 and coordinates the deterministic creation of both bridges (L1 and L2) ensuring predictable addresses and proper configuration.

### Interface

```solidity
interface IL1CGTBridgeFactory {
    // Errors
    error InvalidSalt();
    error InvalidERC20Token();
    error DeploymentFailed();
    error BridgeAlreadyDeployed();
    error InvalidL2ChainId();

    // Events
    event L1BridgeDeployed(address indexed l1Bridge, address indexed cgtToken, uint256 indexed l2ChainId);
    event L2DeploymentInitiated(address indexed l1Bridge, address indexed l2Bridge, uint256 indexed l2ChainId);
    event BridgeDeploymentCompleted(address indexed l1Bridge, address indexed l2Bridge, uint256 indexed l2ChainId);

    // Core deployment function
    function deploy(
        address _cgtToken,
        uint256 _l2ChainId,
        bytes32 _salt,
        bool _withLegacySupport,
        uint32 _minGasLimit
    ) external returns (address l1Bridge, address l2Bridge);

    // Address prediction
    function computeL1BridgeAddress(
        address _cgtToken,
        uint256 _l2ChainId,
        bytes32 _salt,
        bool _withLegacySupport
    ) external view returns (address);

    function computeL2BridgeAddress(
        IERC20 _cgtToken,
        uint256 _l2ChainId,
        bytes32 _salt
    ) external view returns (address);

    // Configuration getters
    function l1CrossDomainMessenger() external view returns (ICrossDomainMessenger);
    function superchainConfig() external view returns (ISuperchainConfig);
    function deployedBridges(IERC20 _cgtToken, uint256 _l2ChainId) external view returns (address l1Bridge, address l2Bridge);
}
```

### Deployment Process

The deployment process follows a sequential flow that ensures proper configuration of both bridges:

1. **Parameter Validation**: Verifies that the ERC20 token, chainId, and salt are valid
2. **L1 Bridge Creation**: Uses CREATE2 for deterministic deployment of the L1CGTBridge
3. **L1 Bridge Initialization**: Configures messenger, superchainConfig, cgtToken, and l2CGTBridge address
4. **Cross-Chain Message**: Sends message to L2CGTBridgeFactory to create the L2 bridge
5. **Final Configuration**: Updates registry of deployed bridges

#### `deploy`

Main function that orchestrates the complete deployment of the bridge system.

```solidity
function deploy(
    IERC20 _cgtToken,
    uint256 _l2ChainId,
    bytes32 _salt,
    bool _withLegacySupport,
    uint32 _minGasLimit
) external returns (address l1Bridge, address l2Bridge)
```

**Execution Flow:**

1. **Initial Validations:**

   - MUST verify that `_cgtToken` is a valid ERC20 contract
   - MUST verify that `_l2ChainId` corresponds to a valid Superchain chain
   - MUST verify that no bridge already exists for this token/chain combination
   - MUST verify that `_salt` does not generate address collisions

2. **Address Calculation:**

   - MUST calculate the L1 bridge address using `computeL1BridgeAddress()`
   - MUST calculate the L2 bridge address using `computeL2BridgeAddress()`
   - MUST ensure that both addresses are unique and valid

3. **L1 Bridge Deployment:**

   - MUST use CREATE2 with deterministic salt to deploy L1CGTBridge or L1CGTBridgeWithLegacyWithdrawal
   - MUST initialize the contract with correct parameters: messenger, superchainConfig, cgtToken, l2Bridge address
   - MUST verify that initialization was successful

4. **L2 Bridge Deployment (Cross-Chain):**

   - MUST send cross-chain message to L2CGTBridgeFactory on the target chain
   - MUST include all necessary parameters for deployment: l1Bridge address, salt, cgtToken info
   - MUST specify appropriate gas limit for the deployment operation

5. **Registry Update:**
   - MUST update `deployedBridges[_cgtToken][_l2ChainId]` with the addresses
   - MUST emit deployment and completion events

**Parameters:**

- `_cgtToken`: Address of the ERC20 token that will be the chain's gas token
- `_l2ChainId`: Chain ID of the L2 where the L2 bridge will be deployed
- `_salt`: Salt for deterministic deployment (must be unique per token/chain)
- `_withLegacySupport`: Whether to use L1CGTBridgeWithLegacyWithdrawal instead of basic L1CGTBridge
- `_minGasLimit`: Minimum gas limit for the cross-chain deployment message

**Returns:**

- `l1Bridge`: Address of the deployed L1CGTBridge
- `l2Bridge`: Predicted address of the L2CGTBridge (will be deployed via cross-chain message)

### Deterministic Addressing

The factory uses CREATE2 to guarantee deterministic and predictable addresses for both bridges, allowing off-chain calculation of addresses before deployment.

**L1 Bridge Address Calculation:**

The L1 factory MUST calculate bridge addresses using a deterministic salt that combines the token address, L2 chain ID, and provided salt. The calculation MUST differentiate between legacy and non-legacy bridge types.

**L2 Bridge Address Calculation:**

The L2 factory MUST calculate bridge addresses using the same deterministic scheme to ensure consistency between L1 predictions and L2 deployments.

**Salt Generation Strategy:**

The combined salt MUST be generated by encoding the token address, chain ID, and provided salt together. This ensures that:

- Each token/chain combination has unique addresses
- The provided salt allows variations for the same token/chain
- Addresses are calculable off-chain
- There is no risk of collisions between different deployments

### Security Considerations

**Deployment Security:**

- MUST verify that only authorized addresses can deploy bridges
- MUST validate that the ERC20 token is legitimate before deployment
- MUST prevent duplicate deployments for the same token/chain combination
- MUST ensure that gas limit for cross-chain messages is sufficient

**Salt Management:**

- MUST use salts that cannot be predicted by attackers
- MUST verify that calculated addresses do not collide with existing contracts

**Cross-Chain Deployment:**

- MUST validate that the L2 chain ID corresponds to a valid Superchain chain
- MUST handle failures in L2 bridge deployment
- MUST provide retry mechanisms for failed cross-chain messages

**Access Control:**

- SHOULD implement access controls for deployments
- SHOULD allow pausing of the factory in emergencies
- SHOULD have upgrade functionality for security fixes

## L2CGTBridgeFactory

The `L2CGTBridgeFactory` is a predeploy contract on L2 that receives cross-chain messages from the L1CGTBridgeFactory to deploy L2CGTBridge instances. This contract ensures that only legitimate bridges are deployed and handles the required initial configuration.

### Interface

```solidity
interface IL2CGTBridgeFactory {
    // Errors
    error OnlyL1Factory();
    error BridgeAlreadyDeployed();
    error InvalidParameters();
    error DeploymentFailed();
    error AuthorizationFailed();

    // Events
    event L2BridgeDeployed(address indexed l2Bridge, address indexed l1Bridge, address indexed cgtTokenL1);
    event MinterAuthorized(address indexed l2Bridge, address indexed liquidityController);
    event BridgeConfigured(address indexed l2Bridge, address indexed messenger, address indexed liquidityController);

    // Core deployment function (called via cross-chain message)
    function deployL2Bridge(
        address _l1Bridge,
        address _cgtTokenL1,
        string memory _tokenName,
        string memory _tokenSymbol,
        bytes32 _salt
    ) external returns (address l2Bridge);

    // Address prediction
    function computeL2BridgeAddress(
        address _l1Bridge,
        address _cgtTokenL1,
        bytes32 _salt
    ) external view returns (address);

    // Configuration getters
    function l1CGTBridgeFactory() external view returns (address);
    function l2CrossDomainMessenger() external view returns (ICrossDomainMessenger);
    function liquidityController() external view returns (ILiquidityController);
    function deployedBridges(address _l1Bridge) external view returns (address l2Bridge);
}
```

### Deployment Process

The L2CGTBridgeFactory receives cross-chain messages from the L1CGTBridgeFactory and executes the deployment of the L2 bridge with automatic configuration.

**Deployment Flow:**

1. **Cross-Chain Message Reception:**

   - MUST verify that the message comes from the authorized L1CrossDomainMessenger
   - MUST verify that the `xDomainMessageSender()` is the authorized L1CGTBridgeFactory
   - MUST validate all deployment parameters

2. **Parameter Validation:**

   - MUST verify that `_l1Bridge` is a valid contract address
   - MUST verify that `_cgtTokenL1` is a valid ERC20 token address
   - MUST verify that no bridge already exists for this L1 bridge
   - MUST validate that the salt does not generate collisions

3. **L2 Bridge Deployment:**

   - MUST use CREATE2 for deterministic deployment
   - MUST initialize the L2CGTBridge with: messenger, liquidityController, l1Bridge
   - MUST verify that initialization was successful

4. **Automatic Configuration:**

   - MUST authorize the L2 bridge as minter in the LiquidityController
   - MUST configure the LiquidityController with the token name and symbol
   - MUST update the registry of deployed bridges

### Post-Deployment Requirements

After successful deployment of the L2 bridge, additional configurations are required for complete operation:

**Automatic Configuration (Handled by Factory):**

1. **Minter Authorization:**

   - The factory MUST call `liquidityController.authorizeMinter(l2Bridge)` automatically
   - This authorization allows the L2 bridge to mint/burn native assets
   - Without this authorization, all L1→L2 bridging operations will fail

2. **LiquidityController Initialization:**
   - If it's the first bridge deployed on the chain, initialize with token name/symbol
   - Configure gas token parameters for the chain
   - Establish initial security configurations

**Post-Deployment Verifications:**

1. **Authorization Verification:**

   - MUST verify that the L2 bridge is authorized as a minter in the LiquidityController

2. **Configuration Verification:**

   - MUST verify that the L2 bridge is correctly configured with the L1 bridge address
   - MUST verify that the L2 bridge is correctly configured with the LiquidityController address

3. **Address Verification:**
   - MUST verify that the deployed bridge address matches the predicted deterministic address

### Deterministic Address Calculation

The factory MUST use the same deterministic scheme as the L1 factory to ensure consistency. The address calculation MUST combine the L1 bridge address, L1 token address, and salt to generate a unique deterministic address.

### Security Considerations

**Cross-Chain Message Authentication:**

- MUST verify that only the L1CGTBridgeFactory can initiate deployments
- MUST validate that the cross-chain message is authentic via CrossDomainMessenger
- MUST reject messages from unauthorized addresses

**Deployment Security:**

- MUST prevent duplicate deployments for the same L1 bridge
- MUST validate all parameters before deployment
- MUST ensure that minter authorization is performed automatically

**Access Control:**

- SHOULD implement emergency pause functionality
- SHOULD allow updates of the L1 factory address if necessary
- SHOULD have authorization revocation mechanisms in emergencies

**Upgrade Safety:**

- MUST ensure that deployed bridges are not affected by factory upgrades
- SHOULD maintain backward compatibility with existing bridges
- SHOULD implement versioning for different bridge implementations

## Bridge Communication

Both bridges maintain a trusted relationship and can only accept finalization messages from their designated counterpart:

**L1CGTBridge Configuration:**

- `l2CGTBridge`: Address of the corresponding L2CGTBridge
- `messenger`: Address of the L1CrossDomainMessenger
- `cgtToken`: Address of the ERC20 token being bridged (immutable)
- `superchainConfig`: Address of the SuperchainConfig for pause control

**L2CGTBridge Configuration:**

- `l1CGTBridge`: Address of the corresponding L1CGTBridge
- `messenger`: Address of the L2CrossDomainMessenger
- `liquidityController`: Address of the LiquidityController predeploy

The bridges enforce cross-domain message authentication by:

1. Verifying calls to `finalizeBridgeCGT()` come from the CrossDomainMessenger
2. Verifying the `xDomainMessageSender()` matches the expected counterpart bridge address
3. Only processing messages that originate from the trusted counterpart bridge

## Security Considerations

**Access Control:**

- L2CGTBridge MUST be authorized as a minter in the LiquidityController via `authorizeMinter()` before any L1→L2 transfers can be completed
- This authorization must be performed by the LiquidityController owner (L1 ProxyAdmin owner) after bridge deployment
- Failure to authorize the L2CGTBridge will cause all `finalizeBridgeCGT()` calls to revert when attempting to mint native assets
- Only the designated L1CGTBridge can send finalization messages to L2CGTBridge
- Only the L2CGTBridge can send finalization messages to L1CGTBridge

**Token Safety:**

- L1CGTBridge holds locked ERC20 tokens as collateral for minted L2 native assets
- Native assets on L2 are backed 1:1 by locked ERC20 tokens on L1
- Burns on L2 immediately deposit assets into NativeAssetLiquidity, reducing circulating supply

**Pause Control:**

- L1CGTBridge integrates with SuperchainConfig for granular contract-specific pause control
- Only the SuperchainConfig Guardian can pause/unpause the bridge
- Pauses automatically expire after 3 months to prevent indefinite pausing
- L1CGTBridgeWithLegacyWithdrawal includes additional pause controls for deposits and withdrawals

**Bridge Integrity:**

- Failed cross-domain messages can be replayed through standard message relay mechanisms
- Both bridges MUST validate all message parameters to prevent invalid minting or unlocking operations
- Bridge contracts SHOULD implement pausability for emergency situations

**Legacy Withdrawal Security (L1CGTBridgeWithLegacyWithdrawal only):**

- `setTrustedStateOnce()` MUST revert if called by any address other than contract owner and MUST only be callable once
- `trustedMessagePasserStorageRoot` MUST represent valid post-migration L2ToL1MessagePasser state
- Legacy finalization MUST check both OptimismPortal and L1CGTBridge to prevent replays
- Legacy finalization MUST reject `_tx.target == cgtToken` to prevent approval attacks
- Legacy withdrawals MUST only process transactions with `_tx.value > 0`
- Proof verification MUST use SecureMerkleTrie with same standards as OptimismPortal
