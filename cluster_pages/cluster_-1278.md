# Cluster -1278

**Rank:** #127  
**Count:** 117  

## Label
Insufficient input validation and incorrect state initialization lead to denial-of-service and functional failures by allowing invalid or zero values to disable critical transaction processing and retry mechanisms.

## Cluster Information
- **Total Findings:** 117

## Examples

### Example 1

**Auto Label:** Missing or insufficient input validation leads to incorrect state transitions, unauthorized access, or invalid execution, enabling logic errors, denial-of-service, or exploitable conditions.  

**Original Text Preview:**

**Description:** While the combination of `LibBuyTheDipGeometricDistribution::isValidParams` and `LibBuyTheDipGeometricDistribution::geometricIsValidParams` should be functionally equivalent to `LibGeometricDistribution::isValidParams`, except for the shift mode enforcement, this is not the case.

```solidity
    function geometricIsValidParams(int24 tickSpacing, bytes32 ldfParams) internal pure returns (bool) {
        (int24 minUsableTick, int24 maxUsableTick) =
            (TickMath.minUsableTick(tickSpacing), TickMath.maxUsableTick(tickSpacing));

        // | shiftMode - 1 byte | minTickOrOffset - 3 bytes | length - 2 bytes | alpha - 4 bytes |
        uint8 shiftMode = uint8(bytes1(ldfParams));
        int24 minTickOrOffset = int24(uint24(bytes3(ldfParams << 8)));
        int24 length = int24(int16(uint16(bytes2(ldfParams << 32))));
        uint256 alpha = uint32(bytes4(ldfParams << 48));

        // ensure minTickOrOffset is aligned to tickSpacing
        if (minTickOrOffset % tickSpacing != 0) {
            return false;
        }

        // ensure length > 0 and doesn't overflow when multiplied by tickSpacing
        // ensure length can be contained between minUsableTick and maxUsableTick
        if (
            length <= 0 || int256(length) * int256(tickSpacing) > type(int24).max
                || length > maxUsableTick / tickSpacing || -length < minUsableTick / tickSpacing
        ) return false;

        // ensure alpha is in range
@>      if (alpha < MIN_ALPHA || alpha > MAX_ALPHA || alpha == ALPHA_BASE) return false;

        // ensure the ticks are within the valid range
        if (shiftMode == uint8(ShiftMode.STATIC)) {
            // static minTick set in params
            int24 maxTick = minTickOrOffset + length * tickSpacing;
            if (minTickOrOffset < minUsableTick || maxTick > maxUsableTick) return false;
        }

        // if all conditions are met, return true
        return true;
    }
```

The missing `alphaX96` validation can result in the potential issue described in the `LibGeometricDistribution` comment:

```solidity
    function isValidParams(int24 tickSpacing, uint24 twapSecondsAgo, bytes32 ldfParams) internal pure returns (bool) {
        ...
        // ensure alpha is in range
        if (alpha < MIN_ALPHA || alpha > MAX_ALPHA || alpha == ALPHA_BASE) return false;

@>      // ensure alpha != sqrtRatioTickSpacing which would cause cum0 to always be 0
        uint256 alphaX96 = alpha.mulDiv(Q96, ALPHA_BASE);
        uint160 sqrtRatioTickSpacing = tickSpacing.getSqrtPriceAtTick();
        if (alphaX96 == sqrtRatioTickSpacing) return false;
```

Additionally, `LibBuyTheDipGeometricDistribution::geometricIsValidParams` does not validate the minimum liquidity densities, unlike `LibGeometricDistribution::isValidParams`.

**Impact:** The current implementation of `LibBuyTheDipGeometricDistribution::geometricIsValidParams` does not prevent configuration with an `alpha` that can cause unexpected behavior, resulting in swaps and deposits reverting.

**Recommended Mitigation:** Ensure that `alpha` is in range:
```diff
        if (alpha < MIN_ALPHA || alpha > MAX_ALPHA || alpha == ALPHA_BASE) return false;
++      // ensure alpha != sqrtRatioTickSpacing which would cause cum0 to always be 0
++      uint256 alphaX96 = alpha.mulDiv(Q96, ALPHA_BASE);
++      uint160 sqrtRatioTickSpacing = tickSpacing.getSqrtPriceAtTick();
++      if (alphaX96 == sqrtRatioTickSpacing) return false;
```

Additionally ensure the liquidity density is nowhere equal to zero, but rather at least `MIN_LIQUIDITY_DENSITY`.

**Bacon Labs:** Fixed in [PR \#103](https://github.com/timeless-fi/bunni-v2/pull/103). The minimum liquidity check is intentionally omitted to allow the alternative LDF to be at a lower price where the default LDF may have less than the minimum required liquidity.

**Cyfrin:** Verified, the alpha validation has been added to `LibBuyTheDipGeometricDistribution::geometricIsValidParams`.

---
### Example 2

**Auto Label:** Missing or insufficient input validation leads to incorrect state transitions, unauthorized access, or invalid execution, enabling logic errors, denial-of-service, or exploitable conditions.  

**Original Text Preview:**

**Description:** `validatorMEVRewardsPercentage`is a reward percentage with valid values ranging from 0 to 10000 (100%). `PolygonStrategy::setValidatorMEVRewardsPercentage` rightly checks that the reward percentage does not exceed 10000. However this validation is missing during initialization.

```solidity
    function initialize(
        address _token,
        address _stakingPool,
        address _stakeManager,
        address _vaultImplementation,
        uint256 _validatorMEVRewardsPercentage,
        Fee[] memory _fees
    ) public initializer {
        __Strategy_init(_token, _stakingPool);

        stakeManager = _stakeManager;
        vaultImplementation = _vaultImplementation;
        validatorMEVRewardsPercentage = _validatorMEVRewardsPercentage; //@audit missing check on validatorMEVRewardsPercentage

```

**Recommended Mitigation:** Consider making the validation checks consistent at every place where the `validatorMEVRewardsPercentage` is updated.

**Stake.Link:** Resolved in [PR 151](https://github.com/stakedotlink/contracts/pull/151/commits/2ef53d4f0dac80c6ba1dc31ec516a28d7ed02851)

**Cyfrin:** Resolved.

---
### Example 3

**Auto Label:** Inadequate input and state validation leads to inconsistent, exploitable system configurations and data integrity failures, enabling unauthorized state changes and financial manipulation.  

**Original Text Preview:**

In cases where the edge between tokens is not set, the `edge()` function will return a default edge. However, this default edge is not properly checked for correctness. Specifically, when `defaultEdge.amplitude == 0`, the `edge()` function should `revert` to prevent using an invalid edge, as it is used in the `SwapFacet._preSwap()` and `LiqFacet.addLiq()` functions this could lead to unexpected behavior.

The issue arises from the removal of the proper check in the `Edge.getSlot0()` function, which was never reintroduced in the `Store::edge()` function. Additionally, the removal of this check leaves the `error NoEdgeSettings(address token0, address token1);` unused in the code.

Consider adding the missing sanity check to the `Store::edge()` function.

---
