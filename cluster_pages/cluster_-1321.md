# Cluster -1321

**Rank:** #219  
**Count:** 49  

## Label
Lack of proper access control and validation leads to unauthorized asset manipulation, fund locking, and incorrect liquidity assessments, enabling users to bypass safety checks and exploit system invariants.

## Cluster Information
- **Total Findings:** 49

## Examples

### Example 1

**Auto Label:** Missing interaction layers cause critical access control flaws, rendering owner-controlled functions effectively unusable and leading to loss of operational flexibility and fund access.  

**Original Text Preview:**

## Severity: Low Risk

## Context
(No context files were provided by the reviewers)

## Description
The `BaseFeeCalculator` contract's `setVaultFees` function currently checks that the caller is the owner of the vault by verifying:

```solidity
if (msg.sender != ISingleDepositorVault(vault).owner()) revert FeeCalculator__OnlyOwner();
```

This check assumes that only the owner of a vault should be authorized to modify its fee parameters. However, vaults built on `BaseVault` inherit from `Auth2Step`, not from a traditional `Ownable` contract. In `Auth2Step`, ownership can be renounced (setting the owner to `address(0)`), while an auth address can still retain administrative authority. 

After ownership renouncement, this check would permanently prevent a vault from updating its fees, even if an authorized administrator (auth) is still active and should be allowed to perform administrative operations. While this situation is rare and generally discouraged, it represents a mismatch between the vault's real authority model (owner OR auth) and the enforced access control in the fee configuration.

## Recommendation
Consider updating the access control logic in `BaseFeeCalculator.setVaultFees` to allow either the vault owner or the vault auth to call the function, aligning it with the vault's internal authority model. Alternatively, document this limitation explicitly if supporting fee updates post-ownership renouncement is not a design goal. 

Additionally, it could be beneficial to note that vaults inherit from `Auth2Step` rather than `Ownable`, which clarifies the expected ownership semantics.

## Aera
This is intentional. The `Auth` contract is responsible for determining which addresses can call which functions on a given contract. They do not ascribe any semantics to what "roles" these addresses have. For that reason, these permissions are not exposed to external users.

## Spearbit
Acknowledged by Aera team as indicated. This is an intentional design decision.

---
### Example 2

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
### Example 3

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
