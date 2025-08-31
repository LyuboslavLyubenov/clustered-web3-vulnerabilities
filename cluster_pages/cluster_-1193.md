# Cluster -1193

**Rank:** #132  
**Count:** 107  

## Label
Decimal precision mismatches in token handling lead to incorrect value calculations, flawed transfers, and financial losses due to improper scaling and lack of consistent decimal normalization across functions and chains.

## Cluster Information
- **Total Findings:** 107

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

**Auto Label:** Precision loss in integer arithmetic due to improper scaling, overflow, or truncation leads to incorrect token calculations, enabling unauthorized claims, distribution manipulation, or denial-of-service attacks.  

**Original Text Preview:**

```python
SHARES * 1e16 // DENOM
9999999.0
SHARES * 1e14 // DENOM
99999.0
SHARES * 1e13 // DENOM
9999.0
SHARES * 1e9 // DENOM
1.0
SHARES * 1e8 // DENOM
0.0
SHARES * 1e10 // DENOM
10.0
SHARES * 2e8 // DENOM
0.0
SHARES * 9e8 // DENOM
0.0
SHARES * 1.9e8 // DENOM
0.0
SHARES * 1.9e9 // DENOM
1.0
0.06 * 1e9 / 1e18
6e-11
0.06 * 1e9 / 1e18 * 3463
2.0778e-07
0.06 * 1e9 / 1e18 * 3463 * 300_000
0.062334
0.06 * 1e9 / 1e18 * 3463 * 300_00
0.0062334
0.06 * 1e9 / 1e18 * 3463 * 3000
0.00062334
1e9 / 1e18 * 3463
3.463e-06
```

---
