# Cluster -1422

**Rank:** #94  
**Count:** 175  

## Label
Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.

## Cluster Information
- **Total Findings:** 175

## Examples

### Example 1

**Auto Label:** Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The function `rebase()` handles the share price increase to apply the desired APR. The issue is that the code doesn't use `lastRebaseTime` when calculating rate increase and it assumes 24 hours have passed from the last time which couldn't not be the case. This would wrong APR for some of the time. For example if `rebase()` is called 1 hour sooner or later then for those 1 hour the share price rate would be wrong.

## Recommendations

Use `duration = block.timestamp - lastRebaseTime` to calculate the real rate increase that accrued from the last rebase call.

---
### Example 2

**Auto Label:** Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `PositionManager.setInterestRateStrategy()` function enables the protocol owner to update the interest rate strategy contract. However, this operation does not include any mechanism to migrate or initialize the internal state required by the new strategy. Parameters such as `targetUtilization`, `endRateAt`, and `lastUpdate` for each vault remain uninitialized in the new strategy, which leads to inaccurate and inflated interest rate calculations.

```solidity
function setInterestRateStrategy(address _newInterestRateStrategy) external onlyOwner {
    if (_newInterestRateStrategy == address(0)) {
        revert ZeroAddress();
    }

    emit NewInterestRateStrategy(address(interestRateStrategy), address(_newInterestRateStrategy));

    interestRateStrategy = IInterestRateStrategy(_newInterestRateStrategy);
}
```

The interest rate calculation function, `interestRate()`, relies on vault-specific parameters to compute a smooth and adaptive rate curve based on the difference between current utilization and the target utilization. If these parameters are missing or zero-initialized, the function calculates a large error (`err`), resulting in excessive linear adaptation and an artificially high interest rate.

However, this function is strictly controlled by the owner and is expected to be used only in exceptional circumstances.

## Recommendation

The new `interestRateStrategy` contract should either be initialized with the current state of each vault or properly migrate relevant parameters from the previous strategy to ensure accuracy and consistency in interest rate calculations.

---
### Example 3

**Auto Label:** Failure to account for accrued interest in rate calculations and state updates leads to mispriced rates, retroactive rate application, and inaccurate utilization, enabling financial exploitation and protocol instability.  

**Original Text Preview:**

The `registerVault` operation takes in two MCR values—one explicitly passed as a variable and recorded in the global `VaultData`, while the other is encoded in `_registerData` and used to calculate the initial `interestRate`. These two MCR values must be the same; otherwise, the initial `interestRate` calculation may be incorrect.

```diff
    function registerVault(address _vaultAddress, uint256 _mcr, bytes memory _registerData)
        external
        onlyOwner
        returns (uint8)
    {
        uint8 index = lastVaultIndex;
+       (uint256 mcr, ) = abi.decode(_registerData, (uint256, uint256));
+       require(mcr == _mcr, "mcr misMatch");
```

---
