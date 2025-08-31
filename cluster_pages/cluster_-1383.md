# Cluster -1383

**Rank:** #208  
**Count:** 53  

## Label
Failure to claim rewards before token burning leads to permanent loss of user-entitled yields due to improper state transitions and ownership invalidation.

## Cluster Information
- **Total Findings:** 53

## Examples

### Example 1

**Auto Label:** Failure to validate state or inputs before reward or token changes leads to incorrect payouts, asset loss, or economic distortion.  

**Original Text Preview:**

The `Staking.transferPayout()` function allows the **owner** to distribute rewards to **stakers**. However, this function deducts the rewards from the `lockedLiquidity` **that has been locked by game contracts**.
This results in inconsistencies when the `lockedLiquidity` becomes insufficient for game contracts, as the owner’s distribution of staker rewards affects the liquidity that should only be modified by game contracts.

```solidity
 function transferPayout(address token, address recipient, uint256 amount)
        external
        nonReentrant
        returns (bool)
    {
        require(authorizedGames[msg.sender] || msg.sender == owner(), "Caller not authorized");
        //...
            if (callSucceeded) {
             //...
            lockedLiquidity[token] -= amount;
        }
        //...
    }
```

Recommendations:

Consider introducing a mechanism to track the rewards distributed to **stakers** separately from the `lockedLiquidity`, to ensure that the `lockedLiquidity` remains exclusively controlled by the game contracts.

---
### Example 2

**Auto Label:** Failure to validate state or inputs before reward or token changes leads to incorrect payouts, asset loss, or economic distortion.  

**Original Text Preview:**

In the `Staking` contract, the `removeAcceptedToken()` function allows the owner to remove a token from the `acceptedTokens` list without checking if the token has any `lockedLiquidity`. This causes the `transferPayout()` function to revert when called by a game contract to transfer the token payout to a winner. As a result, the winner loses their winnings, and the tokens become locked in the `Staking` contract, preventing the winner from receiving their payout.

```solidity
    function removeAcceptedToken(address token) external onlyOwner {
        require(tokenInfo[token].totalShares == 0, "Token still has staked shares");
        acceptedTokens[token] = false;
        emit TokenRemoved(token);
    }
```

```solidity
  function transferPayout(address token, address recipient, uint256 amount)
        external
        nonReentrant
        returns (bool)
    {
        //...
        require(acceptedTokens[token], "Token not supported");
        //...
    }
```

Recommendations:

Consider updating the `removeAcceptedToken()` function to check if the token has any `lockedLiquidity` before it is removed from the `acceptedTokens` list, or alternatively, remove the `acceptedTokens` check from the `transferPayout()` function to prevent payout failures.

---
### Example 3

**Auto Label:** Failure to validate state or inputs before reward or token changes leads to incorrect payouts, asset loss, or economic distortion.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `Staking.removeAuthorizedGame()` function is meant to remove a game contract's authorization so it can no longer interact with the `Staking` contract. However, if there is `lockedLiquidity` locked by a game contract while the contract is being removed from the `authorizedGames` list, the locked liquidity will be permanently locked in the `Staking` contract and will not be allocated to stakers.

```solidity
 function transferPayout(address token, address recipient, uint256 amount)
        external
        nonReentrant
        returns (bool)
    {
        require(authorizedGames[msg.sender] || msg.sender == owner(), "Caller not authorized");
        //...
            if (callSucceeded) {
             //...
            lockedLiquidity[token] -= amount;
        }
        //...
    }
```

## Recommendation:

To mitigate this issue, the `Staking` contract should track `lockedLiquidity` per game contract, ensuring that a game contract cannot be removed from the `authorizedGames` list if it still has any locked liquidity.

---
