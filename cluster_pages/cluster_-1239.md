# Cluster -1239

**Rank:** #363  
**Count:** 16  

## Label
Incorrect ratio or liquidity calculations due to flawed logic, improper state usage, or bypassing composability, leading to erroneous caps, false quotes, or invalid allocations with potential financial loss.

## Cluster Information
- **Total Findings:** 16

## Examples

### Example 1

**Auto Label:** Incorrect ratio or liquidity calculations due to flawed logic, improper state usage, or bypassing composability, leading to erroneous caps, false quotes, or invalid allocations with potential financial loss.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The `quotePotentialWithdraw()` and `quotePotentialSwap()` functions in the `VariantPool` and `StablePool` contracts are designed to provide estimates for withdrawal and swap operations without executing the actual transactions. However, unlike their actual execution counterparts (`withdraw()` and `swap()`), these quote functions do not verify if the post-operation coverage ratio meets the minimum threshold requirements.

In the actual execution functions:

- `withdraw()` verifies that after withdrawal, the coverage ratio is above `_rThreshold` (0.25e18 by default)
- `swap()` checks that after the swap, the coverage ratio of `toAsset` is above `_rThreshold`
- `withdrawFromOtherAsset()` verifies that after withdrawal, the coverage ratio is above `ETH_UNIT` (1e18)

Similar coverage ratio check is present in `quotePotentialWithdrawFromOtherAsset()`, but missing in `quotePotentialWithdraw()` and `quotePotentialSwap()`:

```solidity
function quotePotentialWithdraw(address token, uint256 liquidity)
    external
    view
    whenNotPaused
    returns (uint256 amount, uint256 fee)
{
    if (liquidity == 0) revert Pool_InputAmountZero();
    VariantAsset asset = _assetOf(token);
    uint256 amountInWad;
    uint256 feeInWad;
    (amountInWad,, feeInWad) = _quoteWithdraw(asset, liquidity);
    amount = _fromWad(amountInWad, asset.underlyingTokenDecimals());
    fee = _fromWad(feeInWad, asset.underlyingTokenDecimals());
    // Missing coverage ratio check
}
```

This inconsistency can lead to misleading quotes for users, who might receive quotes for operations that would actually revert when executed due to an insufficient coverage ratio.

## Location of Affected Code

