# Cluster -1305

**Rank:** #310  
**Count:** 23  

## Label
Missing input bounds checking allows malicious or invalid configurations to be accepted, leading to protocol instability, mismanagement, or denial-of-service through unreasonably large or out-of-range parameter values.

## Cluster Information
- **Total Findings:** 23

## Examples

### Example 1

**Auto Label:** Missing input bounds checking allows malicious or invalid configurations to be accepted, leading to protocol instability, mismanagement, or denial-of-service through unreasonably large or out-of-range parameter values.  

**Original Text Preview:**

##### Description
This issue has been identified within the `setConfig` function of the `LPManager` contract. 

There are minimal checks on parameters like `cooldownDuration` and `maxOracleAge`. If set to very long values, they could inadvertently disrupt normal protocol operations or lead to unpredictable behaviors.

The issue is classified as **low** severity because it mainly involves a risk of misconfiguration rather than an immediate exploit.

##### Recommendation
We recommend setting reasonable upper and lower limits for both `cooldownDuration` and `maxOracleAge` to prevent harmful configurations.

---
### Example 2

**Auto Label:** Missing input bounds checking allows malicious or invalid configurations to be accepted, leading to protocol instability, mismanagement, or denial-of-service through unreasonably large or out-of-range parameter values.  

**Original Text Preview:**

##### Description
This issue has been identified within the `setConfig` function of the `LPManager` contract. 

There are minimal checks on parameters like `cooldownDuration` and `maxOracleAge`. If set to very long values, they could inadvertently disrupt normal protocol operations or lead to unpredictable behaviors.

The issue is classified as **low** severity because it mainly involves a risk of misconfiguration rather than an immediate exploit.

##### Recommendation
We recommend setting reasonable upper and lower limits for both `cooldownDuration` and `maxOracleAge` to prevent harmful configurations.

---
### Example 3

**Auto Label:** Missing input validation and bounds checking allows attackers to set unreasonably large or invalid values, leading to unbounded behavior, denial of service, or permanent misconfiguration of core protocol parameters.  

**Original Text Preview:**

## Issue Summary

The function `state::calculate_points_for_level` does not properly enforce `MAX_LEVEL`, allowing users to potentially accumulate enough experience points to exceed the intended maximum level. The function only handles the case where `level == MAX_LEVEL`, returning a fixed `LEVEL_25_POINTS`. It does not return an error if `level > MAX_LEVEL`, which implies that the function may calculate required points for any level, even beyond the intended cap. This allows users to continue leveling up indefinitely as long as they accumulate enough points. 

Additionally, due to point accumulation, users may theoretically reach level 256, which is significantly higher than the existing cap of 25.

> _claynosaurz-staking/src/state.rs rust_

```rust
/// Calculates the points required for a given level.
fn calculate_points_for_level(&self, level: u8) -> Result<u64> {
    if level == MAX_LEVEL {
        return Ok(LEVEL_25_POINTS);
    }
    [...]
}
```

## Remediation

Enforce the `MAX_LEVEL` limit to prevent over-leveling and also adjust the value of `MAX_LEVEL` to be consistent with the point calculation.

## Patch

Fixed in commit `0b87d6e`.

---
