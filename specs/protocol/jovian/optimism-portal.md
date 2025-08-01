# Optimism Portal

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Function Specification](#function-specification)
  - [donateETH](#donateeth)
  - [depositTransaction](#deposittransaction)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Function Specification

### donateETH

- MUST revert if `systemConfig.isCustomGasToken()` returns `true` and `msg.value > 0`.

### depositTransaction

- MUST revert if `systemConfig.isCustomGasToken()` returns `true` and `msg.value > 0`.
