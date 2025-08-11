# Optimism Portal

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Function Specification](#function-specification)
  - [donateETH](#donateeth)
  - [depositTransaction](#deposittransaction)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Rationale

The OptimismPortal’s ETH-related logic must revert when Custom Gas Token mode is enabled to prevent ETH from acting as the native asset. Since the client side does not discern native asset supply creation, allowing ETH deposits would incorrectly imply that it can be minted in the chain.

## Function Specification

### donateETH

- MUST revert if `systemConfig.isCustomGasToken()` returns `true` and `msg.value > 0`.

### depositTransaction

- MUST revert if `systemConfig.isCustomGasToken()` returns `true` and `msg.value > 0`.
