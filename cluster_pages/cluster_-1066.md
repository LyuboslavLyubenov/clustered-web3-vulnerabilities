# Cluster -1066

**Rank:** #50  
**Count:** 360  

## Label
Failure to validate or enforce input constraints leads to incorrect state transitions, unauthorized access, or flawed logic execution—enabling exploitation through invalid or unintended parameter handling.

## Cluster Information
- **Total Findings:** 360

## Examples

### Example 1

**Auto Label:** Failure to validate critical inputs (addresses, chain IDs, token data) during initialization or execution, leading to invalid state, reentrancy, or unauthorized operations.  

**Original Text Preview:**

## Severity

**Impact**: High, malicious attacker can set L2 custom address to different address to break the bridge token.

**Likelihood**: Medium, attacker can front-ran the `registerTokenOnL2` to break the bridge token.

## Description

tSQD is designed so that it can be bridged from Ethereum (L1) to Arbitrum (L2) via Arbitrum’s generic-custom gateway.
However, the `registerTokenOnL2` function, which sets the L2 token address via `gateway.registerTokenToL2`, is not currently restricted.

```solidity
  function registerTokenOnL2(
    address l2CustomTokenAddress,
    uint256 maxSubmissionCostForCustomGateway,
    uint256 maxSubmissionCostForRouter,
    uint256 maxGasForCustomGateway,
    uint256 maxGasForRouter,
    uint256 gasPriceBid,
    uint256 valueForGateway,
    uint256 valueForRouter,
    address creditBackAddress
  ) public payable {
    require(!shouldRegisterGateway, "ALREADY_REGISTERED");
    shouldRegisterGateway = true;

    gateway.registerTokenToL2{value: valueForGateway}(
      l2CustomTokenAddress, maxGasForCustomGateway, gasPriceBid, maxSubmissionCostForCustomGateway, creditBackAddress
    );

    router.setGateway{value: valueForRouter}(
      address(gateway), maxGasForRouter, gasPriceBid, maxSubmissionCostForRouter, creditBackAddress
    );

    shouldRegisterGateway = false;
  }
```

An attacker can front-run the `registerTokenOnL2` and put an incorrect address for `l2CustomTokenAddress` to break the bridge token. Once it is called, the L2 token cannot be changed inside the gateway.

## Recommendations

Use the Ownable functionality inside tSQD and restrict `registerTokenOnL2` so that it can only be called by the owner/admin, as suggested by the Arbitrum bridge token design.

---
### Example 2

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
### Example 3

**Auto Label:** Insufficient input validation leads to invalid state transitions, unauthorized parameter manipulation, and unintended execution, resulting in financial loss, operational downtime, or system misbehavior.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

### Description
Multiple `op::update` operations in the smart contracts update the code and the data of the contract at once. If such an update is incorrect, there is no way to recover from it. For example, a snippet of this operation in the factory contract is shown in **Figure 3.1**:

```plaintext
} elseif (opcode == opcodes::update_admin) {
;; ok
throw_unless(errors::not_an_admin, equal_slice_bits(storage::admin_address,
sender_address));
update_addresses(in_msg_body~load_msg_addr(), storage::withdrawer_address);
send_builder(
sender_address,
begin_cell()
.store_uint(opcodes::excesses, 32)
.store_uint(query_id, 64),
SEND_MODE_CARRY_ALL_BALANCE
);
```

**Figure 3.1:** The logic managing admin role updates  
(dex/contracts/factory.fc#L273–L283)

### Recommendations
- **Short term:** Use a two-step process with a timelock to ensure that incorrect code or data updates are not immediately irreversible.
- **Long term:** Identify and document all possible actions that privileged accounts can take, along with their associated risks. This will facilitate codebase reviews and prevent future mistakes.

---
