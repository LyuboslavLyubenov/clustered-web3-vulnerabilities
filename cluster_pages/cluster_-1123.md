# Cluster -1123

**Rank:** #260  
**Count:** 33  

## Label
Cross-chain trust vulnerabilities arising from insufficient validation of chain context, state mutations, and message integrity, enabling spoofing, fund redirection, and denial of service.

## Cluster Information
- **Total Findings:** 33

## Examples

### Example 1

**Auto Label:** **State manipulation and fee misconfiguration enabling unauthorized fee bypass or loss through improper arithmetic, validation, or logic flow.**  

**Original Text Preview:**

## Vulnerability Details
In `UpliftOnlyExample.sol`, when calculating swap fees in `onAfterSwap()`, the contract calls `getQuantAMMUpliftFeeTake()` instead of `getQuantAMMSwapFeeTake()`. This is incorrect because:

 Looking at the UpdateWeightRunner contract, there are two separate fee take percentages:
   - `quantAMMSwapFeeTake` - For swap fees (accessed via `getQuantAMMSwapFeeTake()`)
   - An uplift fee take intended for withdrawal uplift fees (accessed via `getQuantAMMUpliftFeeTake()`)

The context is clearly for swap fees since this occurs in onAfterSwap(), so it should have used getQuantAMMSwapFeeTake().
```solidity
function onAfterSwap(AfterSwapParams calldata params) // ...
// ... snip ...
// @audit should be getQuantAMMSwapFeeTake()
@>   uint256 quantAMMFeeTake = IUpdateWeightRunner(_updateWeightRunner).getQuantAMMUpliftFeeTake(); 
         uint256 ownerFee = hookFee;
// ...
```

## Impact
The bug causes the wrong fee percentage to be used for swap fee calculations. 

## Tools used
Manual Review

## Recommendations
Fix the function call in UpliftOnlyExample.sol in the `onAfterSwap` function:
```diff
- uint256 quantAMMFeeTake = IUpdateWeightRunner(_updateWeightRunner).getQuantAMMUpliftFeeTake();
+ uint256 quantAMMFeeTake = IUpdateWeightRunner(_updateWeightRunner).getQuantAMMSwapFeeTake(); 
```

---
### Example 2

**Auto Label:** **State manipulation and fee misconfiguration enabling unauthorized fee bypass or loss through improper arithmetic, validation, or logic flow.**  

**Original Text Preview:**

## Summary

> _**Note!:**_ This bug assumes that `upliftFeeBps` is applied in the upLifted value only as intended in the whitePaper and assumes the rounding down to 0 of `lpTokenDepositValueChange` is solved

The `UpliftOnlyExample` contract's uplift fee calculation can result in fees lower than the intended minimum when small uplifts occur, potentially enabling MEV attacks that were meant to be prevented by `minWithdrawalFeeBps`.

## Vulnerability Details

Current implementation uses an if/else block that chooses between fees types during liquidity removal:

```solidity
if (localData.lpTokenDepositValueChange > 0) {
    feePerLP = (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) / 10000;
} else {
    feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;
}
```

When calculating fees for uplifted positions:

```solidity
if (localData.lpTokenDepositValueChange > 0) {
    feePerLP = (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) / 10000;
}
```

Example scenario:

1. `minWithdrawalFeeBps` = 0.5% (50 bps)
2. `upliftFeeBps` = 50% (5000 bps)
3. MEV deposits 100e18
4. Gets 1% uplift (1e18)
5. Fee calculation: 1e18 \* 50% = 5e17 (0.05% of total deposit)
6. Actual fee (0.05%) < `minWithdrawalFeeBps` (0.5%)

This creates a gap where MEV can extract value while paying less than the intended minimum fee.

## Impact

The vulnerability enables:

1. MEV attacks with fees below intended minimum (for example, just in time liquidity for large swaps and feeless swaps attacks, etc)
2. Potential value extraction through rapid deposit/withdraw cycles attacks

## Tools Used

Manual  review

## Recommendations

add the `minWithdrawalFeeBps` to the `upliftFeeBps`

---
### Example 3

**Auto Label:** **State manipulation and fee misconfiguration enabling unauthorized fee bypass or loss through improper arithmetic, validation, or logic flow.**  

**Original Text Preview:**

## Summary

The `UpliftOnlyExample` contract's fee calculation mechanism can be bypassed for significant amounts when using low-decimal tokens like `GUSD` (2 decimals). This allows users to execute large trades without paying any fees, leading to potential revenue loss for the protocol and the pool creator.

## Vulnerability Details

**NOTE:** As per the protocol readme: all balancer tokens are in scope and `GUSD` is a valid balancer token.

The fee calculation in `UpliftOnlyExample::afterSwap` hook uses the following formula:\
[UpliftOnlyExample.sol#L298](https://github.com/Cyfrin/2024-12-quantamm/blob/a775db4273eb36e7b4536c5b60207c9f17541b92/pkg/pool-hooks/contracts/hooks-quantamm/UpliftOnlyExample.sol#L298)

```js
uint256 hookFee = params.amountCalculatedRaw.mulUp(hookSwapFeePercentage);
```

With 0.1% fee setting:

```js
hookFee = amountCalculatedRaw * 1e15 / 1e18
        = amountCalculatedRaw / 1000
```

For zero fee (rounding down in integer arithmetic):

```js
amountCalculatedRaw / 1000 < 1
amountCalculatedRaw < 1000
```

For `GUSD` (2 decimals):

1000 base units = 10.00 GUSD\
Therefore, any amount up to 9.99 GUSD will result in zero fees.

Concrete Example:

1. Trade amount: 9.99 GUSD = 999 base units
2. Fee calculation with 0.1% fee:

```js
999 * 1e15 / 1e18 = 0.999 (rounds down to 0)
```

## Impact

Swap fee can be bypassed for low-decimal tokens

## Tools Used

Manual Review

## Recommendations

Use decimal normalization for fee calculations

---
