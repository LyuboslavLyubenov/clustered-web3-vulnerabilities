# Cluster -1451

**Rank:** #263  
**Count:** 33  

## Label
Failure to update internal token balances after transfers leads to stale state data, causing incorrect reserve calculations, over-issuance, and potential double-dip attacks that compromise economic stability and protocol accuracy.

## Cluster Information
- **Total Findings:** 33

## Examples

### Example 1

**Auto Label:** Failure to validate contract token balances before operations, leading to potential fund loss, incorrect state transitions, or unauthorized transactions due to insufficient or unverified balance checks.  

**Original Text Preview:**

Code uses `_balances[address(this)].classA` to get the contract's token balance during the migration and withdrawal of excess funds. The issue is that the contract address may have `classB` balance too and those funds won't be used in migration and the deployer can't withdraw them too. It's possible to transfer `classB` balance to the contract address or buy tokens for the contract address which would increase the `classB` balance of the contract address. It would be better to use `balanceOf(This)` to get the contract's balance to make sure there would be no leftovers.

---
### Example 2

**Auto Label:** Failure to validate contract token balances before operations, leading to potential fund loss, incorrect state transitions, or unauthorized transactions due to insufficient or unverified balance checks.  

**Original Text Preview:**

**Severity** : Low

**Status**: Acknowledged

**Description**: 

The claim() function does not validate whether the contract has enough HEU tokens to fulfill a user's claim. If the balance of HEU is insufficient (e.g., due to previous withdrawals or unexpected transfers), the claim could fail.
 Users trying to claim their vested HEU may experience transaction reverts
 
**Recommendation**: 

Add a validation to ensure that the contract holds enough HEU tokens to fulfill the claim:
```solidity
require(heu.balanceOf(address(this)) >= heuAmount, "Insufficient HEU balance in contract to fulfill claim");
```

---
### Example 3

**Auto Label:** Failure to verify actual token balances post-transfer, leading to incorrect state updates, financial loss, or transaction failures due to unaccounted fees or dynamic token behavior.  

**Original Text Preview:**

##### Description
Some parts of the code operate under the assumption that the `safeTransfer()` and `safeTransferFrom()` functions will transfer an exact amount of tokens.

However, the actual amount of tokens received may be less for **fee-on-transfer** tokens. Moreover, the protocol doesn't accommodate **rebasing** tokens whose balances fluctuate over time.

These discrepancies can lead to transaction reverts or other unexpected behaviour.

##### Recommendation
We recommend implementing verification checks both before and after token transfers.

---