File: [src/pool/VariantPool.sol](https://github.com/MobiusExchange/core/blob/21eeeca84ea26abfb131f758164256af8a706aaa/src/pool/VariantPool.sol)

File: [src/pool/StablePool.sol](https://github.com/MobiusExchange/core/blob/21eeeca84ea26abfb131f758164256af8a706aaa/src/pool/StablePool.sol)

## Impact

This inconsistency can lead to misleading quotes for users, who might receive quotes for operations that would actually revert when executed due to an insufficient coverage ratio.

## Recommendation

Add the missing coverage ratio checks to both quote functions

## Team Response

Fixed.

## [I-01] Integer Underflow in Haircut Calculation for High Decimal Tokens

## Severity

Informational

## Description

In `MobiusRouter`, there is an integer underflow vulnerability in the haircut calculation when tokens with more than 18 decimals are used. The issue occurs in the `_swap()` function where the haircut is calculated:

```solidity
haircut += localHaircut * 10 ** (18 - IERC20Metadata(tokenPath[i + 1]).decimals());
```

When a token's decimal value exceeds 18, the expression `(18 - decimals)` becomes negative, causing an underflow in the exponentiation operation, which will revert the transaction.

## Location of Affected Code

File: [src/router/MobiusRouter.sol](https://github.com/MobiusExchange/core/blob/21eeeca84ea26abfb131f758164256af8a706aaa/src/router/MobiusRouter.sol)

## Impact

The `MobiusRouter.swapTokensForTokens()` does not work for tokens with decimals larger than 18.

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.2 <0.9.0;

contract UnderFlow {

    function checkUnderFlow(uint256 decimal) public returns (uint256){
        uint256 haircut = 1e18;
        uint256 localHaircut = 1e18;
        haircut += localHaircut * 10 ** (18-decimal);
        return haircut;
    }
}
```

Deploy this contract in Remix, when `decimal==17`, it can run successfully, when `decimal == 19`, the tx will revert.

<img width="1044" alt="Image" src="https://github.com/user-attachments/assets/84194ff3-3056-41cc-9ae0-097b34437e14" />

## Recommendation

Modify the haircut calculation to handle tokens with more than 18 decimals:

```solidity
if (IERC20Metadata(tokenPath[i + 1]).decimals() <= 18) {
    haircut += localHaircut * 10 ** (18 - IERC20Metadata(tokenPath[i + 1]).decimals());
} else {
    haircut += localHaircut / 10 ** (IERC20Metadata(tokenPath[i + 1]).decimals() - 18);
}
```

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Inaccurate state calculations due to flawed balance aggregation and missing external price or interest validation, leading to misleading liquidity estimates and unauthorized pool manipulation.  

**Original Text Preview:**

## Severity: Low Risk

## Context
AaveV3Farm.sol#L69-L72

## Description
In the `AaveV3.ValidationLogic` contract, the maximum deposit calculation considers the `accruedToTreasury` variable when checking against `supplyCap`:

```solidity
require(
    supplyCap == 0 ||
    ((IAToken(reserveCache.aTokenAddress).scaledTotalSupply() +
    uint256(reserve.accruedToTreasury)).rayMul(reserveCache.nextLiquidityIndex) + amount) >=
    supplyCap * (10 ** reserveCache.reserveConfiguration.getDecimals()),
    Errors.SUPPLY_CAP_EXCEEDED
);
```

However, the `AaveV3Farm` contract calculates the maximum deposit using only `aToken.totalSupply()`, while it should also consider the amount accrued to treasury. By returning an incorrect maximum deposit, it could revert when trying.

## Recommendation
Consider changing the code to the following:

```solidity
uint256 currentUnderlyingProtocolSupply = ERC20(aToken).totalSupply();
return supplyCapInAssetTokenDecimals - currentUnderlyingProtocolSupply;
```

Change to:

```solidity
(, uint256 _accruedToTreasuryScaled, uint256 _totalAToken, , , , , , , , ,) =
IAaveDataProvider(dataProvider).getReserveData(address(asset));

// Supply cap already reached
if (_totalAToken + _accruedToTreasuryScaled >= _supplyCap) {
    return 0;
}
return supplyCapInAssetTokenDecimals - (_totalAToken + _accruedToTreasuryScaled);
```

## Additional Notes
- **infiniFi**: Fixed in `0b20a51`.
- **Spearbit**: Fix verified.

---
### Example 3

**Auto Label:** Inaccurate state calculations due to flawed balance aggregation and missing external price or interest validation, leading to misleading liquidity estimates and unauthorized pool manipulation.  

**Original Text Preview:**

**Impact**

`addLiquidity` looks as follows

https://github.com/blkswnStudio/ap/blob/8fab2b32b4f55efd92819bd1d0da9bed4b339e87/packages/contracts/contracts/SwapOperations.sol#L228-L246

```solidity
  function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    PriceUpdateAndMintMeta memory _priceAndMintMeta,
    uint deadline
  ) public payable virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
    ProvidingVars memory vars;
    vars.pair = getPair[tokenA][tokenB]; /// @audit QA not sorting tokens means this can revert | No fix is ok but it's a gotcha
    if (vars.pair == address(0)) revert PairDoesNotExist();
    /// @audit not checking that min < desired - S
    {
      (vars.reserveA, vars.reserveB) = getReserves(tokenA, tokenB);
      if (vars.reserveA == 0 && vars.reserveB == 0) {
        (amountA, amountB) = (amountADesired, amountBDesired); /// @audit First LP Can attack by massively imbalancing by performing a single sided swap or similar
      } else {
```

When no liquidity was previously added, the ratio for the LP will be taken at face value

This could be used by the first depositor to purposefully imbalance the pool, as a means to take on debts that either allow them to be closer to delta neutral, or as a means to grief other users

**Mitigation**

Consider using a ratio that is taken from the oracles

---
