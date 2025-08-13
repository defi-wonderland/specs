# Predeploys

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

**Table of Contents**

- [Overview](#overview)
- [WETH9](#weth9)
- [L2CrossDomainMessenger](#l2crossdomainmessenger)
- [L2StandardBridge](#l2standardbridge)
- [SequencerFeeVault](#sequencerfeevault)
- [BaseFeeVault](#basefeevault)
- [L1FeeVault](#l1feevault)
- [Native Asset Liquidity](#native-asset-liquidity)
  - [Functions](#functions)
    - [`deposit`](#deposit)
    - [`withdraw`](#withdraw)
    - [`fund`](#fund)
  - [Events](#events)
    - [`LiquidityDeposited`](#liquiditydeposited)
    - [`LiquidityWithdrawn`](#liquiditywithdrawn)
    - [`LiquidityFunded`](#liquidityfunded)
  - [Invariants](#invariants)
- [Liquidity Controller](#liquidity-controller)
  - [Functions](#functions-1)
    - [`authorizeMinter`](#authorizeminter)
    - [`mint`](#mint)
    - [`burn`](#burn)
    - [`gasPayingAssetName`](#gaspayingassetname)
    - [`gasPayingAssetSymbol`](#gaspayingassetsymbol)
  - [Events](#events-1)
    - [`MinterAuthorized`](#minterauthorized)
    - [`AssetsMinted`](#assetsminted)
    - [`AssetsBurned`](#assetsburned)
  - [Invariants](#invariants-1)
- [L2CGTBridge](#l2cgtbridge)
  - [Functions](#functions-2)
    - [`bridgeCGT`](#bridgecgt)
    - [`bridgeCGTTo`](#bridgecgtto)
    - [`finalizeBridgeCGT`](#finalizebridgecgt)
  - [Events](#events-2)
    - [`CGTBridgeInitiated`](#cgtbridgeinitiated)
    - [`CGTBridgeFinalized`](#cgtbridgefinalized)
  - [Invariants](#invariants-2)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

| Name                 | Address                                    | Introduced | Deprecated | Proxied |
| -------------------- | ------------------------------------------ | ---------- | ---------- | ------- |
| NativeAssetLiquidity | 0x4200000000000000000000000000000000000029 | Jovian     | No         | Yes     |
| LiquidityController  | 0x420000000000000000000000000000000000002A | Jovian     | No         | Yes     |
| L2CGTBridge          | 0x420000000000000000000000000000000000002B | Jovian     | No         | Yes     |

## WETH9

On chains using Custom Gas Token mode, this contract serves as the Wrapped Native
Asset (WNA) instead of Wrapped Ether. The WETH predeploy implementation remains
unchanged and continues to fetch metadata from the `L1Block` predeploy. However,
the `L1Block` predeploy is updated to source the name and symbol from the
`LiquidityController` instead of using hardcoded ETH values, with the name and
symbol being prefixed with "Wrapped" and "W" respectively.

**Important**: Currently, a chain using ETH cannot migrate to use a custom gas token,
as this migration would introduce significant risks and breaking changes to existing
infrastructure. Custom Gas Token mode is only supported for fresh deployments.
Migration only focuses on chains utilizing the old CGT implementation.

For fresh Custom Gas Token deployments, the `L1Block` is deployed from genesis
with the metadata functions already configured to fetch from the `LiquidityController`.

## L2CrossDomainMessenger

The `sendMessage` function MUST revert if `L1Block.isCustomGasToken()` returns `true` and `msg.value > 0`.
This revert occurs because `L2CrossDomainMessenger` internally calls `L2ToL1MessagePasser.initiateWithdrawal`
which enforces the CGT restriction.

## L2StandardBridge

ETH bridging functions MUST revert if `L1Block.isCustomGasToken()` returns `true` and the function involves ETH transfers.
This revert occurs because `L2StandardBridge` internally calls `L2CrossDomainMessenger.sendMessage`,
which in turn calls `L2ToL1MessagePasser.initiateWithdrawal` that enforces the CGT restriction.

## SequencerFeeVault

The contract constructor takes a `WithdrawalNetwork` parameter that determines whether
funds are sent to L1 or L2:

- `WithdrawalNetwork.L1`: Funds are withdrawn to an L1 address (default behavior)
- `WithdrawalNetwork.L2`: Funds are withdrawn to an L2 address

For existing deployments, to change the withdrawal network or recipient address,
a new implementation must be deployed and the proxy must be upgraded.

For fresh deployments with Custom Gas Token mode enabled, the withdrawal network
MUST be set to `WithdrawalNetwork.L2`.

## BaseFeeVault

The contract constructor takes a `WithdrawalNetwork` parameter that determines whether
funds are sent to L1 or L2:

- `WithdrawalNetwork.L1`: Funds are withdrawn to an L1 address (default behavior)
- `WithdrawalNetwork.L2`: Funds are withdrawn to an L2 address

For existing deployments, to change the withdrawal network or recipient address,
a new implementation must be deployed and the proxy must be upgraded.

For fresh deployments with Custom Gas Token mode enabled, the withdrawal network
MUST be set to `WithdrawalNetwork.L2`.

## L1FeeVault

The contract constructor takes a `WithdrawalNetwork` parameter that determines whether
funds are sent to L1 or L2:

- `WithdrawalNetwork.L1`: Funds are withdrawn to an L1 address (default behavior)
- `WithdrawalNetwork.L2`: Funds are withdrawn to an L2 address

For existing deployments, to change the withdrawal network or recipient address,
a new implementation must be deployed and the proxy must be upgraded.

For fresh deployments with Custom Gas Token mode enabled, the withdrawal network
MUST be set to `WithdrawalNetwork.L2`.

## Native Asset Liquidity

Address: `0x4200000000000000000000000000000000000029`

The `NativeAssetLiquidity` predeploy stores a large amount of pre-minted native asset
that serves as the central liquidity source for Custom Gas Token chains. This contract
is only deployed on chains using Custom Gas Token mode and acts as the vault for all
native asset supply management operations.

### Functions

#### `deposit`

Accepts native asset deposits and locks them in the contract, reducing circulating supply.

```solidity
function deposit() external payable
```

- MUST only be callable by the `LiquidityController` predeploy
- MUST accept any amount of native asset via `msg.value`
- MUST emit `LiquidityDeposited` event

#### `withdraw`

Sends native assets from the contract to the `LiquidityController`, increasing circulating supply.

```solidity
function withdraw(uint256 _amount) external
```

- MUST only be callable by the `LiquidityController` predeploy
- MUST send exactly `_amount` of native asset to the caller
- MUST revert if the contract balance is insufficient
- MUST emit `LiquidityWithdrawn` event

#### `fund`

Allows funding the contract with native assets. This function is used to initialize the contract with a large liquidity pool, similar to how ETHLiquidity is initialized in interop chains.

```solidity
function fund() external payable
```

- MUST accept any amount of native asset via `msg.value`
- MUST revert if `msg.value` is zero
- MUST emit `LiquidityFunded` event
- MUST be callable by any address

#### `burn`

Burns native supply from the contract, permanently removing it from circulation.
This function utilizes the Burn.sol library for burn functionality.

```solidity
function burn(uint256 _amount) external
```

- MUST burn exactly `_amount` of native asset from the contract balance
- MUST permanently reduce the total native asset supply
- MUST revert if the contract balance is insufficient
- MUST be callable by authorized addresses only

### Events

#### `LiquidityDeposited`

Emitted when native assets are deposited into the contract.

```solidity
event LiquidityDeposited(address indexed caller, uint256 value)
```

#### `LiquidityWithdrawn`

Emitted when native assets are withdrawn from the contract.

```solidity
event LiquidityWithdrawn(address indexed caller, uint256 value)
```

#### `LiquidityFunded`

Emitted when the contract receives funding, typically during initial deployment.

```solidity
event LiquidityFunded(address indexed funder, uint256 amount)
```

#### `LiquidityBurned`

Emitted when native assets are permanently burned from the contract.

```solidity
event LiquidityBurned(address indexed burner, uint256 amount)
```

### Invariants

- Only the `LiquidityController` predeploy can call `deposit()` and `withdraw()`
- All native asset supply changes must go through this contract when CGT mode is active
- No direct user interaction is permitted with liquidity management functions
- The `burn()` function permanently reduces total supply and cannot be reversed
- Burned assets are permanently removed from circulation

## Liquidity Controller

Address: `0x420000000000000000000000000000000000002A`

The `LiquidityController` predeploy manages access to the `NativeAssetLiquidity` contract
and provides the governance interface for Custom Gas Token chains. This contract is only
deployed on chains using Custom Gas Token mode and serves as the central authority for
native asset supply management and metadata provisioning.

### Functions

#### `authorizeMinter`

Authorizes an address to mint native assets from the liquidity pool.

```solidity
function authorizeMinter(address _minter) external
```

- MUST only be callable by the L1 ProxyAdmin owner
- MUST authorize `_minter` to call the `mint()` function
- MUST emit `MinterAuthorized` event

#### `mint`

Unlocks native assets from the `NativeAssetLiquidity` contract and sends them to a specified address.

```solidity
function mint(address _to, uint256 _amount) external
```

- MUST only be callable by authorized minters
- MUST call `NativeAssetLiquidity.withdraw(_amount)` to unlock assets
- MUST send exactly `_amount` of native asset to `_to` address
- MUST revert if `NativeAssetLiquidity` has insufficient balance
- MUST emit `AssetsMinted` event

#### `burn`

Deposits native assets back into the `NativeAssetLiquidity` contract, reducing circulating supply.

```solidity
function burn() external payable
```

- MUST accept any amount of native asset via `msg.value`
- MUST call `NativeAssetLiquidity.deposit{value: msg.value}()` to lock assets
- MUST only be callable by authorized minters
- MUST revert if `msg.value` is zero
- MUST emit `AssetsBurned` event

#### `gasPayingAssetName`

Returns the human-readable name of the gas-paying asset for metadata purposes.

```solidity
function gasPayingAssetName() external view returns (string memory)
```

- MUST return the name of the native asset (e.g., "MyToken")
- MUST be used by L1Block predeploy for `name()` function
- Returns value used to construct "Wrapped {AssetName}" for WNA

#### `gasPayingAssetSymbol`

Returns the symbol of the gas-paying asset for metadata purposes.

```solidity
function gasPayingAssetSymbol() external view returns (string memory)
```

- MUST return the symbol of the native asset (e.g., "MTK")
- MUST be used by L1Block predeploy for `symbol()` function
- Returns value used to construct "W{AssetSymbol}" for WNA

### Events

#### `MinterAuthorized`

Emitted when a new minter is authorized by the contract owner.

```solidity
event MinterAuthorized(address indexed minter, address indexed authorizer)
```

Where `minter` is the address being authorized and `authorizer` is the L1 ProxyAdmin owner who authorized them.

#### `AssetsMinted`

Emitted when native assets are unlocked from the liquidity pool and sent to a recipient.

```solidity
event AssetsMinted(address indexed minter, address indexed to, uint256 amount)
```

Where `minter` is the authorized address calling the function, `to` is the recipient address,
and `amount` is the amount of native assets minted.

#### `AssetsBurned`

Emitted when native assets are locked back into the liquidity pool.

```solidity
event AssetsBurned(address indexed burner, uint256 amount)
```

Where `burner` is the `msg.sender` who burned the assets and `amount` is the amount of native assets burned.

### Invariants

- Only authorized minters can call `mint()` to unlock native assets
- Only the L1 ProxyAdmin owner can authorize new minters via `authorizeMinter()`
- All native asset supply changes must flow through this contract's governance
- The contract acts as the sole interface between governance and `NativeAssetLiquidity`
- `burn()` operations always increase locked supply by calling `NativeAssetLiquidity.deposit()`
- `mint()` operations always decrease locked supply by calling `NativeAssetLiquidity.withdraw()`

## L2CGTBridge

Address: `0x420000000000000000000000000000000000002B`

The `L2CGTBridge` predeploy handles the L2 side of Custom Gas Token bridging operations, enabling bidirectional transfers between L1 ERC20 tokens and L2 native assets. This contract is only deployed on chains using Custom Gas Token mode and has minting authorization from the `LiquidityController` to manage native asset supply.

The bridge maintains a trusted relationship with its L1 counterpart (`L1CGTBridge`) and only accepts cross-domain messages from the designated L1 bridge address through the `L2CrossDomainMessenger`.

### Functions

#### `bridgeCGT`

Initiates a CGT transfer from L2 to L1 for the caller.

```solidity
function bridgeCGT(uint32 _minGasLimit, bytes calldata _extraData) external payable
```

- MUST accept native assets via `msg.value`
- MUST call `LiquidityController.burn{value: msg.value}()` to deposit native assets into NativeAssetLiquidity contract
- MUST send cross-domain message to L1CGTBridge via `L2CrossDomainMessenger` to unlock equivalent ERC20 tokens to `msg.sender`
- MUST emit `CGTBridgeInitiated` event
- MUST revert if `msg.value` is zero

#### `bridgeCGTTo`

Initiates a CGT transfer from L2 to L1 to a specified recipient address.

```solidity
function bridgeCGTTo(address _to, uint32 _minGasLimit, bytes calldata _extraData) external payable
```

- MUST accept native assets via `msg.value`
- MUST call `LiquidityController.burn{value: msg.value}()` to deposit native assets into NativeAssetLiquidity contract
- MUST send cross-domain message to L1CGTBridge via `L2CrossDomainMessenger` to unlock equivalent ERC20 tokens to `_to`
- MUST emit `CGTBridgeInitiated` event
- MUST revert if `_to` is zero address
- MUST revert if `msg.value` is zero

#### `finalizeBridgeCGT`

Finalizes a CGT transfer from L1 to L2 by minting native assets to the recipient.

```solidity
function finalizeBridgeCGT(address _from, address _to, uint256 _amount, bytes calldata _extraData) external
```

- MUST only be callable by the `L2CrossDomainMessenger` when relaying a message from the authorized L1CGTBridge
- MUST call `LiquidityController.mint(_to, _amount)` to mint native assets to recipient
- MUST emit `CGTBridgeFinalized` event
- MUST revert if called by any address other than the `L2CrossDomainMessenger`
- MUST revert if the `CrossDomainMessenger.xDomainMessageSender()` is not the authorized L1CGTBridge address
- MUST revert if `LiquidityController.mint()` operation fails

### Events

#### `CGTBridgeInitiated`

Emitted when a CGT bridge transfer is initiated from L2 to L1.

```solidity
event CGTBridgeInitiated(address indexed from, address indexed to, uint256 amount, bytes extraData)
```

Where `from` is the L2 sender, `to` is the L1 recipient, `amount` is the native asset amount being bridged, and `extraData` contains additional bridging parameters.

#### `CGTBridgeFinalized`

Emitted when a CGT bridge transfer from L1 to L2 is finalized on L2.

```solidity
event CGTBridgeFinalized(address indexed from, address indexed to, uint256 amount, bytes extraData)
```

Where `from` is the L1 sender, `to` is the L2 recipient, `amount` is the native asset amount minted, and `extraData` contains additional bridging parameters.

### Invariants

- Only the `L2CrossDomainMessenger` can call `finalizeBridgeCGT()` when relaying messages from the authorized L1CGTBridge
- The L2CGTBridge MUST be authorized as a minter in the `LiquidityController` before any bridging operations
- All L2→L1 transfers immediately burn native assets by depositing them into `NativeAssetLiquidity`
- All L1→L2 transfers mint native assets by withdrawing them from `NativeAssetLiquidity` via `LiquidityController`
- Native asset supply on L2 is backed 1:1 by locked ERC20 tokens on L1
- Cross-domain message authentication ensures only trusted L1CGTBridge can trigger finalization
