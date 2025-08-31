# Cluster -1205

**Rank:** #148  
**Count:** 95  

## Label
Race conditions enabling front-running and state-dependent exploits, allowing attackers to manipulate share allocations, trigger reverts, or withdraw funds prematurely through transaction ordering vulnerabilities.

## Cluster Information
- **Total Findings:** 95

## Examples

### Example 1

**Auto Label:** **Race conditions and dynamic state inference enable attackers to manipulate prices, trigger unintended liquidations, or exploit timing gaps for fund drainage and oracle manipulation.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-06-superfluid-locker-system-judging/issues/245 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
0rpse, 0x73696d616f

### Summary

Locker owners can match their SUP tokens with ETH and deploy liquidity into uniswapv3 pools, after doing this for 6 months they will get their SUP without paying tax, otherwise this tax can get up to 80%. The problem is that `FluidLocker.sol::withdrawLiquidity` transfers ETH to locker owners without considering price manipulations, because of this a locker owner can manipulate a pool's price such that  liquidity will mostly consist of ETH and withdraw their liquidity in ETH, without paying taxes.

### Root Cause

In `FluidLocker.sol::withdrawLiquidity` there is no check against price manipulations
https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/fluid/packages/contracts/src/FluidLocker.sol#L447-L485

### Internal Pre-conditions

N/A

### External Pre-conditions

1. Pool's liquidity is "low"

### Attack Path

Assuming 1 ETH = 2000 SUP, and the initial liquidity is spread across the whole range, since this would be more resistant to price manipulation:
1. Pool's reserves consist of 25 ETH and 50_000 SUP
2. Locker owner matches this with 25 ETH and 50_000 SUP
3. Locker owner flashloans and swaps 1500 ETH for 96764 SUP, pool reserves are 1550 ETH - 3235 SUP
4. Locker owner withdraws liquidity, getting 775 ETH paid to his address and 1617.5 SUP back in the locker
5. Locker owner swaps 46764 SUP of the 96764 SUP he had in the first swap, getting 749 ETH back
In the end locker owner walks away with 22.65 ETH after paying the flashloan back with 0.09% fees, and 50K SUP in his address and 1617 SUP locked.

### Impact

Depending on the liquidity present in the pools, a locker owner can bypass the tax mechanism with miniscule amounts lost compared to the normal exits present in the protocol.

### Mitigation

Either make sure there is enough liquidity in the pools or use TWAP oracle to see if the price has been pushed to extreme while withdrawing liquidity.

---
### Example 2

**Auto Label:** **Race conditions and dynamic state inference enable attackers to manipulate prices, trigger unintended liquidations, or exploit timing gaps for fund drainage and oracle manipulation.**  

**Original Text Preview:**

**Description:** The codebase makes use of modified vendored versions of the `LiquidityAmounts` and `SqrtPriceMath` libraries (excluded from the scope of this review). However, the current implementation of the modified `LiquidityAmounts` library still references the original Uniswap v4 version of `SqrtPriceMath`, rather than the intended modified vendored version. This appears to be unintentional and affects the `queryLDF()` function which performs an unsafe downcast on `liquidityDensityOfRoundedTickX96` from `uint256` to `uint128` when invoking `LiquidityAmounts::getAmountsForLiquidity` as shown below:

```solidity
(uint256 density0OfRoundedTickX96, uint256 density1OfRoundedTickX96) = LiquidityAmounts.getAmountsForLiquidity(
    sqrtPriceX96, roundedTickSqrtRatio, nextRoundedTickSqrtRatio, uint128(liquidityDensityOfRoundedTickX96), true
);
```

While a silent overflow is not possible so long as the invariant holds that the liquidity density of a single rounded tick cannot exceed the maximum normalized density of `Q96`, the `LiquidityAmounts` library should be updated to correctly reference the modified version such that the need to downcast is avoided completely.

**Bacon Labs:** Fixed in [PR \#111](https://github.com/timeless-fi/bunni-v2/pull/111).

**Cyfrin:** Verified, the custom `SqrtPriceMath` library is now used within `LiquidityAmounts` instead of that from `v4-core`.

---
### Example 3

**Auto Label:** **Race conditions and dynamic state inference enable attackers to manipulate prices, trigger unintended liquidations, or exploit timing gaps for fund drainage and oracle manipulation.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/937 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
0xc0ffEE, 0xpetern, 4b, Bizarro, Constant, Cybrid, EgisSecurity, Goran, JuggerNaut, Mimis, Ob, Petrus, ZeroTrust, bladeee, cccz, dhank, fromeo\_016, iamandreiski, richa, sheep, silver\_eth, the\_haritz, upWay, x0lohaclohell

### Summary

The reliance on spot reserves from `UniswapV2Library.getAmountsIn()` will cause a loss of value for cross-chain users as an attacker will manipulate liquidity pools to skew price quotes and steal excess input tokens.



### Root Cause

In [`gatewayCrossChaine.sol:_swapAndSendERC20Tokens()`](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/d4834a468f7dad56b007b4450397289d4f767757/omni-chain-contracts/contracts/GatewayCrossChain.sol#L309), the contract uses `UniswapV2Library.getAmountsIn()` to estimate how many `targetZRC20` tokens are needed to obtain gasFee in `gasZRC20`. This quote is then used to swap tokens without checking for liquidity manipulation, allowing attackers to front-run the contract and change pool pricing.



### Internal Pre-conditions

.

### External Pre-conditions

.

### Attack Path

1. Attacker identifies a low-liquidity or custom pool for a token pair used in `getPathForTokens()`.
2. Attacker front-runs a cross-chain user operation, sending a transaction that manipulates the reserves in the pool (e.g via a flashloan ).
3. The contract executes `_swapAndSendERC20Tokens()` using skewed reserves from `getAmountsIn()`, miscalculating the required amount.
4. As a result, the attacker receives an overpayment or manipulates swap slippage to extract value.
5. After the swap completes, the attacker reverts the pool to its original state and profits from the imbalance.

### Impact

The cross-chain user suffers a loss in `targetZRC20` tokens, which are overpaid due to manipulated pricing.
The attacker gains the difference between the quoted and fair amount or causes a denial of service by making the swap fail (griefing).

### PoC

_No response_

### Mitigation

Enforce a hard slippage limit

---
