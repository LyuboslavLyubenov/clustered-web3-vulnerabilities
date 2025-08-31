# Cluster -1419

**Rank:** #348  
**Count:** 18  

## Label
Underflow errors enable attackers to front-run debt repayments, causing transaction failures and allowing malicious users to prevent full debt settlement and manipulate on-chain state.

## Cluster Information
- **Total Findings:** 18

## Examples

### Example 1

**Auto Label:** **Arithmetic errors in repayment logic lead to underflow, overflows, or invalid state transitions, enabling debt manipulation, unauthorized state changes, or transaction failure.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-11-teller-finance-update-judging/issues/46 

## Found by 
OpaBatyo, hash
### Summary

`LenderCommitmentGroup_Smart` doesn't handle excess repayments making it possible to brick the lending functionality 

### Root Cause

The [`getTotalPrincipalTokensOutstandingInActiveLoans` function](https://github.com/sherlock-audit/2024-11-teller-finance-update/blob/0c8535728f97d37a4052d2a25909d28db886a422/teller-protocol-v2-audit-2024/packages/contracts/contracts/LenderCommitmentForwarder/extensions/LenderCommitmentGroup/LenderCommitmentGroup_Smart.sol#L965-L972) performs `totalPrincipalTokensLended - totalPrincipalTokensRepaid` without handling the underflow scenario. This is problematic as excess amount can be repaid for loans which will cause an underflow here

```solidity
    function getTotalPrincipalTokensOutstandingInActiveLoans()
        public
        view
        returns (uint256)
    {
        return totalPrincipalTokensLended - totalPrincipalTokensRepaid;
    }
```

on excess repayments, `totalPrincipalTokensRepaid` will become greater than `totalPrincipalTokensLended`
```solidity
    function repayLoanCallback(
        uint256 _bidId,
        address repayer,
        uint256 principalAmount,
        uint256 interestAmount
    ) external onlyTellerV2 whenForwarderNotPaused whenNotPaused bidIsActiveForGroup(_bidId) { 
        totalPrincipalTokensRepaid += principalAmount;
        totalInterestCollected += interestAmount;
```

function in TellerV2.sol contract to repay excess amount. User can specify any amount greater than minimum amount
```solidity
    function repayLoan(uint256 _bidId, uint256 _amount)
        external
        acceptedLoan(_bidId, "rl")
    {
        _repayLoanAtleastMinimum(_bidId, _amount, true);
    }
```

The function `getTotalPrincipalTokensOutstandingInActiveLoans` is invoked before every lending. Hence an underflow here will cause the lending functionality to revert

Another quirk regarding excess repayments is that the lenders of the pool won't obtain the excess repaid amount since it is not accounted anywhere. But this cannot be considered an issue since the lenders are only guarenteed the original lending amount + interest

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

1. Attacker borrows 100 from the lender pool
totalPrincipalTokensLended == 100
totalPrincipalTokensRepaid == 0

2. Attacker repays 101
totalPrincipalTokensLended == 100
totalPrincipalTokensRepaid == 101

Now `getTotalPrincipalTokensOutstandingInActiveLoans` will always revert

### Impact

Bricked lending functionality

### PoC

_No response_

### Mitigation

In case repaid principal is more, return 0 instead

---
### Example 2

**Auto Label:** **Arithmetic errors in repayment logic lead to underflow, overflows, or invalid state transitions, enabling debt manipulation, unauthorized state changes, or transaction failure.**  

**Original Text Preview:**

**Description**

the 'repay' function increases a vault_sol_balance and decreases a borrowed_amount without receiving any payments
File: lending/programs/water/src/lib.rs: 614

**Recommendation**

add a Transfer function to get the payment from the borrower

**Re-audit comment**

Resolved

---
### Example 3

**Auto Label:** **Arithmetic errors in repayment logic lead to underflow, overflows, or invalid state transitions, enabling debt manipulation, unauthorized state changes, or transaction failure.**  

**Original Text Preview:**

In the `repay` method the `MarginAccount` contract, the `amount` is checked before it is potentially modified again a few lines below. In the case of initially `amount == 0`, the transaction could revert due to an arithmetic underflow error instead of failing with an error message.  
Consider moving the `require` check below the potential amount change.

```solidity
function repay(uint marginAccountID, address token, uint amount) external onlyRole(MARGIN_TRADING_ROLE) {
    address liquifityPoolAddress = tokenToLiquidityPool[token];
    require(liquifityPoolAddress != address(0), "Token is not supported");

    require(amount <= erc20ByContract[marginAccountID][token], "Insufficient funds to repay the debt");  // @audit move this check

    uint debtWithAccruedInterest = ILiquidityPool(liquifityPoolAddress).getDebtWithAccruedInterest(marginAccountID);
    if (amount == 0 || amount > debtWithAccruedInterest) {
        amount = debtWithAccruedInterest;  // @audit amount is modified here again
    }

    // @audit move here

    erc20ByContract[marginAccountID][token] -= amount; // @audit potential underflow error
    ILiquidityPool(liquifityPoolAddress).repay(marginAccountID, amount);
}
```

---
