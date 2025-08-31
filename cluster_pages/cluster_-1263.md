# Cluster -1263

**Rank:** #254  
**Count:** 36  

## Label
Race conditions enabling front-running and state-dependent exploits, allowing attackers to manipulate share allocations, trigger reverts, or withdraw funds prematurely through transaction ordering vulnerabilities.

## Cluster Information
- **Total Findings:** 36

## Examples

### Example 1

**Auto Label:** Lack of proper access control and validation leads to unauthorized asset manipulation, fund locking, and incorrect liquidity assessments, enabling users to bypass safety checks and exploit system invariants.  

**Original Text Preview:**

In the `setLiquidationManager()` function of the `PositionManager` contract, when changing the `liquidationManager` address, the approval of the previous `liquidationManager` address **is not revoked from all registered vaults**. This oversight could result in compromising the `PositionManager` contract shares if the previous `liquidationManager` contract is compromised, as it would still retain approval to manage the assets in the vaults:

```solidity
 function setLiquidationManager(address _newLiquidationManager) external onlyOwner {
        emit NewLiquidationManager(liquidationManager, _newLiquidationManager);

        liquidationManager = _newLiquidationManager;
    }
```

Recommendation: implement a mechanism to revoke vaults approval from the old `liquidationManager` address.

---
### Example 2

**Auto Label:** Lack of proper access control and validation leads to unauthorized asset manipulation, fund locking, and incorrect liquidity assessments, enabling users to bypass safety checks and exploit system invariants.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

In the `setLiquidationManager()` function of the `PositionManager` contract, when the `liquidationManager` address is changed, the new contract is not granted approval on the share tokens for all registered vaults. This results in the **failure of liquidation for these vaults** when the `liquidationManager` attempts to redeem assets to pay for the liquidator:

```solidity
 function setLiquidationManager(address _newLiquidationManager) external onlyOwner {
        emit NewLiquidationManager(liquidationManager, _newLiquidationManager);

        liquidationManager = _newLiquidationManager;
    }
```

## Recommendations

Implement a mechanism to grant the new `liquidationManager` address approval on all registered vaults shares.

---
### Example 3

**Auto Label:** Lack of proper access control and validation leads to unauthorized asset manipulation, fund locking, and incorrect liquidity assessments, enabling users to bypass safety checks and exploit system invariants.  

**Original Text Preview:**

## Severity

High Risk

## Description

`PositionManager` can receive liquidity position NFTs, collect fees, and add liquidity to them but there is no way for the admin to transfer these positions outside the contract, access the liquidity they are holding, or burn them.

## Location of Affected Code

File: [PositionManager.sol]()

## Impact

As a result, the underlying liquidity will not be accessible anymore resulting in a loss of funds. Furthermore, these LP positions will be configured for a specific price range. Once the price goes outside of this price range it is normal for liquidity providers to be able to move the liquidity to another price range that is more profitable by decreasing the liquidity from their current LP position and to create a new position with a more relevant price range.

So over some time when the price moves the admin will not be able to collect fees efficiently as well.

## Recommendation

Consider implementing logic for either transferring the LP position out of the contract or logic that will be able to access the underlying liquidity of the LP positions that are in control of the `PositionManager`.

## Team Response

Acknowledged.

---
