# Cluster -1060

**Rank:** #121  
**Count:** 129  

## Label
Insufficient input validation allows malicious or erroneous values to bypass critical bounds, leading to invalid state transitions, over-leveraging, and denial-of-service, compromising collateralization and system integrity.

## Cluster Information
- **Total Findings:** 129

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

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Severity: Low Risk

## Context
BaseVault.sol#L126-L130

## Description
In the `_executeSubmit()` function, `_noPendingApprovalsInvariant()` is executed before the `_afterSubmitHooks()` call. Therefore, if the after-submit hook makes a callback to the vault to execute `approve()`, that approval will be unchecked. 

This scenario is theoretically possible because the vault does not check if a registered callback has been received during the operation, so the callback can still be executed after the operation. The callback would need to pass the caller and selector checks, so the situation would likely be a configuration error and compromise of the hook or target contract.

## Recommendation
While the above scenario is unlikely to occur, to be on the safe side, consider checking `_noPendingApprovalsInvariant()` after `_afterSubmitHooks()` (i.e., at the end of the `submit()` function) to explicitly guarantee that no approvals are left at the end of the submission.

## Aera
Fixed in PR 218.

## Spearbit
Verified.

---
### Example 3

**Auto Label:** Failure to validate critical state invariants during execution flows, enabling race conditions, unchecked approvals, or invalid state transitions that compromise asset integrity and protocol security.  

**Original Text Preview:**

## Severity: Medium Risk

**Context:** `OracleRegistry.sol#L187-L198`

**Description:**  
The vault owners could accept an unexpected oracle through the `acceptPendingOracle()` function. 

**Consider the following scenario:**

1. The protocol owner schedules an update.
2. The vault owner sees the update and sends a transaction to accept it.
3. Before the transaction is executed, the protocol owner cancels the update and schedules another one.
4. The vault owner ends up accepting an oracle they do not expect.

Eventually, the vault owner will still use the new oracle after the update delay has passed. However, the vault owner may want to take enough time to review the new oracle before accepting it in case of security risks.

**Recommendation:**  
Consider modifying the `acceptPendingOracle()` function so that it takes a parameter `pendingOracle`, which is compared with the on-chain `pendingOracleData[base][quote]` value. If the two values are different, the function should revert.

**Aera:** Fixed in PR 289.  
**Spearbit:** Verified.

---
