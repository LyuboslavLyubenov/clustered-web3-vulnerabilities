# Cluster -1270

**Rank:** #164  
**Count:** 80  

## Label
Inaccurate state tracking leading to flawed interest, borrowing, or liquidity calculations, enabling unfair distributions, incorrect fee allocations, or bypassing safety thresholds.

## Cluster Information
- **Total Findings:** 80

## Examples

### Example 1

**Auto Label:** **State inconsistency due to improper sequencing or external state modifications, leading to accounting errors, liquidity loss, or mispriced reserves.**  

**Original Text Preview:**

**Description:** Bunni utilizes Hooklets to enable pool deployers to inject custom logic into various pool operations, allowing for the implementation of advanced strategies, customized behavior, and integration with external systems. These Hooklets can be invoked during key operations, including initialization, deposits, withdrawals, swaps, and Bunni token transfers.

Based on the `IHooklet` interface, it appears that "before operation" Hooklet invocations are passed the user input data and in turn "after operation" Hooklet invocations receive the operations return data. The return data from deposit, withdraw, and swap functionality always contains pool reserve changes, which can be used in the custom logic of a Hooklet:

```solidity
    struct DepositReturnData {
        uint256 shares;
@>      uint256 amount0;
@>      uint256 amount1;
    }

    ...

    struct WithdrawReturnData {
@>      uint256 amount0;
@>      uint256 amount1;
    }

    ...

    struct SwapReturnData {
        uint160 updatedSqrtPriceX96;
        int24 updatedTick;
@>      uint256 inputAmount;
@>      uint256 outputAmount;
        uint24 swapFee;
        uint256 totalLiquidity;
    }
```

However, the protocol rebalancing functionality fails to deliver any such information to Hooklets, meaning that any custom Hooklet logic based on changing reserves can not be implemented.

**Impact:** The absence of reserve information for rebalancing operations limits the ability of Hooklets to implement custom logic that relies on this data. This inconsistency in data provisioning may hinder the development of advanced strategies and customized behavior, ultimately affecting the overall functionality and usability of the protocol.

**Recommended Mitigation:** Bunni should provide reserve change information to its Hooklets during rebalancing operations. This could be achieved by reusing the `afterSwap()` hook, and implementing the `beforeSwap()` hook could also be useful; however, it would be preferable to introduce a distinct rebalance hooklet call.

**Bacon Labs:** Fixed in [PR \#121](https://github.com/timeless-fi/bunni-v2/pull/121) and [PR \#133](https://github.com/timeless-fi/bunni-v2/pull/133).

**Cyfrin:** Verified, `IHooklet.afterRebalance` is now called after rebalance order execution.

---
### Example 2

**Auto Label:** **Incorrect asset accounting due to missing state updates and improper balance tracking, leading to fund loss, inflated reserves, and compromised protocol integrity.**  

**Original Text Preview:**

##### Description
The `FORCE_RELEASER_ROLE` can call `TreasuryIntermediateEscrow.forceRelease()` with a `_payoutAmount` of zero multiple times. This leads to a decrease in the `lockedBalances` mapping for a specific asset by an amount greater than the `depositedAmount` of a given `_escrowId`. As a result, `lockedBalances` will decrease faster than expected and may underflow when the last locked entry is released.

https://github.com/resolv-im/resolv-contracts/blob/91c52d9801c6a37f39566e2854722be1700dedc0/contracts/TreasuryIntermediateEscrow.sol#L111

##### Recommendation
We recommend adding a restriction that `_payoutAmount` is not equal to zero in `TreasuryIntermediateEscrow.forceRelease()`.

---
### Example 3

**Auto Label:** **Incorrect state updates due to flawed logic and missing boundaries, leading to erroneous balances, locked assets, and compromised protocol stability.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

- In `processWithdrawals()` function: when the `_withdrawShares = 0`, the `s.totalLiquidity` is increased by the liquidity of the collected-and-deposited fees of the position, where it assumes that the totalLiquidity of the position will not be decreased when `_reviseHedge()` is called:

```javascript
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
             s.totalLiquidity = _totalLiquidity;
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

Where:

```javascript
 function _reviseHedge(
        uint256 _totalLiquidity,
        bytes memory path0,
        bytes memory path1
    ) internal virtual {
        uint256 _newHedgeLiquidity = (_totalLiquidity * s.hedgeSize) / 1e18;

        uint256 _prevHedgeLiquidity = _loanData.liquidity;

        if (_newHedgeLiquidity > _prevHedgeLiquidity) {
            _increaseHedge(
                _newHedgeLiquidity - _prevHedgeLiquidity,
                path0,
                path1
            );
        } else if (_newHedgeLiquidity < _prevHedgeLiquidity) {
            _decreaseHedge(
                _prevHedgeLiquidity - _newHedgeLiquidity,
                path0,
                path1
            );
        }
    }
```

- Given that there's a setter for the `hedgeSize` (where it's used to calculate the `_newHedgeLiquidity`), then the `s.totalLiquidity` should be updated based on the liquidity position in case the `hedgeSize` was updated (also, in `_reviseHedge()`, the hedge liquidity is fetched based on the last updated data without checking if the hedge has accrued interest over time).

- So if the new `hedgeSize` is decreased and results in `_newHedgeLiquidity < _prevHedgeLiquidity` , then the hedge will be decreased, which might result in the hedge incurring losses, hence affecting the liquidity, which will affect the next process that's going to be executed after the withdrawal: if the next process is a deposit, then this incorrect `s.totalLiquidity` will result in minting wrong shares for that deposited period as the liquidity doesn't reflect the actual one (since it's the cashed one + liquidity from the collected-and-deposited fees of the LP position).

- Same issue in `DepositProcess.processDeposits()` function.

## Recommendations

```diff
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
-            s.totalLiquidity = _totalLiquidity;
+            s.totalLiquidity = _getTotalLiquidity(s.tokenId);
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

---
