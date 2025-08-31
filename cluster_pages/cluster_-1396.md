# Cluster -1396

**Rank:** #342  
**Count:** 19  

## Label
Misuse of data or state references leading to incorrect calculations, flawed validations, or erroneous state transitions, resulting in financial misrepresentation, over-distribution, or transaction failures.

## Cluster Information
- **Total Findings:** 19

## Examples

### Example 1

**Auto Label:** Mismanagement of claimable amounts across periods due to flawed state tracking, improper aggregation, or incorrect order of operations, leading to lost funds, over-claims, or incorrect entitlements.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The comment for `claimAll()` states that the return value is the `Total amount of interest claimed`, but from the code, it can be seen that if the owners of these tokens are not the same, the return value is only the amount claimed by the last owner, not the total amount claimed throughout the process:

```solidity
/// @return uint256 Total amount of interest claimed
function claimAll(uint256[] memory tokenIds) public whenNotPaused returns (uint256) {
    // code
    for (uint256 i; i < tokenIdsLength;) {
        uint256 tokenId = tokenIds.unsafeMemoryAccess(i);
        ...
        address owner = ownerOf(tokenId);
        if (owner != lastOwner) {
            totalClaimed += claimableInterestCurrentOwner;
            payableToken.safeTransfer(lastOwner, claimableInterestCurrentOwner);
            claimableInterestCurrentOwner = 0; //@audit reset when owner change
        }
        lastOwner = owner;
        uint256 claimableInterest =
            Math.min(payableToken.balanceOf(address(this)), accruedInterest - claimed[tokenId]);
        claimed[tokenId] += claimableInterest;
        claimableInterestCurrentOwner += claimableInterest;
        emit Claimed(owner, tokenId, claimableInterest);
        unchecked {
            ++i;
        }
    }
    totalClaimed += claimableInterestCurrentOwner;
    payableToken.safeTransfer(lastOwner, claimableInterestCurrentOwner);

    return claimableInterestCurrentOwner;
}
```

## Location of Affected Code

File: [LendingNFT.sol#L287-L324](https://github.com/tokyoweb3/HARVESTFLOW_Ver.2/blob/05f0814caa51dcb034dd01f610315d4c6efedce8/contracts/src/LendingNFT.sol#L287-L324)

## Impact

The `claimAll()` does not return the total amount of interest claimed as the comment states, as well as `redeemAll()` which uses `claimAll()`'s return.

## Recommendation

Use the increment of `totalClaimed` as the return value.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Mismanagement of claimable amounts across periods due to flawed state tracking, improper aggregation, or incorrect order of operations, leading to lost funds, over-claims, or incorrect entitlements.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

The FundingVault's `disburseClaims` function incorrectly assigns all current contract token balances to the current period, without accounting for unclaimed rewards from previous periods. This allows funds allocated to previous periods to be claimed by users in the current period.

We can see that the FundingVault enables users to make claims for the previous period. The claimable amount is determined by the lesser value between `_claimable` and the current balance of the claimToken.

```solidity
    function claim(uint256 _period) external override virtual lock {
        ...
        _claimable = GSMath.min(GammaSwapLibrary.balanceOf(claimToken, address(this)), _claimable);
        ...
        GammaSwapLibrary.safeTransfer(claimToken, _to, _claimable);
```

However, `disburseClaims` fails to track unclaimed amounts from previous periods. And it assigns the entire token balance to the current period without reservation for pending claims.

```solidity
    function disburseClaims() external override virtual isManager {
        ...
        totalClaimable[_period] = GammaSwapLibrary.balanceOf(claimToken, address(this)); // @audit Incorrectly assigns all current balance to current period
        ...
    }
```

Scenario:

- In period 2, Alice is eligible to claim 1000 tokens but hasn't claimed yet
- Period 3 begins, and new claim amount worth 500 tokens are added
- `disburseClaims` is called, setting totalClaimable[3] = 1500 (1000 + 500)
- Bob, who is eligible for only 100 tokens in period 3, claims first and receives more than his fair share from the total 1500 tokens
- When Alice tries to claim her 1000 tokens from period 2, insufficient funds remain. She may receive 0 if all the tokens are claimed in period 3.

## Recommendations

Tracking separately `totalClaimable` in each period by parameters in `disburseClaims`:

```diff
- function disburseClaims() external override virtual isManager {
+ function disburseClaims(uint256 newClaimable) external override virtual isManager {
    ...
-    totalClaimable[_period] = GammaSwapLibrary.balanceOf(claimToken, address(this));
+    totalClaimable[_period] = newClaimable;
    ...
}
```

---
### Example 3

**Auto Label:** Mismanagement of claimable amounts across periods due to flawed state tracking, improper aggregation, or incorrect order of operations, leading to lost funds, over-claims, or incorrect entitlements.  

**Original Text Preview:**

The `totalClaimAmount` value in the `Claim` contract is supposed to keep internal accounting information. However, it may be set to a non-zero value when on initialization as there is no check to prevent the admin from doing so.

This will lead to improper accounting and can lead to reverts or undesired effects in the functions using it.

Consider checking `_claimSettings.claimAmountDetails.totalClaimAmount == 0` when initializing the contract.

---
