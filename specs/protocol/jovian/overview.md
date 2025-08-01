# Jovian Network Upgrade

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [Custom Gas Token Support](#custom-gas-token-support)
- [Contract Modifications](#contract-modifications)
- [New Predeploys](#new-predeploys)
- [Execution Layer](#execution-layer)
- [Consensus Layer](#consensus-layer)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

This document is not finalized and should be considered experimental.

The Jovian network upgrade introduces support for Custom Gas Token (CGT) functionality,
allowing OP Stack chains to use ERC20 tokens as their native gas-paying asset instead of ETH.

## Custom Gas Token Support

The Jovian upgrade enables chains to use custom ERC20 tokens as their native asset for gas payments.
This fundamental change requires modifications to several core contracts to prevent ETH from being
accidentally locked in the system when it cannot be properly utilized.

Key features:
- Support for ERC20 tokens as the native gas-paying asset
- Prevention of ETH deposits when Custom Gas Token mode is enabled
- New liquidity management system for native asset supply control
- Backward compatibility maintained for ETH-based chains

## Contract Modifications

Several core contracts are modified in Jovian to support Custom Gas Token functionality:

- **OptimismPortal**: Modified `donateETH` and `depositTransaction` functions to prevent ETH deposits on CGT chains
- **Standard Bridges**: ETH bridging functions revert when Custom Gas Token mode is enabled
- **Cross Domain Messengers**: `sendMessage` function reverts when ETH is sent (`msg.value > 0`) on CGT chains

## New Predeploys

Jovian introduces two new predeploy contracts for Custom Gas Token chains:

- **Native Asset Liquidity** (`0x420000000000000000000000000000000000001C`):
  Central liquidity vault for native asset management
- **Liquidity Controller** (`0x420000000000000000000000000000000000001D`):
  Governance interface for asset minting/burning and metadata management

## Execution Layer

The execution layer changes in Jovian focus on preventing accidental ETH deposits and providing
proper liquidity management for custom gas tokens.

## Consensus Layer

No consensus layer changes are introduced in the Jovian upgrade.
