# Cluster -1371

**Rank:** #329  
**Count:** 21  

## Label
Failure to properly handle errors or invalid inputs leads to incorrect state transitions, incorrect calculations, or denial of service, resulting in financial loss or compromised functionality.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Failure to properly handle errors or invalid inputs leads to incorrect state transitions, incorrect calculations, or denial of service, resulting in financial loss or compromised functionality.  

**Original Text Preview:**

The `StrategyPassiveManagerHyperswap` contract contains a potential division by zero vulnerability in the `twap()` function. The owner can set `twapInterval` to zero using `setTwapInterval()`, which would cause the TWAP calculation to revert.

```solidity
function setTwapInterval(uint32 _interval) external onlyOwner {
    emit SetTwapInterval(twapInterval, _interval);
    twapInterval = _interval; // No validation - can be set to 0
}

function twap() public view returns (int56 twapTick) {
    uint32[] memory secondsAgo = new uint32[](2);
    secondsAgo[0] = uint32(twapInterval);
    secondsAgo[1] = 0;
    (int56[] memory tickCuml,) = IUniswapV3Pool(pool).observe(secondsAgo);
    twapTick = (tickCuml[1] - tickCuml[0]) / int32(twapInterval); // Division by zero if twapInterval = 0
}
```
If `twapInterval` is set to zero, the following critical functions will revert due to the division by zero in `twap()`
 
deposit(), withdraw(),harvest(),moveTicks(),setPositionWidth(),unpause()
This would effectively break the core functionality of the strategy until `twapInterval` is set to a non-zero value.

This can be resolved by adding validation in `setTwapInterval()` to prevent setting the interval to zero.

---
### Example 2

**Auto Label:** Failure to properly handle errors or invalid inputs leads to incorrect state transitions, incorrect calculations, or denial of service, resulting in financial loss or compromised functionality.  

**Original Text Preview:**

##### Description

The `AutomationPositionsV2` and `AutomationOrdersV2` contracts allow setting different `PRICE_BAND_WIDTH` values independently.

This inconsistency can lead to:

1. **Mismatched Band Calculations:** Orders and positions may be **mapped to different bands**, causing issues when deleting **related orders** tied to positions.
2. **Missed Deletions:** Orders may not be correctly identified and removed, potentially leading to **orphaned entries** or **stale data**.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:M/A:L/D:N/Y:N/R:F/S:C (1.8)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:M/A:L/D:N/Y:N/R:F/S:C)

##### Recommendation

Ensure synchronization of `PRICE_BAND_WIDTH` across related contracts.

1. **Factory Contract:** Deploy **AutomationPositionsV2** and **AutomationOrdersV2** through a **factory contract** that initializes them with the **same** `PRICE_BAND_WIDTH` value.
2. **Immutable Band Width:** Consider making `PRICE_BAND_WIDTH` **immutable** to prevent runtime changes that could lead to mismatches.
3. **Shared Storage (Optional):** Store the `PRICE_BAND_WIDTH` in a **shared storage contract** or **config module** to enforce consistency without duplicating values.

This approach guarantees **synchronized band calculations**, reduces the risk of **mismatched deletions**, and improves **data consistency** across contracts.

##### Remediation

**ACKNOWLEDGED:** The **Dexodus team** acknowledged this finding.

---
### Example 3

**Auto Label:** Failure to properly handle errors or invalid inputs leads to incorrect state transitions, incorrect calculations, or denial of service, resulting in financial loss or compromised functionality.  

**Original Text Preview:**

##### Description

The `setPRICE_BAND_WIDTH` function in **Automation contracts** allows arbitrary updates to the `PRICE_BAND_WIDTH` value.

Changing this value dynamically can lead to **inconsistent band indexing** for existing positions because:

* **Old positions** remain mapped to bands calculated with the previous width, while **new positions** are mapped based on the updated width.
* Positions that should be in separate bands may get **merged into the same band**, resulting in **missed liquidations**, **incorrect executions**, or **skipped processing** during upkeep.

Example:

* Initial `PRICE_BAND_WIDTH = 200`, placing a position at index 2 (400).
* Updated `PRICE_BAND_WIDTH = 100`, causing positions at 400 to map to index 4 (400/100).
* **Result:** Old and new positions incorrectly share the same band, leading to **unstable system behavior**.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:H/A:H/D:N/Y:N/R:P/S:C (5.9)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:H/A:H/D:N/Y:N/R:P/S:C)

##### Recommendation

1. **Remove** `setPRICE_BAND_WIDTH`: Prevent runtime modifications to `PRICE_BAND_WIDTH` entirely, enforcing immutability for consistent indexing.
2. **Add Migration Function (Optional):** If updates are required, introduce a **migration process** to recalculate and **remap all positions** to new bands. However, this approach may be **gas-intensive** and should be carefully considered.
3. **Off-Chain Band Management (Alternative):** Consider managing band calculations **off-chain** and only submitting validated liquidation data on-chain, reducing complexity and gas usage.

Restricting dynamic changes or implementing migrations ensures **system stability**, **data consistency**, and **reliable liquidations**.

##### Remediation

**SOLVED**: The `setPRICE_BAND_WIDTH` function was removed.

##### Remediation Hash

2a9423c22a20ceddb37bd7c4a166e686817373a6

---
