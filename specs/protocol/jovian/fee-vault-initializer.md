<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [FeeVaultInitializer](#feevaultinitializer)
    - [Constructor](#constructor)
    - [Events](#events)
  - [Upgrade Path](#upgrade-path)
    - [Considerations](#considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# FeeVaultInitializer

This contract serve as a runbook to deploy the new vaults implementations keeping the configuration values of the current vaults to be upgraded. It is intended to be used in a NUT series context to upgrade the vaults to the new implementation once they are deployed.

### Constructor

On deployment, the constructor will:

- Read the current configuration from each live vault (`SequencerFeeVault`, `L1FeeVault`, `BaseFeeVault`, `OperatorFeeVault`): recipient, withdrawal network, and minimum withdrawal amount
- Deploy a new implementation for each vault, passing these values as constructor immutables
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
)
```

## Upgrade Path

These are the steps to deploy and activate new implementations through NUTs:

0. Precalculate:
   a. `FeeVaultInitializer` address to be deployed
   b. each of the vaults implementation addresses to be deployed by the `FeeVaultInitializer` address
1. Deploy the `FeeVaultInitializer`. Its constructor deploys the new implementations keeping the current configuration values.
2. For each vault proxy, call `upgradeTo` (or `upgradeToAndCall` if local policy requires) from the admin or via the `address(0)` NUT flow to point the proxy at the emitted implementation addresses (4 different NUTs).

Execute in order so deployments exist before proxies are upgraded.

### Considerations

For chains opting into `FeeSplitter`, ensure `WITHDRAWAL_NETWORK = L2` and `RECIPIENT = FeeSplitter` after upgrade (via admin setters).
