# FeeVaultInitializer

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Purpose](#purpose)
  - [Constructor](#constructor)
  - [Events](#events)
- [Upgrade Path](#upgrade-path)
- [Alternatives Considered](#alternatives-considered)
- [Pros & Cons Of Both Approaches](#pros--cons-of-both-approaches)
  - [Getter-driven Approach (Current)](#getter-driven-approach-current)
  - [Initializer-driven Approach](#initializer-driven-approach)
  - [Considerations](#considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Purpose

This contract serves as a runbook for deploying new vault implementations while preserving the current vault
configuration. It is intended to be used in a NUT series context to upgrade the vaults to the new
implementation once they are deployed.

### Constructor

On deployment, the constructor will:

- Read and grab the current values for `recipient()`, `withdrawalNetwork()`, and `minWithdrawalAmount()` from
  each live vault (`SequencerFeeVault`, `L1FeeVault`, `BaseFeeVault`, `OperatorFeeVault`).
  Use a `try/catch` block to handle the case where the call fails because the vault is a legacy one, and query
  for the `RECIPIENT()` and `MIN_WITHDRAWAL_AMOUNT()` immutable values instead.
- Deploy a new implementation for each vault, passing the grabbed values per vault as constructor immutables,
  and also passing `L2` as the `WITHDRAWAL_NETWORK` constructor immutable.
- Emit a single event containing the four deployed implementation addresses

Invariants:

- It MUST deploy implementations whose constructor immutables match the current configuration values.
- It MUST emit an event with the deployed implementation addresses.

### Events

Single event with all deployed implementation addresses:

```solidity
event FeeVaultImplementationsDeployed(
    address sequencerFeeVaultImplementation,
    address l1FeeVaultImplementation,
    address baseFeeVaultImplementation,
    address operatorFeeVaultImplementation
);
```

## Upgrade Path

These are the steps to deploy and activate new implementations through NUTs:

0. Precalculate:
   a. `FeeVaultInitializer` address to be deployed
   b. each vault implementation address to be deployed by the `FeeVaultInitializer` address
1. Deploy the `FeeVaultInitializer`. Its constructor deploys the new implementations, keeping the current configuration values.
2. For each vault proxy, call `upgradeTo` (or `upgradeToAndCall` if local policy requires) from the admin or
   via the `address(0)` NUT flow to point the proxy at the emitted implementation addresses (4 different NUTs).

Execute in order so that deployments exist before proxies are upgraded.

## Alternatives Considered

**Initializer-driven storage init after upgrade**: Stores current configs as immutables on a `FeeVaultInitializer` and
grant it permission to call `initialize()` on each vault after proxies are upgraded to the new implementation.

- Flow:
  1. Deploy `FeeVaultInitializer`. Its constructor reads the current config from the live vaults and stores
     those values as immutables.
  2. Deploy a new implementation for each vault, passing the initializer’s address as the allowed initializer
     target (pre-calculated).
  3. From address(0) (NUT) or via admin, call `upgradeTo` for each vault proxy to point to the new
     implementation.
  4. Call `FeeVaultInitializer.initializeVaults()` so it pushes the stored config into each vault via `initialize()`.

```solidity
address public immutable FEE_VAULT_INITIALIZER;
Config public config;

constructor(address _feeVaultInitializer) {
  FEE_VAULT_INITIALIZER = _feeVaultInitializer;
}

function initialize(Config _config) external {
  if (msg.sender != FEE_VAULT_INITIALIZER) revert NotAllowed();
  config = _config;
}
```

## Pros & Cons Of Both Approaches

### Getter-driven Approach (Current)

**Pros**

- Simple and straightforward approach to deploy the new vault implementations while keeping the current configuration values.
- Fewer NUTs are needed on the series, making this process simpler.
- No need to give any permission to the `FeeVaultInitializer` contract to call `initialize()` on the vaults.
- Less code to maintain

**Cons**

- It requires a change to `FeeVaults` getter functions, which is not very clean, and also incurs a minor added
  gas cost while reading the values.

### Initializer-driven Approach

**Pros**

- No need to change `FeeVaults` getter functions.
- No added gas cost while reading the values.

**Cons**

- More NUTs are needed on the series, making this process more complex.
- Requires permission to the `FeeVaultInitializer` contract to call `initialize()` on the vaults.
- More code to maintain: Vaults need to implement the `initialize()` function.

With these pros and cons in mind, we opted for the getter-driven approach since we find it simpler, and no
permissions nor `initialize()` function are needed to be implemented on the vaults.

### Considerations

For chains opting into `FeeSplitter`, ensure `WITHDRAWAL_NETWORK = L2` and `RECIPIENT = FeeSplitter` after
upgrade (via admin setters).
