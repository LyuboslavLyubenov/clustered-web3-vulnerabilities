# Cluster -1127

**Rank:** #103  
**Count:** 163  

## Label
Insufficient input validation and access control enable attackers to submit invalid, malformed, or unauthorized requests, leading to resource exhaustion, data manipulation, unauthorized operations, and fee drain—undermining system integrity and security.

## Cluster Information
- **Total Findings:** 163

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
### Example 3

**Auto Label:** Insufficient input validation leads to invalid state transitions, unauthorized parameter manipulation, and unintended execution, resulting in financial loss, operational downtime, or system misbehavior.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

### Description
The SYWallet contract is not tested because the `syWalletCode` variable is overwritten in the `setup.js`. Only the `EvaaSyWallet` contract is used for both generic and Evaa SY token contract tests.

```javascript
async function compileContracts() {
    underlyingMinterCode = await compile('StakeMinter');
    underlyingWalletCode = await compile('StakeWallet');
    syMinterCode = await compile('SYMinter');
    syWalletCode = await compile('SYWallet');
    ptMinterCode = await compile('PTMinter');
    ptWalletCode = await compile('PTWallet');
    ytMinterCode = await compile('YTMinter');
    ytWalletCode = await compile('YTWallet');
    redeemDepositCode = await compile('RedeemDeposit');
    syWalletCode = await compile('EvaaSYWallet');
    evaaSYMinterCode = await compile('EvaaSYMinter');
    evaaSYWalletCode = await compile('EvaaSYWallet');
    evaaMockCode = await compile('EvaaMock');
}
```
*Figure 5.1: The test setup function tests/utils/setup.ts#L39-L53*

### Recommendations
- **Short term:** Update the test setup file to use a different variable for the `EvaaSYWallet` contract.
- **Long term:** Ensure that the old code coverage is not affected by the new code and new test cases.

---
