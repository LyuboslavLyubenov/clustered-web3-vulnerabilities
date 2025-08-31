# Cluster -1110

**Rank:** #339  
**Count:** 19  

## Label
Inadequate input validation for slippage parameters leads to inconsistent, unsafe, or invalid arithmetic operations, enabling exploitable trading risks and potential user capital loss.

## Cluster Information
- **Total Findings:** 19

## Examples

### Example 1

**Auto Label:** Inadequate input validation for slippage parameters leads to inconsistent, unsafe, or invalid arithmetic operations, enabling exploitable trading risks and potential user capital loss.  

**Original Text Preview:**

##### Description

The `_checkSlippage` function does not properly validate slippage conditions when decreasing positions, particularly for long positions. The function applies different logic based on the `isLong` value, leading to inconsistent and potentially unsafe checks.

For long positions, a decrease in size may result in a **negative slippage scenario** that is not caught by the function. This allows position modifications with prices outside the acceptable slippage range, potentially resulting in unintended losses for users.

##### Proof of Concept

The provided POC demonstrates a scenario where a long position decrease with a requested price of `2605` does **not revert** even though the final price (`2600`) falls outside the acceptable slippage range:

```
   function test_decrease_no_slippage() public {
        vm.startPrank(owner);

        IERC20(usdcAddr).approve(futuresCoreAddr, IERC20(usdcAddr).balanceOf(owner));

        uint256 collateral = 100e6;
        uint256 size = 1000e6;

        uint256 indexPrice = 2300e6;
        bool isLong = true;
        bool isIncrease = true;
        bytes32 key = keccak256(abi.encode("ETH/USD"));

        console.log("Index price", indexPrice);
        console.log("Perpetual price", futuresCore.getPerpetualPrice(key, indexPrice));
        console.log("Perpetual price position", futuresCore.getPerpetualPriceForPosition(key, indexPrice, int256(size), isLong));

        futuresCore.modifyPosition(owner, collateral, size, indexPrice, isLong, isIncrease, key, futuresCore.getPerpetualPrice(key, indexPrice), 1000, true);

        collateral = 50e6;
        size = 1000e6;
        indexPrice;
        indexPrice = 2600e6;
        isLong = true;
        isIncrease = false;
        key = keccak256(abi.encode("ETH/USD"));

        console.log("Index price", indexPrice);
        console.log("Perpetual price", futuresCore.getPerpetualPrice(key, indexPrice));
        console.log("Perpetual price position (-)", futuresCore.getPerpetualPriceForPosition(key, indexPrice, -int256(size), isLong));

        futuresCore.modifyPosition(owner, collateral, size, indexPrice, isLong, isIncrease, key, futuresCore.getPerpetualPrice(key, indexPrice), 0, true);

        vm.stopPrank();
    }

```

This behavior arises because the `_checkSlippage` function treats long and short positions differently instead of applying a **unified range check**.

![POC output](https://halbornmainframe.com/proxy/audits/images/677a8d19adabd5e3b409d33c)

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:M/A:N/D:N/Y:N/R:N/S:C (6.3)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:M/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

Update `_checkSlippage` to use a **standardized range validation** for both long and short positions. For example:

```
if (finalPrice > value2max || finalPrice < value2min) {
    return false;
}
```

This approach eliminates the need for separate logic paths based on `isLong` and ensures consistent validation for all scenarios, including position decreases.

Additionally, re-test the POC after applying the fix to confirm that slippage validation properly rejects prices outside the specified range.

##### Remediation

**SOLVED**: `_checkSlippage` is now performing validation on the range independently of the position those considering the reduction case for a long position.

##### Remediation Hash

2a9423c22a20ceddb37bd7c4a166e686817373a6

---
### Example 2

**Auto Label:** Missing bounds validation of critical parameters during initialization, allowing invalid configurations that compromise economic logic and system integrity.  

**Original Text Preview:**

## Context
IrmFactory.sol#L20-L21

## Description
Check for other `VariableIrm.Config` parameters missing in `IrmFactory.createVariableIrm` such as:
- **minFullUtilizationRate**
- **maxFullUtilizationRate**

## Recommendation
Make sure `minFullUtilizationRate` <= `maxFullUtilizationRate`.

## Fixes
- **Dahlia**: Fixed in commit `bb8ce8fa`.
- **Cantina Managed**: Fixed.

---
### Example 3

**Auto Label:** Inadequate input validation for slippage parameters leads to inconsistent, unsafe, or invalid arithmetic operations, enabling exploitable trading risks and potential user capital loss.  

**Original Text Preview:**

##### Description

The `setswapSlippage` function does not validate that the provided `slippage` value is less than or equal to `BASIS_POINTS`. If `slippage` is set to a value greater than `BASIS_POINTS`, it will result in underflows during calculations in the `swapEthToUsdc` and `swapUsdcToEth` functions. Specifically, the subtraction for `minUsdcOut` and `minWethOut` could yield negative results, causing the contract to revert.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:M/A:M/D:N/Y:N/R:P/S:C (3.9)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:M/A:M/D:N/Y:N/R:P/S:C)

##### Recommendation

Add a validation check in the `setswapSlippage` function to ensure that `slippage` does not exceed `BASIS_POINTS`. For example:

```
function setswapSlippage(uint256 slippage) external onlyOwner {
    require(slippage <= BASIS_POINTS, "Slippage exceeds allowable range");
    swapSlippage = slippage;
}

```

This ensures the `swapEthToUsdc` and `swapUsdcToEth` functions operate correctly and avoids unintended underflows. By restricting `slippage` to a valid range, the contract's behavior remains predictable and secure.

##### Remediation

**SOLVED**: The code is now checking for the upper bounds.

##### Remediation Hash

1f89558c1394d2c6a59238172e3e17ed50e32265

---
