# Cluster -1418

**Rank:** #294  
**Count:** 26  

## Label
Precision loss in financial calculations due to improper arithmetic ordering and rounding, leading to under-delivery of assets, false validations, and exploitable mispricings.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

**Auto Label:** Precision loss from improper arithmetic ordering in price and bid calculations, leading to rounding errors, bid manipulation, and inaccurate state updates.  

**Original Text Preview:**

The `Folio.getBid()` function is used to retrieve auction bid parameters (sellAmount, bidAmount, and price) for an ongoing auction at a specified `timestamp`. However, while the auction price is correctly calculated based on the provided timestamp, the `sellAmount` and `bidAmount` calculations rely on the current `totalSupply()`.
Since `totalSupply()` can change over time (due to minting, fee inflation, etc.), **using the current `totalSupply()` when the requested timestamp is not the current block** leads to incorrect and inconsistent bid parameters, which makes it unsafe to rely on `getBid()` for off-chain simulation or for use cases that need historical or future bid estimates.

```solidity
    function getBid(
        uint256 auctionId,
        IERC20 sellToken,
        IERC20 buyToken,
        uint256 timestamp,
        uint256 maxSellAmount
    ) external view returns (uint256 sellAmount, uint256 bidAmount, uint256 price) {
        return
            _getBid(
                auctions[auctionId],
                sellToken,
                buyToken,
                totalSupply(), <@ should be at the snapshot supply at timestamp
                timestamp != 0 ? timestamp : block.timestamp,
                0,
                maxSellAmount,
                type(uint256).max
            );
    }
```

**Recommendation:**

Introduce a mechanism to snapshot or track `totalSupply` over time and use the value corresponding to the requested `timestamp` when computing `sellAmount` and `bidAmount`.
Alternatively, document clearly that `getBid()` should only be used with `timestamp == block.timestamp` to avoid incorrect assumptions.

---
### Example 2

**Auto Label:** Precision loss in financial calculations due to improper arithmetic ordering and rounding, leading to under-delivery of assets, false validations, and exploitable mispricings.  

**Original Text Preview:**

## Severity: Low Risk

## Context
(No context files were provided by the reviewer)

## Description
In `subscribeRebasingOUSG()`, the minimum amount check is performed before several decimal conversions and share calculations, which can lead to precision loss. The actual rOUSG amount received may be less than the specified `minimumRwaReceived` due to rounding in `getSharesByROUSG()` and the OUSG to rOUSG share conversion.

## Impact
Users may receive less rOUSG than their specified minimum, effectively bypassing their slippage protection.

## Proof of Concept
```solidity
function test_subscribeRebasingOUSGMRealPrices() public {
    uint256 aliceDepositAmount = 100_000e6;
    deal(address(USDC), address(alice), aliceDepositAmount);
    vm.startPrank(OUSG_GUARDIAN);
    // use live price
    ousgOracleWrapper.setRwaOracle(0x0502c5ae08E7CD64fe1AEDA7D6e229413eCC6abe);
    // set ROUSG oracle to live price
    rOUSGToken.setOracle(0x0502c5ae08E7CD64fe1AEDA7D6e229413eCC6abe);
    vm.startPrank(alice);
    USDC.approve(address(ousgXManager), aliceDepositAmount);
    uint256 rousgAmountOut = ousgXManager.subscribeRebasingOUSG(
        address(USDC),
        aliceDepositAmount,
        1000e18
    );
    // assert minimum amount received
    assertGe(1000e18, rousgAmountOut); // this will revert
}
// The POC reverts with a failing assertion. `[FAIL: assertion failed: 1000000000000000000000 < 99999999999999999999903]`, !
```

## Recommendation
Consider adding a direct slippage check at the end against `minimumRwaReceived` (which should be renamed to `minimumROUSGReceived` for clarity).
```solidity
if (rousgAmountOut < minimumRwaReceived) revert RwaReceiveAmountTooSmall();
```

## Ondo Finance
Fixed in commit `bc1c3b14`.

## Spearbit
Fix verified.

---
### Example 3

**Auto Label:** Underflow errors due to inaccurate value tracking during price fluctuations, leading to invalid liquidation decisions, incorrect fee calculations, and failed withdrawals.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-11-autonomint-judging/issues/840 

## Found by 
0x37, 0x73696d616f, Cybrid, John44, LordAlive, Ocean\_Sky, ParthMandale, Silvermist, futureHack, nuthan2x, t.aksoy

### Summary

