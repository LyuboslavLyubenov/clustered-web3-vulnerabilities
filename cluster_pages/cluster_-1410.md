# Cluster -1410

**Rank:** #129  
**Count:** 110  

## Label
Inaccurate decimal precision and flawed mathematical operations in financial calculations lead to incorrect token valuations, allocations, and conversions, compromising accuracy and trust in token pricing and user outcomes.

## Cluster Information
- **Total Findings:** 110

## Examples

### Example 1

**Auto Label:** Misaligned data representations and loss of precision in arithmetic operations lead to incorrect token allocations and financial misrepresentation, compromising accuracy and trust in distributed balances.  

**Original Text Preview:**

```solidity
function _processExternalTrades(
        BasketManagerStorage storage self,
        ExternalTrade[] calldata externalTrades
    )
        private
    {
        uint256 externalTradesLength = externalTrades.length;
        uint256[2][] memory claimedAmounts = _completeTokenSwap(self, externalTrades);
        // Update basketBalanceOf with amounts gained from swaps
        for (uint256 i = 0; i < externalTradesLength;) {
            ExternalTrade calldata trade = externalTrades[i];
            // nosemgrep: solidity.performance.array-length-outside-loop.array-length-outside-loop
            uint256 tradeOwnershipLength = trade.basketTradeOwnership.length;
            for (uint256 j; j < tradeOwnershipLength;) {
                BasketTradeOwnership calldata ownership = trade.basketTradeOwnership[j];
                address basket = ownership.basket;
                // Account for bought tokens
                self.basketBalanceOf[basket][trade.buyToken] +=
@>                    FixedPointMathLib.fullMulDiv(claimedAmounts[i][1], ownership.tradeOwnership, _WEIGHT_PRECISION);
                // Account for sold tokens
                self.basketBalanceOf[basket][trade.sellToken] = self.basketBalanceOf[basket][trade.sellToken]
@>                   + FixedPointMathLib.fullMulDiv(claimedAmounts[i][0], ownership.tradeOwnership, _WEIGHT_PRECISION)
@>                    - FixedPointMathLib.fullMulDiv(trade.sellAmount, ownership.tradeOwnership, _WEIGHT_PRECISION);
                unchecked {
                    // Overflow not possible: i is less than tradeOwnerShipLength.length
                    ++j;
                }
            }
            unchecked {
                // Overflow not possible: i is less than externalTradesLength.length
                ++i;
            }
        }
    }
```

Taking bought tokens as an example:

If tradeOwnershipLength is 2, claimedAmounts[i][1] is 111, basket0 has a tradeOwnership of 0.3e18, and basket1 has a tradeOwnership of 0.7e18, then using FixedPointMathLib.fullMulDiv() for calculation results in basket0 receiving 33 buyToken and basket1 receiving 77 buyToken. However, 33 + 77 = 110 < 111.

The same issue may also occur with sold tokens.

---
### Example 2

**Auto Label:** Decimal precision mismatches in token handling lead to incorrect value calculations, flawed transfers, and financial losses due to improper scaling and lack of consistent decimal normalization across functions and chains.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/665 

## Found by 
Kirkeelee, Sa1ntRobi, anchabadze, heavyw8t, yaioxy, zxriptor

### Summary

The lack of decimal normalization in cross-chain token transfers will cause users to lose significant funds, as borrowers will receive incorrect amounts when borrowing across chains with different token decimals.

### Root Cause

In [CrossChainRouter.sol:_handleBorrowCrossChainRequest()](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CrossChainRouter.sol#L581) and [CoreRouter.sol:borrowForCrossChain()](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/713372a1ccd8090ead836ca6b1acf92e97de4679/Lend-V2/src/LayerZero/CoreRouter.sol#L195C5-L205C6), the protocol assumes tokens have the same decimals across chains, but tokens like USDC have 6 decimals on Ethereum and 18 decimals on Base. The amount is passed directly through LayerZero without adjusting for these decimal differences, leading to incorrect borrow amounts.

### Internal Pre-conditions

1. User needs to call borrowCrossChain() with a token that has different decimals on source and destination chains
2. The token must be whitelisted and supported on both chains
3. User must have sufficient collateral on the source chain

### External Pre-conditions

Token decimals must differ between chains (e.g. USDC with 6 decimals on Ethereum and 18 decimals on Base)

### Attack Path

1. User calls borrowCrossChain() on Chain A to borrow 1000 USDC (6 decimals)
2. LayerZero delivers message to Chain B where USDC has 18 decimals
3. _handleBorrowCrossChainRequest() processes the amount directly
4. CoreRouter.borrowForCrossChain() transfers 1000 USDC (interpreted as 18 decimals)
5. User receives 1000e18 instead of 1000e6 USDC on Chain B

### Impact

The borrower receives 1000e18 instead of 1000e6 tokens, causing either:

- Massive overborrowing if decimals increase across chains
- Tiny underborrowing if decimals decrease across chains In both cases, users suffer significant financial losses.

### PoC

_No response_

### Mitigation

Add decimal normalization in `CrossChainRouter._handleBorrowCrossChainRequest()`

---
### Example 3

**Auto Label:** Inconsistent decimal precision handling leads to incorrect asset valuation and share calculations, causing potential asset loss and flawed conversion accuracy across tokens with varying decimal configurations.  

**Original Text Preview:**

The `Vault.sharePrice()` function incorrectly assumes 18 decimals when calculating share prices by using a hardcoded `1e18` value. This assumption breaks when the underlying asset has different decimals (e.g., `WBTC` with 8 decimals or the vault that introduces `_decimalsOffset()` > 0), leading to overestimation of the sharePrice and potential incorrect share price calculations that affect collateral valuations across the system.

```solidity
//File: src/core/Vault.sol

function sharePrice() external view returns (uint256) {
    return (convertToAssets(1e18) * assetPrice()) / 1e18;
}
```

```solidity
//File: src/core/Vault.sol -> ERC4626.sol

function decimals() public view virtual override(IERC20Metadata, ERC20) returns (uint8) {
    return _underlyingDecimals + _decimalsOffset();
}
```

For example, the scenario where the shares vault has not been 1:1 with the assets (eg., has some incentive donation), the process will take 1e18 shares for calculation which can be seen as 1e10 multiplied shares for WBTC vaults and as the rate has not been proportional to 1:1 there are potential incorrect precisions.

However, in combination with other calculation processes, the least precision potentially cancels out from the math, but if the `sharePrice()` itself still poses the over price value.

```bash
(convertToAssets(1e18) * assetPrice()) / 1e18
vault8 share price: 92344394845405751500000

decimals: 8 + 0 = 8
(convertToAssets(1e8) * assetPrice()) / 1e8 (maintain the returned 18 decimals)
vault8 share price after: 92344393931150300000000
```

Scale the share amount based on share decimals, this approach assume that `assetPrice()` is returned in 18 decimals:

```diff
function sharePrice() external view returns (uint256) {
-   return (convertToAssets(1e18) * assetPrice()) / 1e18;
+   uint256 oneShare = 10 ** decimals();
+   return (convertToAssets(oneShare) * assetPrice()) / oneShare;
}
```

---
