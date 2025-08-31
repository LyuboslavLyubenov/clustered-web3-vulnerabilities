# Cluster -1379

**Rank:** #222  
**Count:** 48  

## Label
Lack of proper access control and validation leads to unauthorized asset manipulation, fund locking, and incorrect liquidity assessments, enabling users to bypass safety checks and exploit system invariants.

## Cluster Information
- **Total Findings:** 48

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

**Auto Label:** Missing safety buffers and validation checks in liquidation logic lead to unjustified, cascading, or exploitable liquidations, exposing users and lenders to financial loss and systemic risk.  

**Original Text Preview:**

**Description:** According to the [EulerSwap whitepaper](https://github.com/euler-xyz/euler-swap/blob/1022c0bb3c034d905005f4c5aee0932a66adf4f8/docs/whitepaper/EulerSwap_White_Paper.pdf):

> The space of possible reserves is determined by how much real liquidity an LP has and how much debt their operator is allowed to hold. Since EulerSwap AMMs do not always hold the assets used to service swaps at all times, they perform calculations based on virtual reserves and debt limits, rather than on strictly real reserves. Each EulerSwap LP can configure independent virtual reserve levels. These reserves define the maximum debt exposure an AMM will take on. Note that the effective LTV must always remain below the borrowing LTV of the lending vault to prevent liquidation.

This implies that, under proper configuration, an AMM should not be at risk of liquidation due to excessive loan-to-value (LTV). However, external factors can still undermine this guarantee.

For example, if another position in the same collateral vault is liquidated and leaves behind bad debt, the value of the shared collateral used for liquidity could drop. This would affect the effective LTV of all positions using that vault, including those managed by the EulerSwap AMM.

An attacker could exploit this situation by initiating a swap that pushes the AMM's position right up to the liquidation threshold, leveraging the degraded collateral value caused by unrelated bad debt.

**Recommended Mitigation:** Consider enforcing an explicit LTV check after each borrowing operation, and introduce a configurable maximum LTV parameter in `EulerSwapParams`. Additionally, clarify in the documentation that Euler account owners are responsible for monitoring the health of their vaults and should take proactive steps if the collateral accrues bad debt or drops in value—since this can happen independently of swap activity.

**Euler:** Acknowledged. Doing a health computation at the end of a swap would cost too much gas.

**Cyfrin:** The Euler team added several points in their documentation regarding liquidation concerns in [PR#93](https://github.com/euler-xyz/euler-swap/pull/93)

\clearpage

---
