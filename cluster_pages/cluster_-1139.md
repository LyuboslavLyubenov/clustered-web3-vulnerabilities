# Cluster -1139

**Rank:** #187  
**Count:** 61  

## Label
Failure to properly handle podded assets in autocompounding and leverage flows, leading to reward misallocation, liquidity loss, and system failures due to unaccounted token taxes and missing deposit logic.

## Cluster Information
- **Total Findings:** 61

## Examples

### Example 1

**Auto Label:** Flawed conditional logic and incorrect token handling lead to failed swaps, misrouted exits, and broken compounding—causing functional gaps and unintended asset losses without enabling direct exploits.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

In the `_swap()` function in the `MobiusRouter` contract, there is an issue when the `poolPath` contains duplicate pool addresses. Each pool's `swap()` function is protected by the `conditionallySingleCallPerTransaction()` modifier, which is designed to prevent more than one call to the function per transaction:

```solidity
function _swap(address[] calldata tokenPath, address[] calldata poolPath, uint256 fromAmount, address to)
    internal
    returns (uint256 amountOut, uint256 haircut)
{
    // [code omitted for brevity]

    for (uint256 i; i < poolPath.length; ++i) {
        // [code omitted for brevity]

        // make the swap with the correct arguments
        (amountOut, localHaircut) = IPool(poolPath[i]).swap(
            tokenPath[i],
            tokenPath[i + 1],
            nextFromAmount,
            0, // minimum amount received is ensured on calling function
            nextTo,
            type(uint256).max // deadline is ensured on calling function
        );
        // [code omitted for brevity]
    }
}
```

The `conditionallySingleCallPerTransaction()` modifier works by checking the gas consumed when accessing an address's balance:

```solidity
modifier conditionallySingleCallPerTransaction() {
    if (coldAccessGasCost != -1) {
        address addressToCheck = address(uint160(bytes20(blockhash(block.number))));
        uint256 initialGas = gasleft();
        uint256 temp = addressToCheck.balance;
        uint256 gasConsumed = initialGas - gasleft();
        if (gasConsumed != uint256(coldAccessGasCost)) revert AlreadyCalledInThisTransaction();
    }
    _;
}
```

When a pool's `swap()` function is called multiple times in the same transaction, the second call will revert with the `AlreadyCalledInThisTransaction` error. This means that any router transaction with a `poolPath` containing duplicate pool addresses will fail. **But the protocol design allows pools to support multiple assets (as shown in `StablePool` and `VariantPool`), so this creates a limitation in constructing efficient swap paths.**

```solidity
function addAsset(address token, address asset) external onlyOwner {
      if (token == address(0)) revert Pool_AddressShouldNotBeZero();
      if (asset == address(0)) revert Pool_AddressShouldNotBeZero();
      if (_containsAsset(token)) revert Pool_AssetAlreadyExists();

      _addAsset(token, VariantAsset(asset));

      emit AssetAdded(token, asset);
}
```

For example:

1. `PoolA` has `USDC`, `USDT`, and `ETH`, while `PoolB` only has `USDT` and `ETH`.
2. A user spots a price gap in `ETH` between the pools, creating an arbitrage opportunity. Their swap strategy is:
   - `PoolA`: `USDC` → `ETH`
   - `PoolB`: `ETH` → `USDT`
   - `PoolA`: `USDT` → `USDC`
3. This is a valid arbitrage strategy, but `MobiusRouter.swapTokensForTokens()` can't execute it.

## Location of Affected Code

