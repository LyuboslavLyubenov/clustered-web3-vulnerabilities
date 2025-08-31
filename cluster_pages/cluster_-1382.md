# Cluster -1382

**Rank:** #271  
**Count:** 31  

## Label
Self-liquidation flaws enable borrowers or creditors to profitably liquidate undercollateralized positions by misrepresenting debt exposure or bypassing fee and incentive controls, leading to unfair collateral distribution and compromised economic fairness.

## Cluster Information
- **Total Findings:** 31

## Examples

### Example 1

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
### Example 2

**Auto Label:** Missing safety buffers and validation checks in liquidation logic lead to unjustified, cascading, or exploitable liquidations, exposing users and lenders to financial loss and systemic risk.  

**Original Text Preview:**

##### Description
`LendingPool.borrow()` has a limit up to `TypeofLTV.MaximumLTV` (set at `80%` in the project tests). However, `LendingPool.withdraw()` allows withdrawals up to `TypeofLTV.LiquidationThreshold` (set at `90%` in the tests).

An attacker could call `borrow() + withdraw()` in a single transaction to effectively borrow up to `TypeofLTV.LiquidationThreshold`, bypassing the `MaximumLTV` limit.

##### Recommendation
We recommend using the same LTV threshold for both `borrow()` and `withdraw()`.

***

---
### Example 3

**Auto Label:** Missing safety buffers and validation checks in liquidation logic lead to unjustified, cascading, or exploitable liquidations, exposing users and lenders to financial loss and systemic risk.  

**Original Text Preview:**

**Description:** When borrowing from Aave, there is a check in [`Aave_Module::aave_borrow`](https://github.com/d2sd2s/d2-contracts/blob/c2fc257605ebc725525028a5c17f30c74202010b/contracts/modules/Aave.sol#L51-L54) to ensure that the loan-to-value (LTV) ratio remains below `MAX_LTV_FACTOR` (80%):

```solidity
pool.borrow(asset, amount, interestRateMode, referralCode, onBehalfOf);
(uint256 totalCollateralBase, uint256 totalDebtBase, , , uint256 ltv, ) = pool.getUserAccountData(onBehalfOf);
uint256 maxDebtBase = (totalCollateralBase * ltv * MAX_LTV_FACTOR) / (BASIS_FACTOR * MANTISSA_FACTOR);
require(totalDebtBase <= maxDebtBase, "borrow amount exceeds max LTV");
```

However, this LTV check is missing from [`Silo_Module::silo_borrow`](https://github.com/d2sd2s/d2-contracts/blob/c2fc257605ebc725525028a5c17f30c74202010b/contracts/modules/Silo.sol#L44-L47) and [`Dolomite_Module::dolomite_openBorrowPosition`](https://github.com/d2sd2s/d2-contracts/blob/c2fc257605ebc725525028a5c17f30c74202010b/contracts/modules/Dolomite.sol#L208-L216)/[`dolomite_transferBetweenAccounts`](https://github.com/d2sd2s/d2-contracts/blob/c2fc257605ebc725525028a5c17f30c74202010b/contracts/modules/Dolomite.sol#L226-L234), which are also lending protocols. As a result, positions with a higher LTV than 80% can be opened in these platforms.

**Impact:** Since Silo and Dolomite do not enforce the same 80% LTV limit higher risk positions can be entered. This increases the likelihood of unexpected liquidations, exposing users to unnecessary financial risk.

**Recommended Mitigation:** Consider implementing the same LTV checks for Silo and Dolomite.

**D2:** Ignored "Silo and Dolomite lack LTV limit increasing liquidation risk", will assume the trader is not reckless

**Cyfrin:** Acknowledged.

---
