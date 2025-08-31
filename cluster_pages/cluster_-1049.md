# Cluster -1049

**Rank:** #331  
**Count:** 20  

## Label
Inconsistent state cleanup and persistence mechanisms lead to stale, inaccurate, or lost data, enabling state inconsistencies, front-running, and unintended behavior due to improper storage management across operations.

## Cluster Information
- **Total Findings:** 20

## Examples

### Example 1

**Auto Label:** Inconsistent or incomplete storage deletion leads to stale state persistence, causing incorrect data exposure, false usage limits, and unauthorized access due to improper state synchronization.  

**Original Text Preview:**

## Severity

Low Risk

## Description

When loans are closed by calling `closeIndividualLoan()`, the system fails to fully clear all associated data. While it removes the token, related records, it neglects to delete the core loan details stored in the `userLoans[user][loanId]` mapping. This permanently occupies storage space.

The `_removeTokenFromUser()` function fails to fully clear all loan-related storage, specifically neglecting to delete the loan data from `userLoans[user][loanId]`.

## Location of Affected Code

File: [src/CredifiERC20Adaptor.sol#L298](https://github.com/credifi/contracts-audit/blob/ba976bad4afaf2dc068ca9dcd78b38052d3686e3/src/CredifiERC20Adaptor.sol#L298)

```solidity
function _removeTokenFromUser(address user, uint256 tokenId) internal {
   _removeFromArray(userDepositedTokens[user], userTokenIndex[user], tokenId);

   delete tokenDepositor[tokenId];
   delete mintedCreditForToken[tokenId];
   delete tokenToLoanId[tokenId];
}
```

## Recommendation

Consider deleting all of the loan-related storage variables, specifically the `userLoans` mapping:

```diff
function _removeTokenFromUser(address user, uint256 tokenId) internal {
   _removeFromArray(userDepositedTokens[user], userTokenIndex[user], tokenId);

   delete tokenDepositor[tokenId];
   delete mintedCreditForToken[tokenId];
   delete tokenToLoanId[tokenId];
+  delete userLoans[user][loanId];
}
```

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Inconsistent state cleanup and persistence mechanisms lead to stale, inaccurate, or lost data, enabling state inconsistencies, front-running, and unintended behavior due to improper storage management across operations.  

**Original Text Preview:**

Whenever operations modifying the data stored in the `instance` storage of the token contract such as minting or burning tokens are performed, the entire `instance` TTL [is increased](https://github.com/OpenZeppelin/stellar-contracts/blob/01dbcb53d3992570dba5faf517a3d0502ebf1254/contracts/token/fungible/src/storage.rs#L47). However, other operations performed on the token contract, such as [querying balance](https://github.com/OpenZeppelin/stellar-contracts/blob/01dbcb53d3992570dba5faf517a3d0502ebf1254/contracts/token/fungible/src/storage.rs#L58-L66) or [transferring tokens](https://github.com/OpenZeppelin/stellar-contracts/blob/01dbcb53d3992570dba5faf517a3d0502ebf1254/contracts/token/fungible/src/storage.rs#L361-L382) between accounts, do not increase the `instance` TTL. As a result, it is possible that the token contract is used, but its data eventually becomes archived. As a result, it will have to be manually restored, which incurs an additional cost.

Consider increasing the contract's `instance` storage TTL on all operations querying or modifying the state of the contract. Alternatively, consider documenting the current design choice so that it is clear for the implementers.

***Update:** Resolved in [pull request #62](https://github.com/OpenZeppelin/stellar-contracts/pull/62) at commit [4500bfb](https://github.com/OpenZeppelin/stellar-contracts/pull/62/commits/4500bfbc498db3843e0d6317c5c70a80637ac461).*

---
### Example 3

**Auto Label:** Inconsistent state cleanup and persistence mechanisms lead to stale, inaccurate, or lost data, enabling state inconsistencies, front-running, and unintended behavior due to improper storage management across operations.  

**Original Text Preview:**

Whenever operations modifying the data stored in the `instance` storage of the token contract such as minting or burning tokens are performed, the entire `instance` TTL [is increased](https://github.com/OpenZeppelin/stellar-contracts/blob/01dbcb53d3992570dba5faf517a3d0502ebf1254/contracts/token/fungible/src/storage.rs#L47). However, other operations performed on the token contract, such as [querying balance](https://github.com/OpenZeppelin/stellar-contracts/blob/01dbcb53d3992570dba5faf517a3d0502ebf1254/contracts/token/fungible/src/storage.rs#L58-L66) or [transferring tokens](https://github.com/OpenZeppelin/stellar-contracts/blob/01dbcb53d3992570dba5faf517a3d0502ebf1254/contracts/token/fungible/src/storage.rs#L361-L382) between accounts, do not increase the `instance` TTL. As a result, it is possible that the token contract is used, but its data eventually becomes archived. As a result, it will have to be manually restored, which incurs an additional cost.

Consider increasing the contract's `instance` storage TTL on all operations querying or modifying the state of the contract. Alternatively, consider documenting the current design choice so that it is clear for the implementers.

***Update:** Resolved in [pull request #62](https://github.com/OpenZeppelin/stellar-contracts/pull/62) at commit [4500bfb](https://github.com/OpenZeppelin/stellar-contracts/pull/62/commits/4500bfbc498db3843e0d6317c5c70a80637ac461).*

---
