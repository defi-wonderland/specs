# ConditionalDeployer

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Summary](#summary)
- [Definitions](#definitions)
  - [Create2Deployer](#create2deployer)
  - [Address Collision](#address-collision)
- [Assumptions](#assumptions)
  - [A-01: The initcode received is correct](#a-01-the-initcode-received-is-correct)
  - [A-02: The Create2Deployer exists in the target chain](#a-02-the-create2deployer-exists-in-the-target-chain)
  - [A-03: Sufficient gas is available for deployment operations](#a-03-sufficient-gas-is-available-for-deployment-operations)
- [Invariants](#invariants)
  - [I-01: Non-Reverting Execution with Valid Inputs](#i-01-non-reverting-execution-with-valid-inputs)
    - [Impact](#impact)
- [Functions](#functions)
  - [`deploy`](#deploy)
    - [Behavior](#behavior)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Summary

A periphery contract acting as a wrapper for `Create2Deployer` predeploy. The `ConditionalDeployer` ensures that
if a contract's bytecode is unchanged, its implementation address will remain unchanged across upgrades.

The contract receives initcode and determines whether an address collision will occur when deploying the contract using `create2`.
If no collision is detected, it forwards the initcode to the `Create2Deployer` contract.
If a collision is detected (indicating the contract is already deployed at that address),
it returns the address without performing any action.

The `ConditionalDeployer` MUST always succeed when called with properly formed inputs, ensuring that upgrade transactions
can always be included in NUT bundles regardless of whether contracts have been previously deployed.

## Definitions

### Create2Deployer

The deterministic deployment proxy contract that uses CREATE2 opcode to deploy contracts at predictable addresses
based on the initcode and salt. The `ConditionalDeployer` forwards deployment to this contract when no address
collision is detected.

### Address Collision

A situation where a contract already exists at the address that would be computed by CREATE2 opcode
for the given initcode and salt.

## Assumptions

### A-01: The initcode received is correct

The contract assumes that the initcode provided to the `deploy` function is valid and will result in a properly formed
contract when deployed.

### A-02: The Create2Deployer exists in the target chain

The contract assumes that the `Create2Deployer` predeploy exists at its expected address.

### A-03: Sufficient gas is available for deployment operations

The contract assumes that sufficient gas is available for checking address collisions and forwarding calls to `Create2Deployer` to perform contract deployments.

## Invariants

### I-01: Non-Reverting Execution with Valid Inputs

The `deploy` function MUST always succeed when called with properly formed initcode, regardless of whether
an address collision is detected or whether the contract has been previously deployed.
The function MUST always return an address (either the newly deployed contract address
or the existing contract address in case of collision).

#### Impact

**Severity: Critical**

If this invariant is violated, NUT bundle transactions may fail during hard fork activation, preventing network upgrades.

## Functions

### `deploy`

Deploys a contract using `Create2Deployer` if no collision is detected.

```solidity
function deploy(uint256 _value, bytes32 _salt, bytes memory _initCode) external returns (address);
```

#### Behavior

- MUST always succeed when called with properly formed parameters
- MUST determine whether an address collision will occur for the given initcode and salt
- If no collision is detected:
  - MUST forward the initcode and value to the `Create2Deployer` contract
  - MUST return the deployed contract address
- If a collision is detected:
  - MUST return the existing contract address without performing deployment
  - MUST NOT revert
