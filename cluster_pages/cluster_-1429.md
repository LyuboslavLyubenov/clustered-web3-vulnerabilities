# Cluster -1429

**Rank:** #226  
**Count:** 46  

## Label
Rounding and truncation errors in share and token calculations enable attackers to extract profits, drain rewards, or manipulate allocations through imprecise arithmetic and insufficient edge-case validation.

## Cluster Information
- **Total Findings:** 46

## Examples

### Example 1

**Auto Label:** Incorrect rounding in share calculations leads to price manipulation and user fund loss by violating EIP-4626's precision requirements.  

**Original Text Preview:**

## Severity

**Impact:** Low, because it violates EIP and also cause revert in mint()

**Likelihood:** High, because division happens in every mint() call

## Description

According to the EIP4626 function `previewMint()` should round up when calculating assets:

> Finally, EIP-4626 Vault implementers should be aware of the need for specific, opposing rounding directions across the different mutable and view methods, as it is considered most secure to favor the Vault itself during calculations over its users:
>
> If (1) it’s calculating how many shares to issue to a user for a certain amount of the underlying tokens they provide or (2) it’s determining the amount of the underlying tokens to transfer to them for returning a certain amount of shares, it should round down.
>
> If (1) it’s calculating the amount of shares a user has to supply to receive a given amount of the underlying tokens or (2) it’s calculating the amount of underlying tokens a user has to provide to receive a certain amount of shares, it should round up.

in current implementation code rounds down. this will cause calculations to be in favor of the caller instead of the contract.

in function `mint()` code uses `previewMint()` and calls `_deposit()`, as `previewMint()` would calculate smaller amount for `_amount` so the check inside the `_deposit()` that makes sure user don't receive better price than current price would fail and call would revert: (`_amount` would be lower a little and cause right side of the condition to be smaller)

```solidity
        if (_shares > _amount.mulDiv(weiPerShare ** 2, last.sharePrice * weiPerAsset))
            revert AmountTooHigh(_amount);
```

## Recommendations

Change `previewMint` so that it rounds up when calculating shares.

---
### Example 2

**Auto Label:** Rounding bias in asset calculations leads to unfair user allocations and protocol revenue loss through improper rounding direction in withdrawal and share computations.  

**Original Text Preview:**

**Description:** For the L2 `YToken` contracts, assets are not managed directly. Instead, the vault’s exchange rate is provided by an oracle, using the exchange rate from L1 as the source of truth.

This architectural choice requires custom implementations of functions like `previewMint`, `previewDeposit`, `previewRedeem`, and `previewWithdraw`, as well as the internal `_convertToShares` and `_convertToAssets`. These have been re-implemented to rely on the oracle-provided exchange rate instead of local accounting.

However, both `previewMint` and `previewWithdraw` currently perform rounding in favor of the user:

- [`YTokenL2::previewMint`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L249-L250):
  ```solidity
  // Calculate assets based on exchange rate
  return (grossShares * exchangeRate()) / Constants.PINT;
  ```
- [`YTokenL2::previewWithdraw`](https://github.com/YieldFiLabs/contracts/blob/40caad6c60625d750cc5c3a5a7df92b96a93a2fb/contracts/core/tokens/YTokenL2.sol#L261-L262):
  ```solidity
  // Calculate shares needed for requested assets based on exchange rate
  uint256 sharesWithoutFee = (assets * Constants.PINT) / exchangeRate();
  ```

This behavior contradicts the [security recommendations in EIP-4626](https://eips.ethereum.org/EIPS/eip-4626#security-considerations), which advise rounding in favor of the vault to prevent value leakage.

**Impact:** By rounding in favor of the user, these functions allow users to receive slightly more shares or assets than they should. While the two-step withdrawal process limits the potential for immediate exploitation, this rounding error can result in a slow and continuous value leak from the vault—especially over many transactions or in the presence of automation.

**Recommended Mitigation:** Update `previewMint` and `previewWithdraw` to round in favor of the vault. This can be done by adopting the modified `_convertToShares` and `_convertToAssets` functions with explicit rounding direction, similar to the approach used in the [OpenZeppelin ERC-4626 implementation](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC20/extensions/ERC4626Upgradeable.sol#L177-L185).

**YieldFi:** Fixed in commit [`a820743`](https://github.com/YieldFiLabs/contracts/commit/a82074332cc1f57eba398100c3a43e8a70a4c8ce)

**Cyfrin:** Verified. the preview functions now utilizes `_convertToShares` and `_convertToAssets` with the correct rounding direction.

---
### Example 3

**Auto Label:** Rounding and truncation errors in share and token calculations enable attackers to extract profits, drain rewards, or manipulate allocations through imprecise arithmetic and insufficient edge-case validation.  

**Original Text Preview:**

In the `Staking.unstake()` function, the calculation of the shares to be burned for the user is done as follows:

```solidity
79:         uint256 userShare = (userStaked * totalTokenBalance) / info.totalStaked;
80:         require(amount <= userShare, "Cannot unstake more than share");
81:         uint256 fraction = (amount * 1e18) / userShare;
82:
83:         // Decrease user's staked balance proportionally
84:         uint256 sharesToBurn = (userStaked * fraction) / 1e18;
```

The results of the division operations in lines 81 and 84 are rounded down due to the truncation of the division operation, meaning that the shares burned by the user might be underestimated.

Let's consider the following example:

- amount = 1
- userStaked = 10
- totalStaked = 100
- totalTokenBalance = 110
- userShare = 10 \* 110 / 100 = 11
- fraction = 1 \* 1e18 / 11 = 90909090909090909
- sharesToBurn = 10 \* 90909090909090909 / 1e18 = 0

In this case, the user is able to withdraw 1 token, but the shares burned are 0.

The minimum unit of a token is usually a negligible amount, but in the case of tokens with few decimals and high values, the amount can be significant. For example, in WBTC (8 decimals) the minimum unit is valued at approximately 0.001 USD (at current prices) and, the minimum unit of GUSD (2 decimals) is valued at 0.01 USD. In an extreme case where a token with few decimals and high value is used, this attack vector could be exploited to drain the staking pool.

Round up the calculation of `fraction` and `sharesToBurn`, capping the final value to the staked balance of the user.

---
