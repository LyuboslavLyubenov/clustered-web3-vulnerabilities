# Cluster -1366

**Rank:** #311  
**Count:** 23  

## Label
Misleading balance tracking and flawed deposit mechanisms lead to incorrect user entitlements, fund depletion, and loss of rewards due to failure to account for dynamic token supply or transfer fees.

## Cluster Information
- **Total Findings:** 23

## Examples

### Example 1

**Auto Label:** Failure to account for transfer fees in token operations leads to incorrect balances, inflated settlements, and lost funds due to unadjusted amount calculations and missing fee validation.  

**Original Text Preview:**

The `removeToken` function does not check if a token has a non-zero protocol fee before removal. This can prevent the feeDistributor from claiming fees after the token is removed.

````solidity
function removeToken(address _token) external onlyRole(AccessControlStorage.DEFAULT_ADMIN_ROLE) {
    PoolStorage.Layout storage strg = PoolStorage.layout();
    if (strg.ledger.state.poolAmount[_token] != 0) revert PoolErrors.PoolNotEmpty(_token);
    strg.setUp.assets.allAssets.remove(_token);
    strg.setUp.assets.stablecoins.remove(_token);

    if (strg.ledger.state.rewardsByToken[_token].totalRewards == 0) strg.setUp.assets.rewardTokens.remove(_token);
}

---
### Example 2

**Auto Label:** Failure to account for transfer fees in token operations leads to incorrect balances, inflated settlements, and lost funds due to unadjusted amount calculations and missing fee validation.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The `Swappee` contract contains a vulnerability in its handling of Fee-on-Transfer (FOT) tokens. The issue manifests in the swappee function, where the contract assumes the actual received amount of tokens will equal the transferred amount, without accounting for potential transfer fees. Specifically, when processing token transfers via `IERC20(inputToken).transferFrom()`, the contract uses the pre-transfer amount (`amountsClaimedPerWallet[inputToken][msg.sender]`) for all subsequent calculations and approvals, rather than checking the actual received balance. This assumption breaks for FOT tokens, where the received amount may be less than the transferred amount due to built-in transfer fees.

## Location of Affected Code

File: [src/Swappee.sol#L121](https://github.com/smilee-finance/swappee-smart-contracts/blob/16315aa674ffce54e36fadca66da3cf6785150de/src/Swappee.sol#L121)

```solidity
function swappee(
    IBGTIncentiveDistributor.Claim[] calldata claims,
    RouterParams[] memory routerParams,
    address tokenOut
)
    public
    invariantCheck
{

    // code

    IERC20(inputToken).transferFrom(msg.sender, address(this), amount);

    unchecked {
        amountsClaimedPerWallet[inputToken][msg.sender] -= amount;
    }

    if (routerParam.swapTokenInfo.inputAmount != amount) {
        revert InvalidAmount();
    }

    IERC20(inputToken).approve(aggregator, routerParam.swapTokenInfo.inputAmount);

    // Override router params to avoid tempered inputs
    routerParam.swapTokenInfo.outputReceiver = address(this);
    routerParam.swapTokenInfo.outputToken = tokenOut;

    uint256 amountOut = _swapToken(
        routerParam.swapTokenInfo,
        routerParam.pathDefinition,
        routerParam.executor,
        routerParam.referralCode,
        tokenOut
    );

    // code
}
```

## Impact

When users attempt to swap FOT tokens, the contract will: incorrectly calculate fees based on the pre-fee amount, potentially approve more tokens than necessary to the aggregator, and may trigger reverts when attempting to swap the full pre-fee amount that isn't actually available.

## Recommendation

The contract should implement proper FOT token handling by: measuring actual received balances before and after transfers, using the delta as the effective amount for swaps.

## Team Response

Acknowledged.

---
### Example 3

**Auto Label:** Misleading balance tracking and flawed deposit mechanisms lead to incorrect user entitlements, fund depletion, and loss of rewards due to failure to account for dynamic token supply or transfer fees.  

**Original Text Preview:**

##### Description
An oversight has been detected within the `_deposit` function of the `EnjoyoorsVaultDeposits` contract. 

The issue arises from the absence of a fee-on-transfer protection mechanism. This could result in incorrect setting of user amounts. Consequently, users might be in a position to claim more than they are entitled to. It gravely threatens the contract's integrity since this scenario inevitably leads to a situation where the last user cannot claim any funds, given there are insufficient funds on the contract due to the over-claims made by the preceding users.

This issue is classified as **Medium** severity due to its potential to distort the distribution of deposits and consequently funds, which could lead to the breakdown of the contract's operation.

##### Recommendation
We strongly suggest the addition of a fee-on-transfer protection within the `_deposit` function. This would ensure the correct amount is allocated to the user upon executing a deposit transaction, especially for tokens with transfer fees. This can be done by adding balance checks for the token before and after the `transferFrom` call. Note that it is crucial to add a `nonReentrant` check to protect the function from the reentrancy attack.

---
