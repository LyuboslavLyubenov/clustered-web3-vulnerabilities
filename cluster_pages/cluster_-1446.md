# Cluster -1446

**Rank:** #86  
**Count:** 200  

## Label
Inadequate validation of position size changes leads to incorrect state transitions, unauthorized position adjustments, and increased liquidation risk through flawed arithmetic or logic checks.

## Cluster Information
- **Total Findings:** 200

## Examples

### Example 1

**Auto Label:** Insufficient validation and flawed state checks lead to unauthorized access, incorrect liquidation logic, and unbounded debt accumulation under edge conditions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/905 

## Found by 
Brene, Tigerfrake, Uddercover, aman, dreamcoder, dystopia, future, kom, mahdifa, rokinot, t.aksoy, xiaoming90, zraxx

## Title 
Liquidators Must Supply Collateral Asset Before Redeeming Seized Rewards

## Summary
Liquidators can initiate liquidations even if they haven’t previously supplied the asset being liquidated. However, during this process, although their `totalInvestment` is updated based on the difference between `seizeTokens` and `currentReward`, the system does not register the asset as part of their `suppliedAssets`. This omission causes the liquidator’s collateral balance for the asset to remain at zero.
Consequently, when the liquidator tries to redeem the seized collateral, the protocol treats them as having no supplied collateral, causing a failed liquidity check. To redeem, the liquidator is forced to make a fresh supply of the collateral asset, just so the protocol recognizes it as one of their supplied tokens.

## Root Cause
In [`liquidateSeizeUpdate`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L278-L318) function, the `updateTotalInvestment()` function updates the `totalInvestment` mapping for both borrower and liquidator:

```solidity
        // Update total investment
        lendStorage.updateTotalInvestment(
            borrower, lTokenCollateral, lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens
        );

>>      lendStorage.updateTotalInvestment(
            sender,
            lTokenCollateral,
            lendStorage.totalInvestment(sender, lTokenCollateral) + (seizeTokens - currentReward)
        );
```

However, unlike borrowers who already have the asset recorded in their supplied list, the liquidator’s `userSuppliedAssets` array remains empty since no action explicitly registers the new asset. This becomes problematic during redemption, which relies on `userSuppliedAssets` to determine collateral value. The following explain this;

A user calls the `redeem` function to redeem his investments from the seized tokens, the [`getHypotheticalAccountLiquidityCollateral()`](https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L385-L467) is called to return both `borrowed` and `collateral`

```solidity
        // @audit These arrays are empty so far
>>      address[] memory suppliedAssets = userSuppliedAssets[account].values();
>>      address[] memory borrowedAssets = userBorrowedAssets[account].values();

        // First loop: Calculate collateral value from supplied assets
>>      // @audit Will not loop as there is nothing in the array
        for (uint256 i = 0; i < suppliedAssets.length;) {
            ---snip---

            // Add to collateral sum
            vars.sumCollateral =
                mul_ScalarTruncateAddUInt(vars.tokensToDenom, lTokenBalanceInternal, vars.sumCollateral);

            unchecked {
                ++i;
            }
        }

        // Second loop: Calculate borrow value from borrowed assets
        // @audit Will not loop as there is nothing in the array
        for (uint256 i = 0; i < borrowedAssets.length;) {
           ---snip---

            // Add to borrow sum
            vars.sumBorrowPlusEffects =
                mul_ScalarTruncateAddUInt(vars.oraclePrice, totalBorrow, vars.sumBorrowPlusEffects);

            unchecked {
                ++i;
            }
        }

        // Handle effects of current action
>>      // @audit Since address(lTokenModify) != address(0), and `redeemTokens > 0`,
        if (address(lTokenModify) != address(0)) {
            vars.oraclePriceMantissa = UniswapAnchoredViewInterface(priceOracle).getUnderlyingPrice(lTokenModify);
            vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});

            // Add effect of redeeming collateral
            if (redeemTokens > 0) {
                vars.collateralFactor = Exp({
                    mantissa: LendtrollerInterfaceV2(lendtroller).getCollateralFactorMantissa(address(lTokenModify))
                });
                vars.exchangeRate = Exp({mantissa: lTokenModify.exchangeRateStored()});
                vars.tokensToDenom = mul_(mul_(vars.collateralFactor, vars.exchangeRate), vars.oraclePrice);
>>              vars.sumBorrowPlusEffects =
                    mul_ScalarTruncateAddUInt(vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects);
            }

            // Add effect of new borrow
            if (borrowAmount > 0) {
                vars.sumBorrowPlusEffects =
                    mul_ScalarTruncateAddUInt(vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects);
            }
        }

>>      return (vars.sumBorrowPlusEffects, vars.sumCollateral);
```
Then in `redeem()`, the following check is made:
```solidity
    require(collateral >= borrowed, "Insufficient liquidity");
```
But since `collateral == 0` and `borrowed > 0`, this reverts.

## Internal Pre-conditions