File: [src/router/MobiusRouter.sol](https://github.com/MobiusExchange/core/blob/21eeeca84ea26abfb131f758164256af8a706aaa/src/router/MobiusRouter.sol)

## Impact

Any router transaction with a `poolPath` containing duplicate pool addresses will fail. But the protocol design allows pools to support multiple assets (as shown in `StablePool` and `VariantPool`), so this creates a limitation in constructing efficient swap paths.

## Recommendation

If duplicate pools should not be supported, add a check in `MobiusRouter.swapTokensForTokens()` with a direct error. If they should be supported, remove `conditionallySingleCallPerTransaction()` from `pool.swap()`.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Flawed conditional logic and incorrect token handling lead to failed swaps, misrouted exits, and broken compounding—causing functional gaps and unintended asset losses without enabling direct exploits.  

**Original Text Preview:**

## Diﬃculty: Medium

## Type: Undeﬁned Behavior

## Description
The AMM contract fails to account for locked SY tokens when validating PT-to-SY swaps, which can lead to users losing their PT tokens without receiving any SY tokens in return. In the FIVA protocol, users can swap between different token types (SY, PT, and YT) through the AMM pool. During certain operations, some tokens may be temporarily locked in the pool. The protocol tracks these locked tokens using the `storage::sy_locked` and `storage::pt_locked` state variables.

When a user attempts to swap PT for SY tokens, the `handle_swap_pt_for_sy` function should verify that the pool has enough available (non-locked) SY tokens to fulfill the swap. However, unlike other swap functions in the codebase, this function checks against only the total SY supply without accounting for locked tokens:

```plaintext
if ((sy_out < max(min_sy_out, MIN_SWAP_AMOUNT)) | (sy_out > (storage::total_supply_sy - MINIMUM_LIQUIDITY))) {
```

_Figure 8.1: The balance check in the PT to SY swaps contracts/AMM/markets/swaps.fc#75_

In contrast, other swap functions correctly account for locked tokens. For example, the `handle_swap_sy_for_yt` function includes `storage::sy_locked` in its validation:

```plaintext
(sy_out > (storage::total_supply_sy - storage::sy_locked - MINIMUM_LIQUIDITY))
```

_Figure 8.2: The balance check in the SY to YT swaps contracts/AMM/markets/swaps.fc#124_

This inconsistency creates a problematic scenario when locked SY tokens are present in the pool. The swap validation may incorrectly pass because it checks against only the total SY supply, not the available supply. When the transaction proceeds to actually transfer SY tokens, it will fail due to an insufficient available balance. This would result in the user permanently losing their PT tokens without receiving any SY tokens in return, while leaving the pool in an inconsistent state where its accounting and actual token balances are out of sync, potentially affecting future operations.

## Exploit Scenario
A user attempts to swap 200 PT tokens for SY in a pool where a significant amount of SY tokens are locked. The pool has a total of 1,000 SY tokens, but 900 of these are locked, leaving only 100 SY tokens actually available. The validation check passes incorrectly because it only compares against the total supply of SY tokens (1,000), not the available supply (100). When the transaction proceeds and the pool’s state is updated, the user’s 200 PT tokens are added to the pool, but the transfer of 200 SY tokens fails due to insufficient available balance. As a result, the user loses their PT tokens without receiving any SY tokens, and the pool’s accounting becomes inconsistent with its actual token balances.

## Recommendations
- **Short term**: Modify the validation check in `handle_swap_pt_for_sy` to account for locked SY tokens.
- **Long term**: Expand the test suite to specifically verify swap behavior when tokens are locked in the pool.

---
### Example 3

**Auto Label:** **Scaling and precision mismatches in token amount calculations lead to incorrect fund withdrawals, corrupted balances, and undercollateralization.**  

**Original Text Preview:**

The `Arbitrum_CustomGasToken_Adapter` [contract](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L130) is an adapter meant to handle cases where the destination chain uses a custom token to charge gas fees. Such a token might have non-standard decimals so proper scaling must be performed in order to correctly calculate the amounts.

The `_pullCustomGas` [function](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L284) of this contract is used both by the [`relayTokens`](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L221) and [`relayMessage`](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L195) functions to calculate the amount of gas tokens needed to pay for the chosen operation. This function first calculates the amount owed in the `getL1CallValue` [function](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L285) and then [withdraws](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L286) this amount from the `CUSTOM_GAS_TOKEN_FUNDER` contract (a specialized contract that holds the gas token funds needed).

The `getL1CallValue` function calculates the required amount of gas tokens by using the following formula:

```
 return L2_MAX_SUBMISSION_COST + L2_GAS_PRICE * l2GasLimit

```

In the above formula:

* `L2_MAX_SUBMISSION_COST` is the amount of gas allocated to pay for the base submission fee of the L1->L2 operation. [According](https://github.com/OffchainLabs/nitro-contracts/blob/main/src/bridge/AbsInbox.sol#L236) to the `AbsInbox` Arbitrum contract, this amount must be represented with an 18-decimal scale.
* `L2_GAS_PRICE` is the gas price bid for the immediate L2 execution attempt. According to the same `AbsInbox` contract, the scale should also be 18 decimals.
* `l2GasLimit` is a parameter which is in pure units of gas and depends on which operation (`relayTokens` or `relayMessage`) is being performed.

As one can see, the amount returned by the `getL1CallValue` function is in an 18-decimal scale and is directly used for withdrawing funds from the custom gas token funder. However, this is incorrect since the custom gas token might have a different scale.

For example, if the custom gas token is USDC which has 6 decimals, the amount being charged is off the scale by a factor of `10**(18 - 6)`. The result is that the `Arbitrum_CusstomGasToken_Adapter` logic will withdraw an amount way bigger than what is needed. What is worse is that such an amount is then directly passed to the `createRetryableTicket` [function](https://github.com/across-protocol/contracts/blob/f56146a01ca9c62e6206a2c23c55dbe01a25a912/contracts/chain-adapters/Arbitrum_CustomGasToken_Adapter.sol#L206) of the L1 Arbitrum inbox contract, which is then responsible for [pulling](https://github.com/OffchainLabs/nitro-contracts/blob/main/src/bridge/ERC20Inbox.sol#L139) these tokens out of the calling contract, overcharging the needed amount by many factors.

Consider scaling the amount returned by the `getL1CallValue` function by the amount of decimals of the custom gas token, using the scaled value to withdraw the correct amount from the funder contract, and passing it to the `createRetryableTicket` function.

***Update:** Resolved in [pull request #589](https://github.com/across-protocol/contracts/pull/589). The Risk Labs team stated:*

> *Nice catch.*

---
