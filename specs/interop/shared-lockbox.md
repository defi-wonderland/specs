# Shared Lockbox

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Design](#design)
  - [Contracts introductions](#contracts-introductions)
    - [`SharedLockbox`](#sharedlockbox)
      - [**Interface and properties**](#interface-and-properties)
      - [**Events**](#events)
    - [`LiquidityMigrator`](#liquiditymigrator)
      - [**Interface and properties**](#interface-and-properties-1)
  - [Contracts upgrades](#contracts-upgrades)
    - [`SuperchainConfig`](#superchainconfig)
      - [**Interface and properties**](#interface-and-properties-2)
      - [**Events**](#events-1)
    - [`OptimismPortal`](#optimismportal)
      - [**Interface and properties**](#interface-and-properties-3)
- [Reference implementation](#reference-implementation)
  - [`SharedLockbox`](#sharedlockbox-1)
  - [`LiquidityMigrator`](#liquiditymigrator-1)
  - [`SuperchainConfig`](#superchainconfig-1)
  - [`OptimismPortal`](#optimismportal-1)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

With interoperable ETH, withdrawals will fail if the referenced `OptimismPortal` lacks sufficient ETH.
The `SharedLockbox` improves the Superchain's interoperable ETH withdrawal user experience and avoid this issue.
To do so, it unifies ETH L1 liquidity in a single contract (`SharedLockbox`), enabling seamless withdrawals of ETH
from any OP chain in the Superchain, regardless of where the ETH was initially deposited.

## Design

### Contracts introductions

#### `SharedLockbox`

The `SharedLockbox` contract is designed to manage the unified ETH liquidity for the Superchain.
It implements two main functions: `lockETH` for depositing ETH into the lockbox,
and `unlockETH` for withdrawing ETH from the lockbox.
These functions are called by the `OptimismPortal` contracts to manage the shared ETH liquidity
when making deposits or finalizing withdrawals.

##### **Interface and properties**

**`lockETH`**

Deposits and locks ETH into the lockbox's liquidity pool.

- The function MUST accept ETH.
- Only authorized `OptimismPortal` addresses SHOULD be allowed to interact.
- The function MUST emit the `ETHLocked` event with the `portal` that called it and the `amount`.

```solidity
function lockETH() external payable;
```

**`unlockETH`**

Withdraws a specified amount of ETH from the lockbox's liquidity pool.

- Only authorized `OptimismPortal` addresses SHOULD be allowed to interact.
- The function MUST emit the `ETHUnlocked` event with the `portal` that called it and the `amount`.

```solidity
function unlockETH(uint256 _value) external;
```

**`authorizePortal`**

Grants authorization to a specific `OptimismPortal` contract.

- Only `SuperchainConfig` address SHOULD be allowed to interact.
- The function MUST add the specified address to the mapping of authorized portals.
- The function MUST emit the [`AuthorizedPortal`](#events) event when a portal is successfully added.

```solidity
function authorizePortal(address _portal) external;
```

##### **Events**

**`ETHLocked`**

MUST be triggered when `lockETH` is called

```solidity
event ETHLocked(address indexed portal, uint256 amount);
```

**`ETHUnlocked`**

MUST be triggered when `unlockETH` is called

```solidity
event ETHUnlocked(address indexed portal, uint256 amount);
```

**`AuthorizedPortal`**

MUST be triggered when `authorizePortal` is called

```solidity
event AuthorizedPortal(address indexed portal);
```

#### `LiquidityMigrator`

This contract is meant to be used as an intermediate step for the liquidity migration.
Its unique purpose is to transfer the whole ETH balance from `OptimismPortal` to `SharedLockbox`.
This approach avoids adding extra code to the `initialize` function, which could be prone to errors in future updates.

##### **Interface and properties**

**`migrateETH`**

Transfers the entire ETH balance from the `OptimismPortal` to the `SharedLockbox`.

- It MUST transfer the whole ETH balance to the `SharedLockbox` when called.

```solidity
function migrateETH() external;
```

### Contracts upgrades

#### `SuperchainConfig`

This contract will be updated to manage and keep track of the dependency graph.
It will be queried as the source of truth to get which chains are part of the Superchain.
It assumes the role of a `dependencyManager` per `SystemConfig` contract involved.
It will also allow to add a chain to the op-governed cluster and update each chain’s dependency set.

##### **Interface and properties**

The `SuperchainConfig` contract will add the following storage layout and function:

**`SHARED_LOCKBOX`**

- An immutable address pointing to the `SharedLockbox` contract.
- This address MUST be immutable because there's only one `SharedLockbox` for each cluster.

**`systemConfigs`**

- A mapping that associates chain IDs with their respective SystemConfig addresses.
- This will be used when updating dependencies along each chain.

**`dependencySet`**

- An `EnumerableSet` that stores the current list of chain IDs in the dependency set.
- This MUST replicate the same state as the one stored in the `L1BlockInterop` on L2 for each chain.

**`addChain`**

The `addChain` function adds a new chain to the op-governed cluster.

- The function SHOULD only be callable by the authorized `updater` role of the `SuperchainConfig`.
- The function MUST NOT add a chain ID to the dependency set if it is already included.
- The function MUST update all chains dependencies through deposit txs to form a complete mesh graph.
- The function MUST store the provided `SystemConfig` address in the `systemConfigs` mapping.
- The function MUST allowlist the new chain's `OptimismPortal` in the `SharedLockbox`.
- The function MUST emit the `ChainAdded` event with the `chainId` and
  its corresponding `SystemConfig` and `OptimismPortal`.

```solidity
function addChain(uint256 _chainId, address _systemConfig) external;
```

##### **Events**

**`ChainAdded`**

MUST be triggered when `addChain` is called

```solidity
event ChainAdded(uint256 indexed chainId, address indexed systemConfig, address indexed portal);
```

#### `OptimismPortal`

This contract will need an upgrade to integrate the `SharedLockbox` and start using the shared ETH liquidity.

##### **Interface and properties**

The `OptimismPortal` contract will add the following storage layout and modified functions:

**`SHARED_LOCKBOX`**

- An immutable address pointing to the `SharedLockbox` contract.
- This address MUST be immutable because all `OptimismPortals` will point to the same `SharedLockbox`,
  and this address SHOULD not change.

**Integrating `SharedLockbox`**

The integration with the `SharedLockbox` involves adding extra steps when executing deposit transactions
or finalizing withdrawal transactions.
These steps include locking and unlocking ETH without altering other aspects of the current `OptimismPortal` implementation.
To implement this solution, the following changes are needed:

**`depositTransaction`**

Calls `lockETH` on the `SharedLockbox` with the `msg.value`.

- The function MUST call `lockETH` on the `SharedLockbox` if:
  - The token is `ETHER`.
  - `msg.value` is greater than zero.

**`finalizeWithdrawalTransactionExternalProof`**

Calls `unlockETH` on the `SharedLockbox` with the `tx.value`.

- The function MUST call `unlockETH` on the `SharedLockbox` if:
  - The token is `ETHER`.
  - `tx.value` is greater than zero.

## Reference implementation

### `SharedLockbox`

An example implementation could look like this:

```solidity
// OptimismPortals that are part of the dependency cluster.
mapping(address _portal => bool) internal _authorizedPortals;

function lockETH() external payable {
    require(_authorizedPortals[msg.sender], "Unauthorized");

    emit ETHLocked(msg.sender, msg.value);
}

function unlockETH(uint256 _value) external {
    require(_authorizedPortals[msg.sender], "Unauthorized");

    // Using `donateETH` to not trigger a deposit
    IOptimismPortal(msg.sender).donateETH{ value: _value }();

    emit ETHUnlocked(msg.sender, _value);
}

function authorizePortal(address _portal) external {
    require(msg.sender == superchainConfig, "Unauthorized");

    _authorizedPortals[_portal] = true;

    emit AuthorizedPortal(_portal);
}
```

### `LiquidityMigrator`

A possible migrate function implementation could be:

```solidity
// The Shared Lockbox contract
address internal immutable SHARED_LOCKBOX;

function migrateETH() external {
    SHARED_LOCKBOX.lockETH{ value: address(this).balance }();
}
```

### `SuperchainConfig`

An example implementation could look as follow:

```solidity
// The Shared Lockbox contract
address internal immutable SHARED_LOCKBOX;

// Mapping from chainId to SystemConfig address
mapping(uint256 _chainId => ISystemConfig) public systemConfigs;

// Current dependency set list
EnumerableSet.UintSet internal _dependencySet;

event ChainAdded(uint256 indexed chainId, address indexed systemConfig, address indexed portal);

function addChain(uint256 _chainId, address _systemConfig) external {
    require(msg.sender == updater(), "Unauthorized");

    // Add to the dependency set and check it is not already added (`add()` returns false if it already exists)
    require(_dependencySet.add(_chainId), "Chain already added");

    // Store the system config
    systemConfigs[_chainId] = _systemConfig;

    // Loop through the dependency set and update the dependency for each chain
    for (uint256 i; i < _dependencySet.length() - 1; i++) {
        uint256 currentId = _dependencySet.at(i);

        // Skip recently added chain
        if (_chainId == currentId) continue;

        systemConfigs[currentId].addDependency(_chainId);

        systemConfigs[_chainId].addDependency(currentId);
    }

    address portal = _systemConfig.optimismPortal();

    // Authorize the portal on the shared lockbox
    SHARED_LOCKBOX.authorizePortal(portal);

    emit ChainAdded(_chainId, _systemConfig, portal);
}

function dependencySet() external view returns (uint256[] memory) {
    return dependencySet.values();
}

function isInDependencySet(uint256 _chainId) public view returns (bool) {
    return dependencySet.contains(_chainId);
}
```

### `OptimismPortal`

Functions upgrades could look like this:

```solidity
// The Shared Lockbox contract
address internal immutable SHARED_LOCKBOX;

function depositTransaction(...) public payable metered(_gasLimit) {
    ...

    if (token == Constants.ETHER && msg.value != 0) {
        // Lock the ETH in the shared lockbox.
        SHARED_LOCKBOX.lockETH{ value: msg.value }();
    }

    ...
}

function finalizeWithdrawalTransactionExternalProof(
    Types.WithdrawalTransaction memory _tx,
    address _proofSubmitter
)
    public
{
    ...

    if (token == Constants.ETHER) {
        // Unlock and receive the ETH from the shared lockbox.
        if (_tx.value != 0) SHARED_LOCKBOX.unlockETH(_tx.value);

        success = SafeCall.callWithMinGas(_tx.target, _tx.gasLimit, _tx.value, _tx.data);
    } else {
       ...
    }

    ...
}
```
