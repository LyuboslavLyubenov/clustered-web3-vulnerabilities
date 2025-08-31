# Cluster -1335

**Rank:** #433  
**Count:** 9  

## Label
Unauthorized access to critical pool functions due to missing caller validation and permission checks, enabling state manipulation, malicious rule setting, and griefing of offchain monitoring systems.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Unauthorized function execution via insufficient access control and signature validation, enabling attackers to exploit or drain funds through arbitrary or impersonated calls.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

The `OmoVault.sol` contract implements some principles of the ERC7540 standard, Asynchronous redeem shares for assets.
However, users are still able to withdraw directly using `ERC4626.sol#withdraw()`

Also, `_validateSignature()` function in `DynamicAccount.sol` contract accepts the user wallet as a signer, through the Entry Point.
So, the user has the same privileges as the agent.

Malicious users can call `withdraw()` from the vault directly to receive their funds.
Next, he will transfer all the funds and the UniswapV3 positions out of their Dynamic Account

As a result, he will duple his first deposit in the vault.

## Recommendations

1- Override `withdraw()` function in `OmoVault.sol` and make it revert.
2- In `_validateSignature()` you need to check what function can be called if the signer is the owner

---
### Example 2

**Auto Label:** Unauthorized access to critical pool functions due to missing caller validation and permission checks, enabling state manipulation, malicious rule setting, and griefing of offchain monitoring systems.  

**Original Text Preview:**

## Summary

