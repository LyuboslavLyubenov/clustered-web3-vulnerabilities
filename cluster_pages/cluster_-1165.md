# Cluster -1165

**Rank:** #183  
**Count:** 64  

## Label
Failure to account for dynamic token balances during transfers or withdrawals, leading to incorrect state updates, fund loss, and exploitable inconsistencies in rebasing or fee-on-transfer token systems.

## Cluster Information
- **Total Findings:** 64

## Examples

### Example 1

**Auto Label:** Failure to account for dynamic token balances during transfers or withdrawals, leading to incorrect state updates, fund loss, and exploitable inconsistencies in rebasing or fee-on-transfer token systems.  

**Original Text Preview:**

**Description:** As part of the implementation of the autonomous rebalance mechanism in `BunniHook::rebalanceOrderPreHook`, the hook balance of the order output token is set in transient storage before the order is executed. This occurs before the unlock callback, during which `hookHandleSwap()` is called within `_rebalancePrehookCallback()`:

```solidity
    function rebalanceOrderPreHook(RebalanceOrderHookArgs calldata hookArgs) external override nonReentrant {
        ...
        RebalanceOrderPreHookArgs calldata args = hookArgs.preHookArgs;

        // store the order output balance before the order execution in transient storage
        // this is used to compute the order output amount
        uint256 outputBalanceBefore = hookArgs.postHookArgs.currency.isAddressZero()
            ? weth.balanceOf(address(this))
            : hookArgs.postHookArgs.currency.balanceOfSelf();
        assembly ("memory-safe") {
@>          tstore(REBALANCE_OUTPUT_BALANCE_SLOT, outputBalanceBefore)
        }

        // pull input tokens from BunniHub to BunniHook
        // received in the form of PoolManager claim tokens
        // then unwrap claim tokens
        poolManager.unlock(
            abi.encode(
                HookUnlockCallbackType.REBALANCE_PREHOOK,
@>              abi.encode(args.currency, args.amount, hookArgs.key, hookArgs.key.currency1 == args.currency)
            )
        );

        // ensure we have at least args.amount tokens so that there is enough input for the order
@>      if (args.currency.balanceOfSelf() < args.amount) {
            revert BunniHook__PrehookPostConditionFailed();
        }
        ...
    }

    /// @dev Calls hub.hookHandleSwap to pull the rebalance swap input tokens from BunniHub.
    /// Then burns PoolManager claim tokens and takes the underlying tokens from PoolManager.
    /// Used while executing rebalance orders.
    function _rebalancePrehookCallback(bytes memory callbackData) internal {
        // decode data
        (Currency currency, uint256 amount, PoolKey memory key, bool zeroForOne) =
            abi.decode(callbackData, (Currency, uint256, PoolKey, bool));

        // pull claim tokens from BunniHub
@>      hub.hookHandleSwap({key: key, zeroForOne: zeroForOne, inputAmount: 0, outputAmount: amount});

        // lock BunniHub to prevent reentrancy
        hub.lockForRebalance(key);

        // burn and take
        poolManager.burn(address(this), currency.toId(), amount);
        poolManager.take(currency, address(this), amount);
    }
```

The transient storage is again queried in the call to `rebalanceOrderPostHook()` to ensure that only the specified amount of the output token (consideration item) is transferred from `BunniHook`:

```solidity
    function rebalanceOrderPostHook(RebalanceOrderHookArgs calldata hookArgs) external override nonReentrant {
        ...
        RebalanceOrderPostHookArgs calldata args = hookArgs.postHookArgs;

        // compute order output amount by computing the difference in the output token balance
        uint256 orderOutputAmount;
        uint256 outputBalanceBefore;
        assembly ("memory-safe") {
@>          outputBalanceBefore := tload(REBALANCE_OUTPUT_BALANCE_SLOT)
        }
        if (args.currency.isAddressZero()) {
            // unwrap WETH output to native ETH
            orderOutputAmount = weth.balanceOf(address(this));
            weth.withdraw(orderOutputAmount);
        } else {
            orderOutputAmount = args.currency.balanceOfSelf();
        }
@>      orderOutputAmount -= outputBalanceBefore;
        ...
```