An underflow in [cds profits computation](https://github.com/sherlock-audit/2024-11-autonomint/blob/main/Blockchain/Blockchian/contracts/Core_logic/borrowLiquidation.sol#L211-L212) can cause liquidation type 1 to revert.

### Root Cause

Here is the formula for cds profits computation. It is possible that **returnToTreasury** (aka as borrowerDebt) can be greater than **deposited amount value** (((depositDetail.depositedAmountInETH * depositDetail.ethPriceAtDeposit) / BorrowLib.USDA_PRECISION) / 100)  which can cause the formula to revert and DOS the liquidation type 1 process.

```Solidity
File: borrowLiquidation.sol
211:         // Calculate the CDS profits, USDA precision is 1e12, @audit need to double check if profit formula is right logic or can underflow
212:         uint128 cdsProfits = (((depositDetail.depositedAmountInETH * depositDetail.ethPriceAtDeposit) / BorrowLib.USDA_PRECISION) / 100) - returnToTreasury - returnToAbond;
```

Here is the details of returntoTreasury which is equivalent to borrower's debt amount during time of liquidation. This includes interest accumulated from loan creation up to liquidation time. If lastCumulativeRate reach a certain percentage like 126% in my example issue, borrowerDebt will become greater than **deposited amount value**  ((depositDetail.depositedAmountInETH * depositDetail.ethPriceAtDeposit) / BorrowLib.USDA_PRECISION) / 100) which can cause underflow error.

```Solidity
File: borrowLiquidation.sol
204:         // Calculate borrower's debt, rate precision is 1e27, this includes interest accumulated from creation of loan to liquidation
205:         uint256 borrowerDebt = ((depositDetail.normalizedAmount * lastCumulativeRate) / BorrowLib.RATE_PRECISION);
206:         uint128 returnToTreasury = uint128(borrowerDebt);
```

Here is the details of returnToAbond which represents 10% of deposited amount after subtracting returntotreasury.
```Solidity
File: borrowLiquidation.sol
209:         uint128 returnToAbond = BorrowLib.calculateReturnToAbond(depositDetail.depositedAmountInETH, depositDetail.ethPriceAtDeposit, returnToTreasury);
```
```Solidity
File: BorrowLib.sol
137:     function calculateReturnToAbond(
138:         uint128 depositedAmount,
139:         uint128 depositEthPrice,
140:         uint128 returnToTreasury
141:     ) public pure returns (uint128) {
142:         // 10% of the remaining amount, USDA precision is 1e12
143:         return (((((depositedAmount * depositEthPrice) / USDA_PRECISION) / 100) - returnToTreasury) * 10) / 100;
144:     }
```
Regarding the lastcumulativerate on how it exactly increases, it can be computed by the following below. In line 245, there is [_rpow](https://github.com/sherlock-audit/2024-11-autonomint/blob/main/Blockchain/Blockchian/contracts/lib/BorrowLib.sol#L1038-L1056) computation in which time interval, interest rate per second are factors in increasing the last cumulative rate. The more time the loan is not liquidated or not paid, the more the loan get bigger due to interest rate which will eventually reach 126%. The cumulative rate variable should start or set at around 100% based on set APR after protocol deployed the borrowing contract, so the very first loan will use the cumulative rate at this rate, then will increase from there moving forward.

```Solidity
File: BorrowLib.sol
222:     /**
223:      * @dev calculates cumulative rate
224:      * @param noOfBorrowers total number of borrowers in the protocol
225:      * @param ratePerSec interest rate per second
226:      * @param lastEventTime last event timestamp
227:      * @param lastCumulativeRate previous cumulative rate
228:      */
229:     function calculateCumulativeRate(
230:         uint128 noOfBorrowers,
231:         uint256 ratePerSec,
232:         uint128 lastEventTime,
233:         uint256 lastCumulativeRate
234:     ) public view returns (uint256) {
235:         uint256 currentCumulativeRate;
236: 
237:         // If there is no borrowers in the protocol
238:         if (noOfBorrowers == 0) {
239:             // current cumulative rate is same as ratePeSec
240:             currentCumulativeRate = ratePerSec;
241:         } else {
242:             // Find time interval between last event and now
243:             uint256 timeInterval = uint128(block.timestamp) - lastEventTime;
244:             //calculate cumulative rate
245:             currentCumulativeRate = lastCumulativeRate * _rpow(ratePerSec, timeInterval, RATE_PRECISION);
246:             currentCumulativeRate = currentCumulativeRate / RATE_PRECISION;
247:         }
248:         return currentCumulativeRate;
249:     }
```


Let's further discuss the exact situation on the attack path to describe the issue on more details.


### Internal pre-conditions

If the cumulative rate exceed certain percentage like 125% in my sample issue, it will cause revert to the cds profits computation.

### External pre-conditions

_No response_

### Attack Path

Let's consider this scenario wherein the 

**Borrower deposit details at time of loan creation:**
Deposited amount in eth  = 50 eth
Ether price at deposit time = 3000 usd price of 1 Eth per oracle during deposit

**During time of liquidation**
lastCumulativerate is 126% or 1.26e27

1. Let's compute first the returnToTreasury or borrower's debt during time of liquidation. In this case, we use 126% to represent the borrower's debt value in percentage. It increased to that figure due to interest accumulated since loan creation up to liquidation. The amount of returnToTreasury here is 1.512e11 or 151,200 usd
<img width="915" alt="image" src="https://sherlock-files.ams3.digitaloceanspaces.com/gh-images/9b515d43-ea6b-41b0-9a86-d5bdf01005db" />



2. Let's compute the returnToabond as part also of formula, in this case, we already encountered a problem of underflow error due to subtraction of larger returnToTreasury. See below 
<img width="841" alt="image" src="https://sherlock-files.ams3.digitaloceanspaces.com/gh-images/9a02f62c-0dc5-481b-a34b-182f8a5e33ec" />

3. As you can see in this CDS profit computation below, there is already underflow error when the bigger returnToTreasury is subtracted to deposited amount value.
<img width="1153" alt="image" src="https://sherlock-files.ams3.digitaloceanspaces.com/gh-images/c1c8fafd-a1d8-4fc8-bb64-5da4de8a27f5" />


Here is the link of google sheet for [computation copy](https://docs.google.com/spreadsheets/d/1Cmek971NLPn-PoYOFyPdtvBWqUEE11kHzgdkAg_vXts/edit?usp=drive_link)


### Impact

Interruption of liquidation type 1 process. The loan is not liquidatable.

### PoC

See attack path

### Mitigation

Revise the computation of cds profits, it should not revert in any way possible so the liquidation won't be interrupted.

---
