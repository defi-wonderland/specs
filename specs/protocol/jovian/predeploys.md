<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Predeploys](#predeploys)
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

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Predeploys

## Native Asset Liquidity

Address: `0x420000000000000000000000000000000000001C`

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
- MUST revert if called by any address other than `LiquidityController`
- MUST emit `LiquidityDeposited` event

#### `withdraw`

Sends native assets from the contract to the `LiquidityController`, increasing circulating supply.

```solidity
function withdraw(uint256 _amount) external
```

- MUST only be callable by the `LiquidityController` predeploy
- MUST send exactly `_amount` of native asset to the caller
- MUST revert if called by any address other than `LiquidityController`
- MUST revert if the contract balance is insufficient
- MUST emit `LiquidityWithdrawn` event

#### `fund`

Allows funding the contract with native assets, typically used during genesis deployment.

```solidity
function fund() external payable
```

- MUST accept any amount of native asset via `msg.value`
- MUST revert if `msg.value` is zero
- MUST emit `LiquidityFunded` event
- MUST be callable by any address

### Events

#### `LiquidityDeposited`

Emitted when native assets are deposited into the contract.

```solidity
event LiquidityDeposited(address indexed depositor, uint256 amount)
```

Where `depositor` is the address (typically `LiquidityController`) that deposited assets and `amount` is the amount deposited.

#### `LiquidityWithdrawn`

Emitted when native assets are withdrawn from the contract.

```solidity
event LiquidityWithdrawn(address indexed withdrawer, uint256 amount)
```

Where `withdrawer` is the address (typically `LiquidityController`) that withdrew assets and `amount` is the amount withdrawn.

#### `LiquidityFunded`

Emitted when the contract is funded with native assets.

```solidity
event LiquidityFunded(address indexed funder, uint256 amount)
```

Where `funder` is the address that funded the contract and `amount` is the amount funded.

### Invariants

- The contract balance MUST always be greater than or equal to the total amount that can be withdrawn
- Only the `LiquidityController` predeploy MUST be able to call `deposit()` and `withdraw()`
- The contract MUST NOT have any functions that allow arbitrary token transfers
- The contract MUST NOT be pausable or upgradeable beyond proxy upgrades

## Liquidity Controller

Address: `0x420000000000000000000000000000000000001D`

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

- MUST only be callable by the contract owner
- MUST authorize `_minter` to call the `mint()` function
- MUST revert if called by non-owner
- MUST emit `MinterAuthorized` event

#### `mint`

Unlocks native assets from the liquidity pool and sends them to a specified address.

```solidity
function mint(address _to, uint256 _amount) external
```

- MUST only be callable by authorized minters
- MUST call `NativeAssetLiquidity.withdraw(_amount)` to unlock assets
- MUST send exactly `_amount` of native asset to `_to` address
- MUST revert if caller is not an authorized minter
- MUST revert if `NativeAssetLiquidity` has insufficient balance
- MUST emit `AssetsMinted` event

#### `burn`

Locks native assets back into the liquidity pool, reducing circulating supply.

```solidity
function burn() external payable
```

- MUST accept any amount of native asset via `msg.value`
- MUST call `NativeAssetLiquidity.deposit{value: msg.value}()` to lock assets
- MUST be callable by any address
- MUST revert if `msg.value` is zero
- MUST emit `AssetsBurned` event

#### `gasPayingAssetName`

Returns the human-readable name of the gas-paying asset for metadata purposes.

```solidity
function gasPayingAssetName() external view returns (string memory)
```

- MUST return the name of the native asset (e.g., "MyToken")
- MUST be used by WETH9 predeploy for `name()` function
- Returns value used to construct "Wrapped {AssetName}" for WNA

#### `gasPayingAssetSymbol`

Returns the symbol of the gas-paying asset for metadata purposes.

```solidity
function gasPayingAssetSymbol() external view returns (string memory)
```

- MUST return the symbol of the native asset (e.g., "MTK")
- MUST be used by WETH9 predeploy for `symbol()` function
- Returns value used to construct "W{AssetSymbol}" for WNA

### Events

#### `MinterAuthorized`

Emitted when a new minter is authorized by the contract owner.

```solidity
event MinterAuthorized(address indexed minter, address indexed authorizer)
```

Where `minter` is the address being authorized and `authorizer` is the contract owner who authorized them.

#### `AssetsMinted`

Emitted when native assets are unlocked from the liquidity pool and sent to a recipient.

```solidity
event AssetsMinted(address indexed minter, address indexed to, uint256 amount)
```

Where `minter` is the authorized address calling the function, `to` is the recipient address, and `amount` is the amount of native assets minted.

#### `AssetsBurned`

Emitted when native assets are locked into the liquidity pool, reducing circulating supply.

```solidity
event AssetsBurned(address indexed burner, uint256 amount)
```

Where `burner` is the address that burned assets and `amount` is the amount burned.

### Invariants

- Only authorized minters MUST be able to call the `mint()` function
- The contract MUST maintain a list of authorized minters that can only be modified by the owner
- The `burn()` function MUST be callable by any address to ensure permissionless supply reduction
- The contract MUST NOT mint more assets than are available in the `NativeAssetLiquidity`
  contract
- Asset metadata (name/symbol) MUST be immutable once set during deployment