However similar such validation is not applied to the input token (offer item). While the existing validation shown above is performed to ensure that the hook holds sufficient tokens to process the order, it fails to consider that the hook may hold funds designated to other recipients. Previously, per the Pashov Group finding H-04, autonomous rebalance was DoS’d by checking the token balance was strictly equal to the order amount but forgot to account for am-AMM fees and donations. The recommendation was to check that the contract’s token balance increase is equal to `args.amount`, not the total balance.

Given that am-AMM fees are stored as ERC-6909 balances within the `BunniHook`, these could be erroneously used as part of the rebalance order input amount. This could occur when the hook holds insufficient raw balance to cover the input amount, perhaps when rehypothecation is enabled and the `hookHandleSwap()` call pulls fewer tokens than expected due to the vault returning fewer tokens than specified. Instead, the input token balance should be set in transient storage before the unlock callback in the same way that `outputBalanceBefore` is. Then, the difference between the input token balances before the unlock callback and after should be validated to satisfy the order input amount.

**Impact:** am-AMM fees stored as ERC-6909 balance inside `BunniHook` could be erroneously used as rebalance input due to the incorrect balance check.

**Recommended Mitigation:** Consider modifying the validation performed after the unlock callback to:
```solidity
uint256 inputBalanceBefore = args.currency.balanceOfSelf();

// unlock callback

if (args.currency.balanceOfSelf() - inputBalanceBefore < args.amount) {
    revert BunniHook__PrehookPostConditionFailed();
}
```

**Bacon Labs:** Fixed in [PR \#98](https://github.com/timeless-fi/bunni-v2/pull/98) and [PR \#133](https://github.com/timeless-fi/bunni-v2/pull/133).

**Cyfrin:** Verified, stricter input amount validation has been added to `BunniHook::rebalanceOrderPreHook`.

---
### Example 2

**Auto Label:** Failure to validate token balances after transfers, leading to state inconsistencies and potential financial loss due to unaccounted fees or insufficient balances.  

**Original Text Preview:**

##### Description
The issue arises within the `receiveTokens`  function of the `LOB` contract .

There is a vulnerability related to the improper handling of fee-on-transfer tokens. The function does not verify that the balances have been adjusted as expected after the `safeTransferFrom` calls. This oversight can result in incorrect token balances, especially when dealing with tokens that impose transfer fees or employ other mechanisms that alter the expected transfer amount.

The issue is classified as **high** severity as it can lead to discrepancies in token balances, potentially causing financial inconsistencies and loss.
##### Recommendation
We recommend adding a check to ensure that the balances reflect the expected changes after the `safeTransferFrom` calls. If the balances do not match the expected change, the transaction should be reverted to safeguard against financial irregularities.

---
### Example 3

**Auto Label:** **State inconsistency in token burn operations due to failed state updates or accounting errors, leading to misreported token supply and lost user assets.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

Upon minting, we have the following code:

```solidity
if (range.lower == 0 && range.upper == 0) {
            uint256 mintShares = islandLiqToShares(liq);
            island.mint(mintShares, recipient);
        } else {
            // mint the V3 ranges
            pool.mint(address(this), range.lower, range.upper, liq, abi.encode(msg.sender));
        }
```

When minting for the island, we set the recipient as the `recipient` input and when minting Uniswap V3 liquidity, we set the recipient as `address(this)`. The latter is correct while the former is not. This is because when burning, the shares will be burned from the caller of the `burn()` function on the target, which will be the `Burve` contract. As the `Burve` contract does not have the minted shares when minting for the island, we will simply revert.

The user still has 2 options, thus the medium impact:

- batch a transaction by transferring the shares to the `Burve` contract and then burning the liquidity, this will result in the correct result
- simply burn his shares directly on the island, note that this will result in the user still having the minted `Burve` shares

## Recommendations

```diff
-  island.mint(mintShares, recipient);
+  island.mint(mintShares, address(this));
```

---
