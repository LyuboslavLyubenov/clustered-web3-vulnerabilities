# Cluster -1030

**Rank:** #2  
**Count:** 1227  

## Label
Failure to validate account ownership or token identity leads to unauthorized access, mint manipulation, or invalid state transitions, enabling attackers to bypass authorization and exploit token or account mismatches.

## Cluster Information
- **Total Findings:** 1227

## Examples

### Example 1

**Auto Label:** Failure to properly validate or handle token transfer outcomes, leading to unauthorized transfers, reentrancy, or system lockouts due to missing return value checks or inadequate error handling.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The contract performs an ERC20 token `transferFrom()` without verifying its success. While most major tokens revert on failure, some non-ERC20 standard tokens technically allow them to return false instead of reverting.

This introduces a risk where Non-Compliant Tokens may fail silently, while the protocol assumes success, potentially leading to missing or undelivered funds.

## Location of Affected Code

File: [src/CredifiERC20Adaptor.sol#L620](https://github.com/credifi/contracts-audit/blob/ba976bad4afaf2dc068ca9dcd78b38052d3686e3/src/CredifiERC20Adaptor.sol#L620)

```solidity
function _performLoanRepayment( address user, IndividualLoan storage loan, uint256 repayAmount, address payer, bool requireFullRepayment ) internal returns (uint256 actualRepayAmount, uint256 remainingDebt, bool fullyRepaid) {
   // code

   // Transfer repay tokens from payer to this contract
   borrowToken.transferFrom(payer, address(this), actualRepayAmount);
   // code
}
```

## Recommendation

Consider using `safeTransferFrom()` instead of `transferFrom()`:

```solidity
// Use OpenZeppelin's SafeERC20
using SafeERC20 for IERC20;

// Replace transfer with:
borrowToken.safeTransferFrom(payer, address(this), actualRepayAmount);
```

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Failure to properly validate or handle token transfer outcomes, leading to unauthorized transfers, reentrancy, or system lockouts due to missing return value checks or inadequate error handling.  

**Original Text Preview:**

Currently, the `returnExcessiveBond` function is available inside the `WorkerRegistration`. In the event that the bond is ever reduced from the system, the excess amount can be claimed and returned to the creator. However, if the bond ever increases, there is no direct way for workers to update the bond, it has to go through the whole deregister and register process, which requires waiting for a lock period. This will add a time delay for the actual bond inside the system to match the TVL calculation. Consider to add function to update bond in case of the bond ever increased, similar to `returnExcessiveBond`.

---
### Example 3

**Auto Label:** Inconsistent state validation and incomplete access controls enable attackers to manipulate token balances, bypass safeguards, or extract funds—leading to fund loss, supply overruns, or systemic risk.  

**Original Text Preview:**

The `cap` variable is intended to limit the total supply of `rsTAO` and `rsCOMAI`, and is checked in the `wrap()` function to ensure that the total minted does not exceed this cap:

```solidity
File: RivusTAO.sol
887:   function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
...
...
894:     require(
895:       cap >= totalRsTAOMinted,
896:       "Deposit amount exceeds maximum"
897:     );
...
...
```

However, the current implementation allows the `cap` to be exceeded under certain conditions. Consider the following scenario:

1. The `cap` is set to `10` and `totalRsTAOMinted` is `9`.
2. A user performs a `wrap()` operation that would mint `30 rsTAO`, resulting in `totalRsTAOMinted` becoming `39`, thereby exceeding the `cap` of `10`.

This can happen because the `cap` check is performed before the new `rsTAO` is minted and added to the `totalRsTAOMinted`.

It is advisable to adjust the validation logic to include the amount of `rsTAO` to be minted in the cap check. This can be done by moving the cap check to after the calculation of `rsTAO` to be minted. This ensures the cap is not exceeded after new tokens are minted:

```diff
  function wrap(uint256 wtaoAmount) public nonReentrant checkPaused returns (uint256) {
    ...
    ...
--  require(
--    cap >= totalRsTAOMinted,
--    "Deposit amount exceeds maximum"
--  );
    ...
    ...
    uint256 rsTAOAmount = getRsTAObyWTAO(wrapAmountAfterFee);

    // Perform token transfers
    _mintRsTAO(msg.sender, rsTAOAmount);
++  require(
++    cap >= totalRsTAOMinted,
++    "Deposit amount exceeds maximum"
++  );
    ...
    ...
  }
```

---
