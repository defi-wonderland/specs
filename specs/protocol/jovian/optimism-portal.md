<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [OptimismPortal Modifications for Custom Gas Token](#optimismportal-modifications-for-custom-gas-token)
  - [Overview](#overview)
  - [Modified Functions](#modified-functions)
    - [`donateETH`](#donateeth)
    - [`depositTransaction`](#deposittransaction)
  - [Security Considerations](#security-considerations)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# OptimismPortal Modifications for Custom Gas Token

## Overview

The OptimismPortal contract requires modifications to support Custom Gas Token chains.
These changes ensure that ETH cannot be accidentally locked in the portal when a custom gas token is being used.

## Modified Functions

### `donateETH`

Allows any address to donate ETH to the contract without triggering a deposit to L2.

- MUST accept ETH payments via the payable modifier.
- MUST revert if `systemConfig.isCustomGasToken()` returns `true` and `msg.value > 0`.
- MUST not perform any state-changing operations.
- MUST not trigger a deposit transaction to L2.

### `depositTransaction`

Accepts deposits of ETH and data, and emits a TransactionDeposited event for use in
deriving deposit transactions. Note that if a deposit is made by a contract, its
address will be aliased when retrieved using `tx.origin` or `msg.sender`. Consider
using the CrossDomainMessenger contracts for a simpler developer experience.

- MUST lock any ETH value (msg.value) in the ETHLockbox contract.
- MUST revert if `systemConfig.isCustomGasToken()` returns `true` and `msg.value > 0`.
- MUST revert if the target address is not address(0) for contract creations.
- MUST revert if the gas limit provided is too low based on the calldata size.
- MUST revert if the calldata is too large (> 120,000 bytes).
- MUST transform the sender address to its alias if the caller is a contract.
- MUST emit a TransactionDeposited event with the from address, to address, deposit version, and opaque data.

## Security Considerations

These modifications prevent ETH from being locked in the OptimismPortal contract on Custom Gas Token chains,
where ETH cannot be used to mint native assets on L2. The restrictions ensure that users cannot accidentally
lose ETH by sending it to contracts that cannot properly handle it in a custom gas token environment.

The `donateETH` and `depositTransaction` functions both include checks that prevent ETH deposits when
Custom Gas Token mode is enabled, protecting users from accidentally losing ETH that cannot be properly
bridged or utilized on the L2.
