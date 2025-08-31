# Cluster -1171

**Rank:** #230  
**Count:** 43  

## Label
Hardcoded addresses limit flexibility, expose contracts to misconfiguration, and prevent dynamic updates—increasing attack surface and undermining cross-chain deployability and maintainability.

## Cluster Information
- **Total Findings:** 43

## Examples

### Example 1

**Auto Label:** Hardcoded or incorrect addresses in contracts lead to failed interactions, unintended transfers, or loss of funds due to misalignment with actual on-chain token or protocol addresses.  

**Original Text Preview:**

## \[L-01] Incorrect event emitted in \`setUpdateWeightRunnerAddress()\` function

## Summary

Incorrect event emitted in \`setUpdateWeightRunnerAddress()\` function. When changing the `UpdateWeightRunner.sol` address, the event should emit 2 params - old and new addresses:

```solidity
event UpdateWeightRunnerAddressUpdated(address indexed oldAddress, address indexed newAddress);
```

The issue is that this function does not store the old address in memory, but returns the updated `updateWeightRunner` variable as first param instead of old `updateWeightRunner`:

```solidity
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
        updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
        emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner); 
    }
```

## Impact

Frontend or other off-chain services may display incorrect values, potentially misleading users, disrupt update tracking and make it difficult to find updates.

## Recommendations

```diff
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
+       address oldUpdateWeightRunner = updateWeightRunner;
        updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
-       emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner);
+       emit UpdateWeightRunnerAddressUpdated(oldUpdateWeightRunner, _updateWeightRunner);  
    }
```

---
### Example 2

**Auto Label:** Hardcoded token addresses fail input validation, enabling misconfiguration, incorrect asset handling, and exploitable behavior across chains or in swap flows.  

**Original Text Preview:**

## Context

**File:** ISablierFlowBase.sol  
**Line:** 158

## Description/Recommendation

Change ETH to native tokens to avoid confusion, as the contracts may be deployed on other EVM chains.

## Status

- **Sablier:** Fixed in PR 362.
- **Cantina Managed:** Verified.

---
### Example 3

**Auto Label:** Hardcoded addresses limit flexibility, expose contracts to misconfiguration, and prevent dynamic updates—increasing attack surface and undermining cross-chain deployability and maintainability.  

**Original Text Preview:**

## 01. Relevant GitHub Links

&#x20;

* <https://github.com/Cyfrin/2024-12-alchemix/blob/82798f4891e41959eef866bd1d4cb44fc1e26439/src/StrategyMainnet.sol#L43C1-L47C6>

## 02. Summary

The StrategyMainnet.sol contract initializes the router address once during its constructor and does not provide any function to update or modify it afterward. This design differs from other strategy contracts, such as StrategyArb.sol and StrategyOp.sol, where a setRouter function is available for privileged roles. The inability to update the router introduces a risk if the initially set router address becomes deprecated, compromised, or needs replacing to maintain functionality or security.

## 03. Vulnerability Details

In the StrategyMainnet.sol contract, the router is set during construction:

```Solidity
constructor(
    address _asset,
    address _transmuter,
    string memory _name
) BaseStrategy(_asset, _name) {
    transmuter = ITransmuter(_transmuter);
    require(transmuter.syntheticToken() == _asset, "Asset does not match transmuter synthetic token");
    underlying = ERC20(transmuter.underlyingToken());
    asset.safeApprove(address(transmuter), type(uint256).max);
    
    _initStrategy();
}

/**
 * @dev Initializes the strategy with the router address & approves WETH to be swapped via router
*/
function _initStrategy() internal {
    router = ICurveRouterNG(0xF0d4c12A5768D806021F80a262B4d39d26C58b8D);
    underlying.safeApprove(address(router), type(uint256).max);
    
}

```

No subsequent function allows for changing the router address. This design is inconsistent with other strategies, such as StrategyArb.sol and StrategyOp.sol, which include a setRouter function:

```Solidity
function setRouter(address _router) external onlyManagement {
    router = _router;
    underlying.safeApprove(router, type(uint256).max);
}

```

If the initially set router address becomes non-functional (e.g., due to an upgrade, bug, or exploit), the StrategyMainnet contract cannot be updated to reflect a new, secure, and functional router. This creates a single point of failure and reduces the maintainability and upgradability of the strategy.

## 04. Impact

If the router becomes unavailable or needs to be replaced for security or operational reasons, the StrategyMainnet contract will not be able to route swaps or manage underlying assets as intended. This could lead to:

* Permanent loss of functionality in the strategy’s ability to swap and manage underlying tokens.
* Locked funds if the strategy relies on the router for normal operations.
* Reduced agility in responding to emergencies, vulnerabilities, or protocol changes.

Overall, the impact includes the potential inability to respond to changes in external dependencies, which could compromise the strategy’s profitability and functionality.

## 05. Proof of Concept, Scenario

1. Deploy StrategyMainnet with a valid router address.
2. Later, the chosen router contract is deprecated or found vulnerable.
3. Attempting to update the router to a new secure address is impossible since no setRouter function exists.
4. The strategy remains locked to the old router, unable to adapt to the new environment, potentially causing loss or reduced efficiency.

## 06. Tools Used

Manual Code Review and Foundry

## 07. Recommended Mitigation

Introduce an onlyManagement function setRouter(address \_router) similar to other strategy contracts (StrategyArb.sol, StrategyOp.sol). This would allow privileged roles to update the router address when necessary. For example:

```Solidity
function setRouter(address _router) external onlyManagement {
    require(_router != address(0), "Invalid router address");
    router = ICurveRouterNG(_router);
    underlying.safeApprove(router, type(uint256).max);
}

```

By adding this function, the contract can maintain flexibility, adjust to protocol changes, and respond to emergencies without deploying a new strategy contract.

---
