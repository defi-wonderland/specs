# System Config

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Definitions](#definitions)
  - [Custom Gas Token Flag](#custom-gas-token-flag)
- [Function Specification](#function-specification)
  - [isCustomGasToken](#iscustomgastoken)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Definitions

### Custom Gas Token Flag

The **Custom Gas Token Flag** (`isCustomGasToken`) is a boolean value that indicates
whether the chain is operating in Custom Gas Token mode.

## Function Specification

### isCustomGasToken

Returns true if the gas token is a custom gas token, false otherwise.

- MUST return the result of a call to `superchainConfig.isCustomGasToken()`.