1. The liquidator had never supplied the asset being seized before initiating the liquidation.
2. The redeem logic depends on the `userSuppliedAssets` array to calculate liquidity.
3. Collateral balance is calculated to be zero, leading to a failed redeem.

## External Pre-conditions
None

## Attack Path
1. A user liquidates another user’s position and receives seized collateral.
2. Since the liquidator hasn’t supplied the collateral asset before, it isn't included in their `userSuppliedAssets`.
3. When attempting to redeem, the liquidity check fails as the protocol calculates collateral as zero.
4. The liquidator cannot redeem unless they first supply the seized token themselves.

## Impact
Liquidators encounter a barrier when trying to redeem the collateral they rightfully acquired. Without manually supplying the asset first, they are unable to redeem. This creates unnecessary friction and traps seized rewards unless the liquidator takes an extra, unintuitive step of supplying the token they already received.

## POC
1. In `test/TestLiquidations.t.sol`, add the following test:
```solidity
    function test_liquidator_must_supply_tokens_to_redeem_reward(uint256 supplyAmount, uint256 borrowAmount, uint256 newPrice) public {
        // Bound inputs to reasonable values
        supplyAmount = bound(supplyAmount, 100e18, 1000e18);
        borrowAmount = bound(borrowAmount, 50e18, supplyAmount * 60 / 100); // Max 60% LTV
        newPrice = bound(newPrice, 1e16, 5e16); // 1-5% of original price

        // Supply token0 as collateral on Chain A
        (address tokenA, address lTokenA) = _supplyA(deployer, supplyAmount, 0);

        // Supply token1 as liquidity on Chain A from random wallet
        (address tokenB, address lTokenB) = _supplyA(address(1), supplyAmount, 1);

        vm.prank(deployer);
        coreRouterA.borrow(borrowAmount, tokenB);

        // Simulate price drop of collateral (tokenA) only on first chain
        priceOracleA.setDirectPrice(tokenA, newPrice);
        // Simulate price drop of collateral (tokenA) only on second chain
        priceOracleB.setDirectPrice(
            lendStorageB.lTokenToUnderlying(lendStorageB.crossChainLTokenMap(lTokenA, block.chainid)), newPrice
        );

        // Attempt liquidation
        vm.startPrank(liquidator);
        ERC20Mock(tokenB).mint(liquidator, borrowAmount);
        IERC20(tokenB).approve(address(coreRouterA), borrowAmount);

        // Expect LiquidateBorrow event
        vm.expectEmit(true, true, true, true);
        emit LiquidateBorrow(liquidator, lTokenB, deployer, lTokenA);

        // Repay 1% of the borrow
        uint256 repayAmount = borrowAmount / 100;

        coreRouterA.liquidateBorrow(deployer, repayAmount, lTokenA, tokenB);
        vm.stopPrank();

        // Verify liquidation was successful
        assertLt(
            lendStorageA.borrowWithInterestSame(deployer, lTokenB),
            borrowAmount,
            "Borrow should be reduced after liquidation"
        );
        
        // retrieve liquidators investment in protocol from seized collateral
        uint256 liquidatorSeizureRewards = lendStorageA.totalInvestment(liquidator, lTokenA);

        // assert that is greater than 0 from liquidation
        assertGt(liquidatorSeizureRewards, 0, "Liquidator should have some seizure rewards");

        // get liquidator supply assets
        address[] memory liquidatorSupplyAssets = lendStorageA.getUserSuppliedAssets(liquidator);

        // should show that no asset was added for him
        assertEq(liquidatorSupplyAssets.length, 0, "Liquidator should have no supply assets");
        
        // Check liquidity for liquidator
        (uint256 borrowed, uint256 collateral) =
            lendStorageA.getHypotheticalAccountLiquidityCollateral(
                liquidator,
                LToken(lTokenA),
                liquidatorSeizureRewards,
                0
            );
        // should show that collateral is 0  and borrowed is positive from Adding effect of 
        // redeemTokens i.e liquidatorSeizureRewards
        assertEq(collateral, 0);
        assertGt(borrowed, 0);
        
        // liquidator attempts to redeem his seized collateral
        vm.startPrank(liquidator);
        // expect revert
        vm.expectRevert(bytes("Insufficient liquidity"));
        coreRouterA.redeem(liquidatorSeizureRewards, payable(lTokenA));

        vm.stopPrank();

        // Now liquidator has to supply some tokens to redeem his reward
        // Supply token0 as collateral on Chain A
        _supplyA(liquidator, supplyAmount, 0);

        // retrieve liquidators current investment
        uint256 liquidatorCurrentInvestment = lendStorageA.totalInvestment(liquidator, lTokenA);

        // assert current investemnt is greater than seized reward
        assertGt(liquidatorCurrentInvestment, liquidatorSeizureRewards);

        // liquidator now attempts to redeem
        vm.prank(liquidator);
        coreRouterA.redeem(liquidatorCurrentInvestment, payable(lTokenA));

        // Now assert that liquidator has 0 investment in protocol
        assertEq(lendStorageA.totalInvestment(liquidator, lTokenA), 0); 
    }
```
2. Run: `forge test --mt test_liquidator_must_supply_tokens_to_redeem_reward -vvvv`


