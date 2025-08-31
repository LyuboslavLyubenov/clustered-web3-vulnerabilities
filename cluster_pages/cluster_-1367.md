# Cluster -1367

**Rank:** #413  
**Count:** 11  

## Label
Inaccurate numeric representation due to improper precision handling and lack of unit clarity, leading to arithmetic errors, misleading outputs, and compromised financial integrity.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Failure to handle edge cases in arithmetic operations leads to incorrect results or reverts, compromising precision and reliability in mathematical and financial computations.  

**Original Text Preview:**

The [`Float128::eq`](https://github.com/code-423n4/2025-04-forte/blob/4d6694f68e80543885da78666e38c0dc7052d992/src/Float128.sol# L1070) function is designed to return a boolean if the given two packed floats are equal. However, the issue lies with the way the function equates two packed floats via unwrapping:
```

    function eq(packedFloat a, packedFloat b) internal pure returns (bool retVal) {
        retVal = packedFloat.unwrap(a) == packedFloat.unwrap(b);
    }
```

As per the docs, it is clearly mentioned that Mantissa can be of two types M: 38 Digits and L: 72 digits.
```

/****************************************************************************************************************************
 * The mantissa can be in 2 sizes: M: 38 digits, or L: 72 digits                                                             *
 *      Packed Float Bitmap:                                                                                                 *
 *      255 ... EXPONENT ... 242, L_MATISSA_FLAG (241), MANTISSA_SIGN (240), 239 ... MANTISSA L..., 127 .. MANTISSA M ... 0  *
 *      The exponent is signed using the offset zero to 8191. max values: -8192 and +8191.                                   *
 ****************************************************************************************************************************/
```

So if there are two packed floats which are equal in nature upon deduction, but one of them has 38 digit mantissa and other has the 72 digit mantissa, the `eq` function would fail as unwrapping custom float type to underlying type (uint) means that the packed float with 72 digit mantissa will have the `L_MANTISSA_FLAG` set; which would introduce an incorrect unwrapped version than intended leading to false values.

### Impact

The `eq` function is one of the crucial components of the library, this issue renders it useless for scenarios when one of the packed float has the `L_MANTISSA_FLAG` on.

### Recommended mitigation steps

After running through a few different solutions, the recommendation would be to normalise values before equating, further optimizations are made to reduce gas costs:
```

    function eq(packedFloat a, packedFloat b) internal pure returns (bool retVal) {
        // If the bit patterns are equal, the values are equal
        if (packedFloat.unwrap(a) == packedFloat.unwrap(b)) {
            return true;
        }

        // If either is zero (no mantissa bits set), special handling
        bool aIsZero = (packedFloat.unwrap(a) & MANTISSA_MASK) == 0;
        bool bIsZero = (packedFloat.unwrap(b) & MANTISSA_MASK) == 0;

        if (aIsZero && bIsZero) return true;
        if (aIsZero || bIsZero) return false;

        // Getting the mantissa and exponent for each value
        (int mantissaA, int exponentA) = decode(a);
        (int mantissaB, int exponentB) = decode(b);

        // Checking if signs are different
        if ((mantissaA < 0) != (mantissaB < 0)) return false;

        // Getting absolute values
        int absA = mantissaA < 0 ? -mantissaA : mantissaA;
        int absB = mantissaB < 0 ? -mantissaB : mantissaB;

        // Applying exponents to normalize values
        // Convert both to a standard form with normalized exponents

        // Removing trailing zeros from mantissas (binary search kind of optimisation can be made, but later realised mantissas can be 10000000001 as well, sticking with this O(num_of_digits - 1) solution)
        while (absA > 0 && absA % 10 == 0) {
            absA /= 10;
            exponentA += 1;
        }

        while (absB > 0 && absB % 10 == 0) {
            absB /= 10;
            exponentB += 1;
        }

        // Checking if the normalized values are equal
        return (absA == absB && exponentA == exponentB);
    }
```

Below is the test that proves to the issue to be fixed when the mitigation above is replaced with the existing `eq` function:
```

    function testFixedEq() public {
        // Contains 72 digits (71 zeros) and -71 exponent (L Mantissa used here)
        packedFloat packed1 = Float128.toPackedFloat(100000000000000000000000000000000000000000000000000000000000000000000000, -71);

        // This is exactly the same value which would've resulted if packed1 was human readable (a * 10^b)
        packedFloat packed2 = Float128.toPackedFloat(1, 0);

        (int256 mantissa2, int256 exponent2) = packed2.decode();
        (int256 mantissa1, int256 exponent1) = packed1.decode();

        console2.log("Mantissa1: ", mantissa1); // Mantissa1:  100000000000000000000000000000000000000000000000000000000000000000000000
        console2.log("Exponent1: ", exponent1); // Exponent1:  -71
        console2.log("Mantissa2: ", mantissa2); // Mantissa2:  10000000000000000000000000000000000000
        console2.log("Exponent2: ", exponent2); // Exponent2:  -37

        // Eq Passes now
        bool isEqual = Float128.eq(packed1, packed2);
        assertEq(isEqual, true);
    }
```

### Proof of Concept

The below test case can ran inside the `/test` folder by creating a file called `test.t.sol`:
```

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "src/Float128.sol";

contract TestingContract is Test {
    using Float128 for packedFloat;
    using Float128 for int256;

function testBrokenEq() public {
        // Contains 72 digits (71 zeros) and -71 exponent (L Mantissa used here)
        packedFloat packed1 = Float128.toPackedFloat(100000000000000000000000000000000000000000000000000000000000000000000000, -71);

        // This is exactly the same value which would've resulted if packed1 was human readable (a * 10^b)
        packedFloat packed2 = Float128.toPackedFloat(1, 0);

        (int256 mantissa2, int256 exponent2) = packed2.decode();
        (int256 mantissa1, int256 exponent1) = packed1.decode();

        console2.log("Mantissa1: ", mantissa1); // Mantissa1:  100000000000000000000000000000000000000000000000000000000000000000000000
        console2.log("Exponent1: ", exponent1); // Exponent1:  -71
        console2.log("Mantissa2: ", mantissa2); // Mantissa2:  10000000000000000000000000000000000000
        console2.log("Exponent2: ", exponent2); // Exponent2:  -37

        // Eq fails
        bool isEqual = Float128.eq(packed1, packed2);
        assertEq(isEqual, false);
    }

}
```

**oscarserna (Forte) confirmed**

---

---
### Example 2

**Auto Label:** Inaccurate numerical calculations in integer arithmetic lead to precision loss, causing incorrect rate or point regeneration and potential system failure or invalid operations.  

**Original Text Preview:**

The regeneration of loan and redemption points is calculated as follows:

```solidity
File: CollateralControllerState.sol

370:        uint secondsElapsed = targetTimestamp - lastTimestamp;
371:
372:        (bool safeMul, uint _regeneratedPoints) = secondsElapsed.tryMul(regenerationRatePerMin);
373:        uint regeneratedPoints = (safeMul ? _regeneratedPoints : type(uint).max) / 60;
```

When `secondsElapsed * regenerationRatePerMin` is not a multiple of 60, the regenerated points will be rounded down, slowing the regeneration rate. For example, if `regenerationRatePerMin` is 3, and `secondsElapsed` is 12, the regenerated points will be 0.

It is recommended to replace the `regenerationRatePerMin` setting with a `regenerationRatePerSec` setting and remove the division by 60.

---
### Example 3

**Auto Label:** Failure to handle edge cases in arithmetic operations leads to incorrect results or reverts, compromising precision and reliability in mathematical and financial computations.  

**Original Text Preview:**

## Summary

The function `variance()` in Statistics.sol subtracts the average from each number in the array but the type is uint, because of that the function will revert unless all numbers are the same

## Vulnerability Details

The function `variance()` does a subtraction by iterating on every number on the array and subtracting by the average that was previously calculated

```Solidity
function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        mean = avg(data);
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
            uint256 diff = data[i] - mean; 
            sum += diff * diff;
        }
        ans = sum / data.length;
    }
```

The problem is that uint does not support negative values, because of that all subtractions that result in a negative amount will revert, and since it is the average of the numbers in the array, it will revert if the average is bigger than one of the numbers of the array, because of that, the only case it will not revert is if all numbers in the array are equal, and that is unlikely to happen.

## Impact

This function is called in the `finalizeValidation()` function, which is called in the end of the `validate()` function, because of that, almost all calls to `validate()` will revert, for this reason I believe it is a high.

## Proof of code

Install foundry, create a file in a subfolder in the test folder, and paste this:

```Solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.20;

import { Test } from "../../lib/forge-std/src/Test.sol";
import { console } from "../../lib/forge-std/src/console.sol";

contract Statistics {

    function avg(uint256[] memory data) internal pure returns (uint256 ans) {
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
            sum += data[i];
        }
        ans = sum / data.length;
    }

    function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        mean = avg(data);
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
            uint256 diff = data[i] - mean;
            sum += diff * diff;
        }
        ans = sum / data.length;
    }

    function stddev(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        (uint256 _variance, uint256 _mean) = variance(data);
        mean = _mean;
        ans = sqrt(_variance);
    }

    function sqrt(uint256 x) internal pure returns (uint256 y) {
        uint256 z = (x + 1) / 2;
        y = x;
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
    } 
}

contract TestLib is Test, Statistics {

    function test_testAvg(uint256 number1, uint256 number2, uint256 number3, uint256 number4, uint256 number5) public {
        uint256[] memory data = new uint256[]();
        data[0] = number1 % 1e50;
        data[1] = number2 % 1e50;
        data[2] = number3 % 1e50;
        data[3] = number4 % 1e50;
        data[4] = number5 % 1e50;
        require(number1 != number2, "test_testAvg: numbers must be different");
        uint256 ans = avg(data);
        //console.log(ans);
    }

    function test_testStddev(uint256 number1, uint256 number2, uint256 number3, uint256 number4, uint256 number5) public {
        uint256[] memory data = new uint256[]();
        data[0] = number1 % 1e50;
        data[1] = number2 % 1e50;
        data[2] = number3 % 1e50;
        data[3] = number4 % 1e50;
        data[4] = number5 % 1e50;
        require(number1 != number2, "test_testStddev: numbers must be different");
        vm.expectRevert();
        (uint256 ans, uint256 mean) = stddev(data);
    }

    function test_testSqrt() public {
        uint256 x = 16;
        uint256 ans = sqrt(x);
        //console.log(ans);
    }

    function test_testVariance(uint256 number1, uint256 number2, uint256 number3, uint256 number4, uint256 number5) public {
        uint256[] memory data = new uint256[]();
        data[0] = number1 % 1e50;
        data[1] = number2 % 1e50;
        data[2] = number3 % 1e50;
        data[3] = number4 % 1e50;
        data[4] = number5 % 1e50;
        require(number1 != number2, "test_testVariance: numbers must be different");
        vm.expectRevert();
        (uint256 ans, uint256 mean) = variance(data);
    }
}
```

All calls to `stddev()` and `variance()` will revert as expected. The 1e50 limit is to avoid overflows that will fail the test.

## Tools Used

Foundry

## Recommendations

Cast to int256 when calculating, the multiplication by itself will make the number always positive afterwards:

```diff
function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        mean = avg(data);
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
-            uint256 diff = data[i] - mean;
+           int256 diff = int256(data[i]) - int256(mean); 
-            sum += diff * diff;
+           sum += uint256(diff * diff);
        }
        ans = sum / data.length;
    }
```

---
