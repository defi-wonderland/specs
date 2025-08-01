# Withdrawals Modifications for Custom Gas Token

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [L2ToL1MessagePasser Contract Modifications](#l2tol1messagepasser-contract-modifications)
  - [initiateWithdrawal Function](#initiatewithdrawal-function)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## L2ToL1MessagePasser Contract Modifications

### initiateWithdrawal Function

The `initiateWithdrawal` function in the L2ToL1MessagePasser contract requires modifications
to support Custom Gas Token chains.

**Custom Gas Token Restriction:**

The `initiateWithdrawal` function MUST revert if `L1Block.isCustomGasToken()` returns `true` and `msg.value > 0`.

This prevents ETH from being included in withdrawal transactions on Custom Gas Token chains,
where ETH cannot be properly handled or bridged back to L1. The restriction ensures that users
cannot accidentally lose ETH by attempting to withdraw it on chains where it has no value or utility.

```solidity
function initiateWithdrawal(address _target, uint256 _gasLimit, bytes memory _data) payable external {
    // Custom Gas Token check
    if (L1Block.isCustomGasToken() && msg.value > 0) {
        revert("ETH withdrawals not supported on Custom Gas Token chains");
    }
    
    // ... rest of function implementation
}
```

This modification works in conjunction with other Custom Gas Token restrictions across the system
to ensure ETH cannot become locked or lost in contracts that cannot properly handle it in a 
custom gas token environment.