## Mitigation
Add this asset to the liquidator:
```diff
        // Update total investment
        lendStorage.updateTotalInvestment(
            borrower, lTokenCollateral, lendStorage.totalInvestment(borrower, lTokenCollateral) - seizeTokens
        );

+       lendStorage.addUserSuppliedAsset(sender, lTokenCollateral);

>>      lendStorage.updateTotalInvestment(
            sender,
            lTokenCollateral,
            lendStorage.totalInvestment(sender, lTokenCollateral) + (seizeTokens - currentReward)
        );
```

---
### Example 2

**Auto Label:** Insufficient validation and flawed state checks lead to unauthorized access, incorrect liquidation logic, and unbounded debt accumulation under edge conditions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/864 

## Found by 
future, h2134, jokr, mussucal, rudhra1749

### Summary

During cross-chain borrows, the destination chain is expected to verify that the `payload.collateral` provided by the source chain covers **only** the cross-chain borrow amount being originated from that source chain. However, the current implementation incorrectly checks that the `payload.collateral` covers all of the user’s **same-chain borrows of destination chain and cross-chain borrows originated from the destination chain**, which is invalid logic.


```solidity
(uint256 totalBorrowed,) = lendStorage.getHypotheticalAccountLiquidityCollateral(
    payload.sender,
    LToken(payable(payload.destlToken)),
    0,
    payload.amount
);

require(payload.collateral >= totalBorrowed, "Insufficient collateral");
```
https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L616-L622

This causes the borrow requests to fail even if the user has sufficient collateral on the **source** chain to support the **new** cross-chain borrow.

### Root Cause

The destination chain incorrectly validates the borrow by checking whether `payload.collateral` can cover:

- Same-chain borrows on the destination chain  
- Cross-chain borrows originated **from** the destination chain  

This is incorrect because the collateral being validated (`payload.collateral`) resides on the **source** chain, not the destination chain. It should only be checked against the cross-chain borrows taken from the source chain.

### Internal Preconditions

None

### External Preconditions



### Impact

Valid cross-chain borrow requests are incorrectly rejected.  



### PoC

_No response provided._

### Mitigation

Update the destination chain logic to validate only that the `payload.collateral` covers the borrow being initiated from the source chain and previous borrows from source chain. Avoid  including:

- Same-chain borrows on the destination chain  
- Other cross-chain borrows **originating** from the destination chain

---
### Example 3

**Auto Label:** Insufficient validation and flawed state checks lead to unauthorized access, incorrect liquidation logic, and unbounded debt accumulation under edge conditions.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/711 

## Found by 
0xc0ffEE, xiaoming90

### Summary

-

### Root Cause

-

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

The protocol's design or intention is to allow Bob to deposit collateral on Chain A, and then let him use that collateral in Chain A to borrow assets from Chain B.

Assume that Bob supplied 10000 USDC as collateral in Chain A. He calls `borrowCrossChain()` on Chain A to borrow from Chain B. When the LayerZero delivers the message to Chain B, Chain B's `CrossChainRouter._handleBorrowCrossChainRequest()` -> `CoreRouter.borrowForCrossChain()` -> `LErc20.borrow()` will be executed.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CrossChainRouter.sol#L581

```solidity
File: CrossChainRouter.sol
581:     function _handleBorrowCrossChainRequest(LZPayload memory payload, uint32 srcEid) private {
..SNIP..
624:         // Execute the borrow on destination chain
625:         CoreRouter(coreRouter).borrowForCrossChain(payload.sender, payload.amount, payload.destlToken, destUnderlying);
626: 
```

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/CoreRouter.sol#L195

```solidity
File: CoreRouter.sol
195:     function borrowForCrossChain(address _borrower, uint256 _amount, address _destlToken, address _destUnderlying)
196:         external
197:     {
198:         require(crossChainRouter != address(0), "CrossChainRouter not set");
199: 
200:         require(msg.sender == crossChainRouter, "Access Denied");
201: 
202:         require(LErc20Interface(_destlToken).borrow(_amount) == 0, "Borrow failed"); // <= Revert!
203: 
204:         IERC20(_destUnderlying).transfer(_borrower, _amount);
205:     }
```

Inside the internal code of `LErc20.borrow()`, it will call `borrowFresh()` -> `Lendtroller.getHypotheticalAccountLiquidityInternal()`

The problem or issue here is that Chain B's `Lendtroller.getHypotheticalAccountLiquidityInternal()` function only takes into account the collateral of Chain B. Also, note that `Lendtroller.getHypotheticalAccountLiquidityInternal()` is the original Compound V2 function. Thus, the borrowing will eventually revert due to insufficient collateral because it cannot see Bob's Chain A collateral.

### Impact

Core protocol functionality is broken

### PoC

_No response_

### Mitigation

_No response_

---
