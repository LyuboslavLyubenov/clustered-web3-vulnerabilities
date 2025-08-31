# Cluster -1318

**Rank:** #44  
**Count:** 411  

## Label
Failure to validate sender or recipient when handling native tokens leads to unauthorized fund transfers, misappropriation, or improper token routing, enabling exploitation and operational failures.

## Cluster Information
- **Total Findings:** 411

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

**Auto Label:** Failure to properly validate or handle token transfer outcomes, leading to unauthorized transfers, reentrancy, or system lockouts due to missing return value checks or inadequate error handling.  

**Original Text Preview:**

The `safePullERC20()` function is designed to allow the withdrawal of tokens from the contract by authorized users:

```solidity
File: RivusTAO.sol
941:
942:   function safePullERC20(
943:     address tokenAddress,
944:     address to,
945:     uint256 amount
946:   ) public hasTokenSafePullRole checkPaused {
947:     _requireNonZeroAddress(to, "Recipient address cannot be null address");
948:
949:     require(amount > 0, "Amount must be greater than 0");
950:
951:     IERC20 token = IERC20(tokenAddress);
952:     uint256 balance = token.balanceOf(address(this));
953:     require(balance >= amount, "Not enough tokens in contract");
954:
955:     // "to" have been checked to be a non zero address
956:     bool success = token.transfer(to, amount);
957:     require(success, "Token transfer failed");
958:     emit ERC20TokenPulled(tokenAddress, to, amount);
959:   }
```

However, this function also allows the withdrawal of `wrappedTokens` (e.g., `wTAO` or `wCOMAI`), which should remain in the contract as they are assets deposited via the `approveMultipleUnstakes()` function and are intended to be available for users when they execute `unstake()`. Allowing the `safePullERC20` function to withdraw `wrappedTokens` can lead to misuse of the protocol's assets, as these tokens are essential for fulfilling user `unstake` requests.

To prevent unauthorized withdrawal of essential assets, the `safePullERC20` function should be restricted to exclude `wrappedTokens`. This ensures that the tokens meant for user `unstake` requests remain available. Alternatively, ensure that the approved amounts for `unstake` are not eligible for withdrawal through the `safePullERC20` function, thus maintaining the integrity of the assets needed for user operations.

---
