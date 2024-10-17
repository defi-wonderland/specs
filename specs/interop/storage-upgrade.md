# Migrated Liquidity storage update

## Overview

To fully migrate liquidity from `OptimismMintableERC20` tokens to `OptimismSuperchainERC20`, the `OptimismMintableERC20Factory` needs to track its deployment history.

To achieve this, a [Hash Onion](https://github.com/ethereum-optimism/design-docs/blob/main/protocol/superchain-erc20/storage-upgrade.md#2-hash-onion) solution will be used:

- The `OptimismMintableERC20Factory` predeploy will include functionality to set and store the Hash Onion, as well as verify and record deployment data.
- A Foundry script will be added to generate the Hash Onion from a list of local and remote tokens.
- A `superchain-ops` task will be added to set the Hash Onion from the L2 `ProxyAdmin` owner.

## `OptimismMintableERC20Factory` predeploy

Create a new `OptimismMintableERC20FactoryInterop` contract that inherits from `OptimismMintableERC20Factory` and add the following functionality:

```solidity
bytes32 public onionHash;

function setOnionHash(bytes32 _onionHash) external {
    if (msg.sender != Predeploys.PROXY_ADMIN) revert("Unauthorized");
    if (hashOnion != 0) revert("Already initialized");

    onionHash = _onionHash;
}

function verifyAndStore(
	address localToken,
	address remoteToken,
	bytes32 innerLayer
) public {
	// Concatenate localToken and remoteToken using abi.encodePacked
	bytes memory data = abi.encodePacked(localToken, remoteToken);

	// Recompute the hash of the previous layer + data
	bytes32 computedHash = keccak256(abi.encodePacked(innerLayer, data));

	// Ensure the computed hash matches the current stored layer
	require(computedHash == hashOnion, "Invalid layer provided.");

	// Store the localToken and remoteToken in the mapping
	deployments[localToken] = remoteToken;

	// Emit event for logging
	emit AddressesStored(localToken, remoteToken);

	// Update the current layer to the previous one for the next submission
	onionHash = innerLayer;
}
```

Also a function that receives local and remote tokens arrays can be implemented so that many layers can be verified and stored together.

## Generate Hash Onion

Create a Foundry script that generate the final onion hash based on a list of local and remote tokens.

```solidity
function generateOnionHash(
	address[] calldata localTokens,
    address[] calldata remoteTokens
) public {
    require(localTokens.length == remoteTokens.length, "Invalid arrays");

    bytes32 onionHash = keccak256(0);

	for (uint256 i; i < localTokens.length; i++) {
        onionHash = keccak256(abi.encodePacked(onionHash, abi.encodePacked(localTokens[i], remoteTokens[i])));
    }

    return onionHash;
}
```

## `superchain-ops` task

Each chain will need to generate its own hash onion based on its specific token list. This onion will then be set via the setter function in the `OptimismMintableERC20Factory` predeploy.

The L2 `ProxyAdmin` owner will be responsible for performing the upgrade. On OP Mainnet, this is the aliased L1 governance multisig. To facilitate this, a task will be added to the [`superchain-ops` repository](https://github.com/ethereum-optimism/superchain-ops), enabling the multisig to execute it.
