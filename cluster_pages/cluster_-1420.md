# Cluster -1420

**Rank:** #143  
**Count:** 99  

## Label
Inaccurate state updates due to improper edge case handling, leading to fund locking, incorrect matching, or erroneous token transfers and enabling denial-of-service or financial loss.

## Cluster Information
- **Total Findings:** 99

## Examples

### Example 1

**Auto Label:** Misuse of supply values and incorrect state assumptions lead to flawed token conversions, miscalculated fees, and denial-of-service conditions in liquidity and reward mechanisms.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description
`getExchangeRatio()` is supposed to be used for calculating exchange rate between HYPE and kHYPE. It checks kHYPESupply to be bigger than zero:
```solidity
    function getExchangeRatio() public view returns (uint256) {
        uint256 totalSlashing = validatorManager.totalSlashing();
        uint256 totalRewards = validatorManager.totalRewards();
        uint256 kHYPESupply = kHYPE.totalSupply();

@>      require(kHYPESupply > 0, "No kHYPE supply");

        // Calculate total HYPE: totalStaked + totalRewards - totalClaimed - totalSlashing
        uint256 totalHYPE = totalStaked + totalRewards - totalClaimed - totalSlashing;

        // Return ratio = totalHYPE / kHYPE.totalSupply (scaled by 1e18)
        return (totalHYPE * 1e18) / kHYPESupply;
    }
```
As a result, the first user call to `stake` will revert since `kHYPE.totalSupply()` is zero.

## Recommendations

Remove the check for `kHYPESupply > 0`.

---
### Example 2

**Auto Label:** **Improper state checks and runtime overrides enable irreversible fund locking and unfair rate inflation via manipulated liquidity or fee mechanics.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-06-superfluid-locker-system-judging/issues/210 

## Found by 
0x73696d616f, 0xb0k0, 0xbakeng, 0xloscar01, 37H3RN17Y2, SamuelTroyDomi, illoy\_sci, katz, nagato, newspacexyz

### Summary

Pumponomics can be effectively skipped since the eth amount used to create a position on uniswap could be different than the called amount. It uses all available balance inside the contract. This causes the intended buy-pressure mechanism to be skipped, and damages Superfluid's economic model.

### Root Cause

Inside [FluidLocker::provideLiquidity](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/d8beaeed47f766659a1600a87372a7905109aa3c/fluid/packages/contracts/src/FluidLocker.sol#L423-L429) function, only the eth sent in this call are pumped. However, all available wrapped eth are used to create a position. This allows locker owners to send eth manually to the locker, not when they are calling the provideLiquidity function, effectively skipping Pumponomics.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. Locker owner transfers certain amount of wrapped ethers to the locker directly by calling WETH9.transfer, they will be used as the paired asset for provideLiquidity function.
2. Call provideLiquidity function with a dust amount of eth, and a normal amount of supAmount to utilize the wrapped ether we sent in step 1.
Now a position is created without Pumponomics.

### Impact

Weaken the intended structural buy-pressure; causes protocol to lose potential fees earned from swaps and liquidity.

### PoC

Due to the difficulty of observing the effect of the pump function (1% difference), we directly modify the FluidLocker contract for visibility by creating a new field
```uint256 public ethPumped;``` 
and place it after all other fields that already exist.
To use it, we insert this line into the beginning of the [_pump](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/d8beaeed47f766659a1600a87372a7905109aa3c/fluid/packages/contracts/src/FluidLocker.sol#L639) function
```ethPumped += ethAmount;```
These modification should not change the behaviour of the contract.

Now, paste the following test into FluidLockerTest contract inside FluidLocker.t.sol
If needed, please import `import { IWETH9 } from "../src/token/IWETH9.sol";`
```solidity
    function testSkipPumponomics()
    external
    virtual
    {
        uint fundingAmount = 100e18;
        // Set up Alice's Locker to be functional
        // i. e. not revert due to "lack of funds", "no LP pool units", "no staker pool units", etc.
        _helperFundLocker(address(aliceLocker), fundingAmount);
        _helperLockerStake(address(bobLocker));
        _helperLockerProvideLiquidity(address(carolLocker));

        //Alice transfer the eth into locker via a plain call
        //Then call provideLiquidity with dust amount of eth
        //Therefore the pump function only pumps the dust eth amount, but the position is created with all eth in locker
        uint ethAmount = fundingAmount / (2 * 9900);
        address weth = _nonfungiblePositionManager.WETH9();
        vm.startPrank(ALICE);
        IWETH9(weth).deposit{ value: ethAmount }();
        IWETH9(weth).transfer(address(aliceLocker), ethAmount);
        aliceLocker.provideLiquidity{ value: 100 }(fundingAmount);
        vm.stopPrank();

        // Only a dust amount of eth has been pumped
        assertEq(FluidLocker(payable(address(aliceLocker))).ethPumped(), 1);
        // However, a position is opened with basically all the eth in the locker
        assertGt(FluidLocker(payable(address(aliceLocker))).activePositionCount(), 0);
        assertApproxEqAbs(0,
            IWETH9(weth).balanceOf(address(aliceLocker)),
            fundingAmount * 5 / 100 // 5% tolerance
        );
    }
```

### Mitigation

_No response_

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/superfluid-finance/fluid/pull/27

---
### Example 3

**Auto Label:** **Improper state checks and runtime overrides enable irreversible fund locking and unfair rate inflation via manipulated liquidity or fee mechanics.**  

**Original Text Preview:**

The [`BaseDynamicFee` hook contract](https://github.com/OpenZeppelin/uniswap-hooks/blob/f051c147dbf296b2b854de92970905a880cd1d51/src/fee/BaseDynamicFee.sol#L21) has a `virtual` `external` [`poke` function](https://github.com/OpenZeppelin/uniswap-hooks/blob/f051c147dbf296b2b854de92970905a880cd1d51/src/fee/BaseDynamicFee.sol#L59) that allows any pool initiated with this hook to (re)set an updated LPFee by anyone at any time. The updated LPFee comes from the [`_getFee(poolKey)` virtual function](https://github.com/OpenZeppelin/uniswap-hooks/blob/f051c147dbf296b2b854de92970905a880cd1d51/src/fee/BaseDynamicFee.sol#L37), which could return any fee based on either the current pool state or any other external conditions. However, if the `_getFee` implementation depends on some external factors, it opens up the possibility of manipulating the LPFee and swapping in a single transaction.

For example, suppose that `_getFee` for a V4 pool is dependent on the token balance of a Uniswap V3 pool having the same token pair. Suppose also that the returned fee for the V4 pool is inversely correlated to the corresponding V3 pool token balance. That is, `_getFee` returns a lower value when the V3 pool has a large balance and otherwise returns a higher value. Now, an attacker could take a flash loan, deposit into the V3 pool temporarily, and then call `poke` on the V4 pool. Since the V3 pool balance would have increased, the swap fee will drop and the attacker can pay lesser fees for subsequent swaps. If the subsequent users do not `poke` in advance, they can enjoy cheap fees without having to do any manipulation.

Consider imposing access control on the `poke` function so that it cannot be called arbitrarily. Otherwise, the `poke` function can be made `internal`, allowing the inheriting contracts to decide how to appropriately incorporate it with their logic. Alternatively, consider updating the documentation of the `poke` function so that the risk described above is clear to the inheriting contracts.

***Update:** Resolved in [pull request #34](https://github.com/OpenZeppelin/uniswap-hooks/pull/34) at commit [7e81272](https://github.com/OpenZeppelin/uniswap-hooks/pull/34/commits/7e812726b392af7376de78d31453f82fecc072f3).*

---
