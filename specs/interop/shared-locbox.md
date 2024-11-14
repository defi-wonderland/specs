# Shared Lockbox

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Overview](#overview)
- [Implementation](#implementation)
  - [`SharedLockbox`](#sharedlockbox)
  - [`SuperchainConfig`](#superchainconfig)
  - [`OptimismPortal`](#optimismportal)
  - [`LiquidityMigrator`](#liquiditymigrator)
  - [Future Considerations / Additional Notes](#future-considerations--additional-notes)
- [Upgrade and migration process](#upgrade-and-migration-process)
  - [Batch transaction process](#batch-transaction-process)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Overview

With interoperable ETH, withdrawals may fail if the referenced `OptimismPortal` lacks sufficient ETH
— making it mandatory to withdraw from the L2 where the ETH was originally deposited.
To improve the Superchain's interoperable ETH withdrawal user experience and avoid this issue,
the `SharedLockbox` serves as a solution.
It unifies ETH L1 liquidity in a single contract, enabling seamless withdrawals of ETH from any OP chain in the Superchain,
regardless of where the ETH was initially deposited.

## Implementation

### `SharedLockbox`

The `SharedLockbox` contract is designed to manage the unified ETH liquidity for the Superchain.
It provides two main functions: `lockETH` for depositing ETH into the lockbox,
and `unlockETH` for withdrawing ETH from the lockbox.
These functions are called by the `OptimismPortal` contracts to manage the shared ETH liquidity
when making deposits or finalizing withdrawals.

**Storage layout**

The `SharedLockbox` contract maintains a mapping of authorized `OptimismPortal` addresses.
This ensures that only approved portals can interact with the shared liquidity pool.

```solidity
// OptimismPortals that are part of the dependency cluster.
mapping(address _portal => bool) internal _authorizedPortals;
```

**Interface**

The `ShareLockbox` interface is the following:

```solidity
function lockETH() external payable;

function unlockETH(uint256 _value) external;

function authorizePortal(address _portal) external;

event ETHLocked(address indexed portal, uint256 amount);

event ETHUnlocked(address indexed portal, uint256 amount);

event AuthorizedPortal(address indexed portal);
```

A possible implementation could look like this:

```solidity
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

### `SuperchainConfig`

This contract will manage and keep track the dependency graph.
It will be queried as the source of truth to get which chains are part of the Superchain.
It assumes the role of a `dependencyManager` per `SystemConfig` contract involved.
It will also allow to add a chain to the op-governed cluster and update each chain’s dependency set.

**Storage layout**

The `SuperchainConfig` contract will contain the following storage layout:

1. `SHARED_LOCKBOX`: An immutable address pointing to the Shared Lockbox contract.
   This address is immutable because there's only one `SharedLockbox` for each cluster.
2. `systemConfigs`: A mapping that associates chain IDs with their respective SystemConfig addresses.
   This will be used when updating dependencies along each chain.
3. `_dependencySet`: An EnumerableSet that stores the current list of chain IDs in the dependency set.
   This should replicate the same state as the one stored in the `L1BlockInterop` on L2.

```solidity
// The Shared Lockbox contract
address internal immutable SHARED_LOCKBOX;

// Mapping from chainId to SystemConfig address
mapping(uint256 _chainId => ISystemConfig) public systemConfigs;

// Current dependency set list
EnumerableSet.UintSet internal _dependencySet;
```

**`addChain`**

The `addChain` function adds a new chain to the op-governed cluster.
This function can only be called by the `updater`.
It tries to add the chain’s ID to the dependency set, making sure it’s not already included.
The function stores its system config and updates dependencies by linking the new chain with
all existing chains in the set, making a full mesh graph.
It finally adds the portal to the `ShareLockbox` whitelist.

```solidity
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
```

**Getters**

These getter are needed due to using an EnumerableSet.

```solidity
function dependencySet() external view returns (uint256[] memory) {
    return dependencySet.values();
}

function isInDependencySet(uint256 _chainId) public view returns (bool) {
    return dependencySet.contains(_chainId);
}
```

### `OptimismPortal`

This contract will need an upgrade to integrate the `SharedLockbox` and start using the shared ETH liquidity.

**Storage layout**

The storage layout changes include adding the `SharedLockbox` address.
This address will be immutable because all portals will point to the same lockbox, and this address should not change.

```solidity
// The Shared Lockbox contract
address internal immutable SHARED_LOCKBOX;
```

**Integrating `SharedLockbox`**

The integration with the `SharedLockbox` involves adding extra steps when executing deposit transactions
or finalizing withdrawal transactions.
These steps include locking and unlocking ETH without altering other aspects of the current portal implementation.
To implement this solution, we need the following changes:

- When calling `finalizeWithdrawalTransactionExternalProof` if the token is `ETHER` and `tx.value` is different than zero,
  then it should call `unlockETH` on the `SharedLockbox` with the `tx.value`.
- When calling `depositTransaction` if the token is `ETHER` and `msg.value` is different than zero,
  then it should call `lockETH` on the `SharedLockbox` with the `msg.value`.

### `LiquidityMigrator`

This contract is meant to be used as intermediate step for the liquidity migration.
It main purpose is to transfer the whole ETH balance from the `OptimismPortal` to the `SharedLockbox`.

A possible migrate function implementation can be:

```solidity
// The Shared Lockbox contract
address internal immutable SHARED_LOCKBOX;

function migrateETH() external {
    SHARED_LOCKBOX.lockETH{ value: address(this).balance }();
}
```

### Future Considerations / Additional Notes

- Should we add a check in `addChain` to ensure the `chainId` and `systemConfig` match?
  Where should the source of truth reside? Could `SystemConfig` store the `chainId`?

## Upgrade and migration process

Based on the latest decision, where a chain that joins the dependency set will not be allowed to exit
(making it an irreversible process), we are simplifying the on-chain chains list by assuming that joining
the Shared Lockbox is equivalent to joining the op-governed dependency set.
This also means we can consider shortening the required steps.

The migration process consists of four main points:

- `OptimismPortal` code upgrade (includes the `SharedLockbox` integration)
- Move ETH liquidity from `OptimismPortal` to `SharedLockbox`
- Set `SuperchainConfig` as dependency manager on `SystemConfig`
- Add the chain to the op-governed dependency set

The migration process needs some prerequisites:

- `SharedLockbox` deployed
- `SuperchainConfig` upgraded to manage the dependency set

**`OptimismPortal` code upgrade**

The `OptimismPortal` will start locking and unlocking ETH through the `SharedLockbox`.
It will continue to handle deposits and withdrawals but won't directly hold the ETH liquidity.
To set this up, we'll call the upgrade function via `ProxyAdmin` to implement the new code,
which includes the necessary `SharedLockbox` integration.
The `SharedLockbox` address will be set during the `initialize` function. After this step,
the portal can't process deposits and withdrawals until the chain become registered in `SuperchainConfig`.

**Migrate ETH liquidity from `OptimismPortal` to `SharedLockbox`**

The ETH will be transferred from the `OptimismPortal` to the `SharedLockbox` using an intermediate contract.
This contract functions similarly to the `StorageSetter`, being updated prior to the real implementation.
Its sole purpose is to transfer the ETH balance.
This approach eliminates the need for additional code to move the liquidity to the lockbox later.

**Set `SuperchainConfig` as dependency manager on `SystemConfig`**

The `SystemConfig` manages the dependency updates from L1 to L2 by making a deposit through the portal.
It uses access control, allowing only a designated `dependencyManager` to call it.
We must set the `SuperchainConfig` as the `dependencyManager` since it will be the contract responsible
to handle the op-governed dependency set.
This step is crucial before adding a chain to the dependency set, as it's necessary for keeping the
dependency set synchronized between L1 and L2.

There are two alternatives to accomplish this:

- Upgrade the code to `SystemConfigInterop` and call `initialize` with the dependency manager address
- Add a setter function for the dependency manager

**Add the chain to the op-governed dependency set**

The `SuperchainConfig` contract will be responsible for storing and managing the dependency set.
Its `addChain` function will add a chain to the dependency set and call the `SystemConfig` of each chain
to keep them in sync.
It will also whitelist the corresponding `OptimismPortal`, enabling it to lock and unlock ETH from the `SharedLockbox`.
Once this process is complete, the system will be ready to process deposits and withdrawals.

### Batch transaction process

The most efficient approach is to handle the entire migration process in a single batched transaction.
This transaction will consist of:

1. Call `upgradeAndCall` in the `ProxyAdmin` for the `SystemConfig`
   - Updating provisionally to the `StorageSetter` to zero out the initialized slot.
2. Call `upgradeAndCall` in the `ProxyAdmin` for the `SystemConfig`
   - Calling `initialize` with the `SuperchainConfig` address as the dependency manager
3. Call `addChain` in the `SuperchainConfig`
   - Sending chain ID + system config address
4. Call `upgradeAndCall` in the `ProxyAdmin` for the `OptimismPortal`
   - Update provisionally to the `LiquidityMigrator` to transfer the whole ETH balance to the `SharedLockbox` in this call.
5. Call `upgrade` in the `ProxyAdmin` for the `OptimismPortal`
   - The `SharedLockbox` address is set as immutable in the new implementation

The L1 ProxyAdmin owner will execute this transaction. As the entity responsible for updating contracts,
it has the authority to perform the first two steps.
For the third step, the L1PAO has to be set as authorized for adding a chain to the op-governed dependency set
on the `SuperchainConfig` when initializing.
This process can be set as a [superchain-ops](https://github.com/ethereum-optimism/superchain-ops) task.