The [`UpdateWeightRunner::setRuleForPool`](https://github.com/Cyfrin/2024-12-quantamm/blob/a775db4273eb36e7b4536c5b60207c9f17541b92/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L235) lacks validation that allows anyone to set rules for any address which may or may not be a pool, allowing attackers to grief spam events.

## Vulnerability Details

The [`UpdateWeightRunner::setRuleForPool`](https://github.com/Cyfrin/2024-12-quantamm/blob/a775db4273eb36e7b4536c5b60207c9f17541b92/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L235) is used for setting rules for a pool

```solidity
    function setRuleForPool(IQuantAMMWeightedPool.PoolSettings memory _poolSettings) external {
        require(address(rules[msg.sender]) == address(0), "Rule already set");
        require(_poolSettings.oracles.length > 0, "Empty oracles array");
        require(poolOracles[msg.sender].length == 0, "pool rule already set");
```

However, it lacks checks to see if the calling address is actually a pool
This allows griefers to spam events by simply calling `setRuleForPool` from any address.

As per the spec given, event emission is indeed used by the protocol for tracking rule changes

```solidity
    // emit event for easier tracking of rule changes
    emit PoolRuleSet(
        address(_poolSettings.rule),
        _poolSettings.oracles,
        _poolSettings.lambda,
        _poolSettings.ruleParameters,
        _poolSettings.epsilonMax,
        _poolSettings.absoluteWeightGuardRail,
        _poolSettings.updateInterval,
        _poolSettings.poolManager
    );
```

Hence, it can affect the offchain mechanism of the protocol

## Impact

1. Incorrect Event will be emitted which is not intended
2. Offchain mechanism griefing can be done by the attackers.

## Proof of Concept

Add the below test case inside the `UpdateWeightRunner.t.sol` file:

```solidity
    function testAnyoneCanSetRule() public {
        uint40 blockTime = uint40(block.timestamp);
        int256[] memory weights = new int256[]();
        weights[0] = 0.5e18;
        weights[1] = 0.5e18;
        weights[2] = 0;
        weights[3] = 0;
        
        int216 fixedValue = 1000;
        uint delay = 3600;
        chainlinkOracle = deployOracle(fixedValue, delay);

        vm.startPrank(owner);
        updateWeightRunner.addOracle(OracleWrapper(chainlinkOracle));
        vm.stopPrank();


        address[][] memory oracles = new address[][]();
        oracles[0] = new address[]();
        oracles[0][0] = address(chainlinkOracle);

        uint64[] memory lambda = new uint64[]();
        lambda[0] = 0.0000000005e18;

        address randomAddress = vm.addr(321);
        vm.startPrank(address(randomAddress));
        
        updateWeightRunner.setRuleForPool(
            IQuantAMMWeightedPool.PoolSettings({
                assets: new IERC20[](0),
                rule: mockRule,
                oracles: oracles,
                updateInterval: 1,
                lambda: lambda,
                epsilonMax: 0.2e18,
                absoluteWeightGuardRail: 0.2e18,
                maxTradeSizeRatio: 0.2e18,
                ruleParameters: new int256[][](),
                poolManager: addr2
            })
        );
        vm.stopPrank();
  
    }
```

## Tools Used

Manual Review
Foundry

## Recommendations

It is recommended to only allow setting rules for pools deployed via Factory contract.

---
### Example 3

**Auto Label:** Unauthorized access to critical pool functions due to missing caller validation and permission checks, enabling state manipulation, malicious rule setting, and griefing of offchain monitoring systems.  

**Original Text Preview:**

**Description:** Certain actions on a pool are protected by permissions. One such permission is `POOL_GET_DATA`, which governs access to the [`UpdateWeightRunner::getData`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L315-L366) function:

```solidity
bool internalCall = msg.sender != address(this);
require(internalCall || approvedPoolActions[_pool] & MASK_POOL_GET_DATA > 0, "Not allowed to get data");
```

The issue lies in how the `internalCall` condition is enforced. The variable `internalCall` is intended to check if the call is made internally by the contract. However, this logic behaves incorrectly:

- `internalCall` will always be `true` because `UpdateWeightRunner` does not call `getData` in a way that changes the call context (i.e., `address(this).getData()`). As a result, `msg.sender` is never equal to `address(this)`.
- Consequently, the statement `msg.sender != address(this)` will always evaluate to `true`, allowing the `require` condition to pass regardless of the pool's permissions.

As a result, the `POOL_GET_DATA` permission is effectively bypassed.

**Impact:** The `UpdateWeightRunner::getData` function can be called on pools that lack the `POOL_GET_DATA` permission.

**Proof of Concept:** Add the following test to `pkg/pool-quantamm/test/foundry/UpdateWeightRunner.t.sol`:
```solidity
function testUpdateWeightRunnerGetDataPermissionCanByBypassed() public {
    vm.prank(owner);
    updateWeightRunner.setApprovedActionsForPool(address(mockPool), 1);

    assertEq(updateWeightRunner.getPoolApprovedActions(address(mockPool)) & 2 /* MASK_POOL_GET_DATA */, 0);

    // this should revert as the pool is not allowed to get data
    updateWeightRunner.getData(address(mockPool));
}
```

**Recommended Mitigation:** Consider removing the `internalCall` boolean entirely. This change will ensure that only calls with the correct `POOL_GET_DATA` permission are allowed. Additionally, this modification allows `getData` to be a `view` function, as the [event emission](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L363-L365) at the end becomes unnecessary.

Here is the recommended code change:
```diff
- function getData(address _pool) public returns (int256[] memory outputData) {
+ function getData(address _pool) public view returns (int256[] memory outputData) {
-     bool internalCall = msg.sender != address(this);
-     require(internalCall || approvedPoolActions[_pool] & MASK_POOL_GET_DATA > 0, "Not allowed to get data");
+     require(approvedPoolActions[_pool] & MASK_POOL_GET_DATA > 0, "Not allowed to get data");
// ...
-     if(!internalCall){
-         emit GetData(msg.sender, _pool);
-     }
```

**QuantAMM:** Fixed in [`a1a333d`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a1a333d28b0aae82338f4ba440af1a473ef1eda6)

**Cyfrin:** Verified. Code refactored, `internalCall` now passed as a parameter to a new `UpdateWeightRunner::_getData`. The original `UpdateWeightRunner::getData` requires permission but the internal call during update does not.

---
