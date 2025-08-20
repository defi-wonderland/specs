<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [FeeVaultInitializer](#feevaultinitializer)
    - [Functions](#functions)
      - [`migrate`](#migrate)
    - [Events](#events)
    - [NUTs (Network Upgrade Transaction)](#nuts-network-upgrade-transaction)
    - [Considerations](#considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# FeeVaultInitializer

This contract bundles the migration and configuration steps required to make Fee Vaults compatible with the `FeeSplitter`. It upgrades each vault proxy to the new implementation, initializes them, and applies the current configuration in one go. The goal is to make the full flow a single, well-defined operation that reduces UX complexity and the risk of mistakes.

### Functions

#### `migrate`

For each vault (`SequencerFeeVault`, `L1FeeVault`, `BaseFeeVault`, `OperatorFeeVault`), it will:

- Get the current configuration of the vault
- Deploy the new implementation
- Initialize the vault with the current configuration

The function does not perform any auth checks. No params are required because the values are queried from the existing contracts.
In case an update is needed, the proxy admin can update it through the setters.

```solidity
function migrate() external;
```

- It MUST initialize the newly deployed vault's storage variables with the current configuration values from the existing constants.
- Vault's balance MUST remain unchanged.

### Events

Events are per-vault emitted:

```solidity
event SequencerFeeVaultUpgraded(
    address indexed oldImplementation,
    address indexed newImplementation,
    address recipient,
    Types.WithdrawalNetwork network,
    uint256 minWithdrawalAmount
)

event L1FeeVaultUpgraded(
    address indexed oldImplementation,
    address indexed newImplementation,
    address recipient,
    Types.WithdrawalNetwork network,
    uint256 minWithdrawalAmount
)

event BaseFeeVaultUpgraded(
    address indexed oldImplementation,
    address indexed newImplementation,
    address recipient,
    Types.WithdrawalNetwork network,
    uint256 minWithdrawalAmount
)

event OperatorFeeVaultUpgraded(
    address indexed oldImplementation,
    address indexed newImplementation,
    address recipient,
    Types.WithdrawalNetwork network,
    uint256 minWithdrawalAmount
)
```

### NUTs (Network Upgrade Transaction)

For it to be able to upgrade the vault implementations and initialize them, it has to be proxied by the `ProxyAdmin` contract.
To make this possible, a series of NUTs will be required:

1. Deploy the `FeeVaultInitializer` contract using CREATE2 Preinstall, so the address can be precalculated for the next NUTs.
2. Upgrade the `ProxyAdmin` implementation to point to the `FeeVaultInitializer` contract.
3. Call the `FeeVaultInitializer.migrate()` function on the `ProxyAdmin` proxy.
4. Reset the `ProxyAdmin` implementation to the previous or default one.

It's important to execute them sequentially in the defined order for the entire batch to succeed.

### Considerations

For the migration process to be successful, on all the current vaults, the `WITHDRAWAL_NETWORK` MUST be set to `WithdrawalNetwork.L2` and the `RECIPIENT` MUST be set to the `FeeSplitter` address.
