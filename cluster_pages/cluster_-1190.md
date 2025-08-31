# Cluster -1190

**Rank:** #315  
**Count:** 22  

## Label
Lack of asset validation and state consistency checks enables attackers to misconfigure or manipulate asset handling, leading to unauthorized transfers, fund lock-ups, or incorrect valuation.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Failure to validate transfer outcomes or state conditions leads to inconsistent or incorrect state updates, enabling fund loss, reentrancy, or misrepresentation of asset balances.  

**Original Text Preview:**

## Summary

The function provided by the `RToken` contract to [rescue any ERC20 tokens](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341) that were sent to the contract by mistake is [only callable by](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57) [`LendingPool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57).
However, `LendingPool` [does not offer any function](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734) to call the `RToken` contract, resulting in unrecoverable funds.

## Vulnerability Details

The `Rtoken` contract provides [a function to rescue any ERC20 tokens](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341) that have been sent to the contract directly by mistake.
This function is only callable by the configured `reservePool`, which is ensured by the [`onlyReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57) [modifier](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57).
This will be [`LendingPool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L33) in practice.

Looking at the `LendingPool` contract, it comes indeed with a [`rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734) [function](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734), however, that one only allows for rescuing funds from itself.
It does not provide the necessary functionality to recover funds from the `RToken` contract.

Here's what it looks like:

```Solidity
function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
    require(tokenAddress != reserve.reserveRTokenAddress, "Cannot rescue RToken");
    IERC20(tokenAddress).safeTransfer(recipient, amount);
}
```

Notice how it uses `safeTransfer` to move funds from itself to the `recipient`, but it does not call into the `RToken` contract.

## Impact

ERC20 tokens, that are not the configured reserve asset, which are sent to the `RToken` contract by mistake are not recoverable until the owner of the protocol configures a new `reservePool` that provides the necessary functionality

## Tools Used

Manual review.

## Recommendations

There's two ways to go about this:

1. Either allow the `owner` of the protocol to call `rescueFunds` on `RToken` directly or
2. Extend `LendingPool`, which is the expected `reservePool` in production, to provide the necessary function.

I'd recommend going for option 1) simply because any change of `reservePool` could reintroduce this issue.
Here's the necessary change:

```diff
- function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyReservePool {
+ function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyOwner {
    if (recipient == address(0)) revert InvalidAddress();
    if (tokenAddress == _assetAddress) revert CannotRescueMainAsset();
    IERC20(tokenAddress).safeTransfer(recipient, amount);
}
```

## Relevant links

* [`RToken#rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L337-L341)
* [`RToken#onlyReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L54-L57)
* [`RToken#setReservePool`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/tokens/RToken.sol#L85-L90)
* [`LendingPool#rescueToken`](https://github.com/Cyfrin/2025-02-raac/blob/main/contracts/core/pools/LendingPool/LendingPool.sol#L731-L734)

* [Similar finding other contest](https://solodit.cyfrin.io/issues/m-4-rescuetokens-feature-is-broken-sherlock-notional-leveraged-vaults-pendle-pt-and-vault-incentives-git)

---
### Example 2

**Auto Label:** Failure to validate and transfer underlying assets during migration leads to fund loss and invalid state, enabling operational failure and user exposure.  

**Original Text Preview:**

## Context: ExternalGuardian.sol#L83-L87

## Description

When rescue starts, the `_guardian_underlying_withdraw` (see ExternalGuardian.sol#L84) withdraws underlying funds back to the strategy contract.

```solidity
(tokens, rescued) = _guardian_underlying_maxWithdraw();
IEarnStrategy.WithdrawalType[] memory types = _guardian_underlying_withdraw(0, tokens, rescued, address(this));
if (!_areAllImmediate(types)) {
    revert OnlyImmediateWithdrawalsSupported();
}
```

For example, if the strategy is AAVE V3, xxD:
- User deposits underlying asset in exchange for aToken.
- When withdrawing, aToken in the contract is burnt and underlying asset is transferred back.

Yet when the strategy is migrated, only the aToken is transferred (see AaveV3Connector.sol#L319):

```solidity
function _connector_migrateToNewStrategy(
    IEarnStrategy newStrategy,
    bytes calldata
) internal override returns (bytes memory) {
    IERC20 vault_ = aToken();
    uint256 balance = vault_.balanceOf(address(this));
    vault_.safeTransfer(address(newStrategy), balance);
    return abi.encode(balance);
}
```

The compound V2 strategy follows the same pattern, where only cToken is transferred out.

## Sequence of Actions

1. The strategy owner proposes a strategy migration.
2. The migration is subject to a 3-day time-lock (see EarnStrategyRegistry.sol#L110).
3. During these 3 days, a rescue transaction is executed, and all aToken is burnt to withdraw underlying token.
4. Migration is executed; yet no underlying asset is transferred to the new strategy.

## Recommendation

While executing the rescue in the 3-day time-lock is an edge case, the protocol can consider:
- Transfer both aToken and underlying token to the new strategy contract in AAVE V3 connector.
- Transfer both cToken and underlying token to the new strategy contract in Compound V2 connector.
- Transfer both vault share and vault underlying asset to the new strategy contract in ERC4626 connector.

## Balmy

We've decided to simply disable migrations on strategies that:
- Have an ongoing rescue (one that still needs confirmation).
- Have a confirmed rescue.

The fix is in PR 109.

## Cantina Managed

Fixed.

---
### Example 3

**Auto Label:** Lack of asset validation and state consistency checks enables attackers to misconfigure or manipulate asset handling, leading to unauthorized transfers, fund lock-ups, or incorrect valuation.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Undeﬁned Behavior

## Description
According to the Treehouse team, the Vault is a single-asset vault: we went from multiple assets within a single vault to multiple vaults with a single asset. However, the current Vault is not a single-asset vault, as demonstrated by the presence of functions to control the “allowed assets” in the Vault contract.

```solidity
function addAllowableAsset(address _asset) external onlyOwner {
    if (IERC20Metadata(_asset).decimals() > 18) revert UnsupportedDecimals();
    RATE_PROVIDER_REGISTRY.checkHasRateProvider(_asset);
    bool success = _allowableAssets.add(_asset);
    if (!success) revert Failed();
    emit AllowableAssetAdded(_asset);
}
```
_Figure 4.1: The addAllowableAsset function in Vault.sol_

Although this does not pose an immediate security risk, we do want to highlight this issue, as it complicates the Vault compared to an actual “single-asset vault.”

## Recommendations
Consider converting all assets (for example, in the TreehouseRouter) to a single underlying asset, and only allow that asset to be deposited into the Vault. This way, the Vault could be a true “single-asset vault.”

---
