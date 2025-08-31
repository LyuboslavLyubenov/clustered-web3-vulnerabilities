# Cluster -1141

**Rank:** #378  
**Count:** 14  

## Label
Race conditions and flawed state logic enable unauthorized or duplicate operations, leading to inconsistent or invalid state transitions in critical functions.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Race conditions and improper state transitions lead to inconsistent rate limits, allowing unauthorized bursts or manipulation of allowed inflows and outflows.  

**Original Text Preview:**

## Severity: Low Risk

## Context
`OndoRateLimiter.sol#L367-L379`

## Description
Currently, if `CONFIGURER_ROLE` sets `GlobalSubscriptionLimit`, `GlobalRedemptionLimit`, `UserSubscriptionRateLimit`, or `UserRedemptionRateLimit`, then `capacityUsed` is reset to 0. This can cause issues since the User might have already exhausted their limits, and this call revives the full limit without waiting for the window period.

## Proof of Concept
1. Attacker frontruns the `setGlobalSubscriptionLimit` call and subscribes fully to the current limit.
2. The `setGlobalSubscriptionLimit` call executes and sets a new limit. At the same time, it also sets `capacityUsed` to 0.
3. The attacker can subscribe again with the new limit instantly without needing to wait.

## Recommendation
New `capacityUsed` can be set based on the current capacity used percentage. Example:
1. If the old limit was 100 and 50% capacity was used:
2. Now, if the limit is changed to 200, then new `capacityUsed` can be set to 100 (50%).

## Ondo Finance
Acknowledged. If we are ever concerned about limits being abused during a change, we can pause the manager contracts prior to setting the new limits. Regardless, when setting new rate limits, we can always be aware that this may happen and simply be prepared to process more funds as opposed to adding contract complexity.

## Spearbit
Acknowledged.

---
### Example 2

**Auto Label:** Race conditions and improper state transitions lead to inconsistent rate limits, allowing unauthorized bursts or manipulation of allowed inflows and outflows.  

**Original Text Preview:**

The `_changeMintingLimit()` and `_changeBurningLimit()` incorrectly wait for the full replenishment duration before the new limit can be fully used.

When the limit is reduced from `type(uint256).max` to any new limit, the function uses `type(uint256).max` as the `oldMaxLimit` to calculate the current and new limits. This approach results in the new current limit being returned as 0, 1-day replenishment before the limit is available to use.

The limit `type(uint256).max` is a special case that bypasses the checks and allows the limit to be used immediately. In contrast, when the limit is set to any other value for the first time, it can be used immediately without waiting for the full 1-day replenishment duration.

```solidity
function _calculateNewCurrentLimit(uint256 newMaxLimit, uint256 oldMaxLimit, uint256 currentLimit)
    internal
    pure
    returns (uint256)
{
    if (newMaxLimit > oldMaxLimit) {
        return currentLimit + (newMaxLimit - oldMaxLimit);
    }
    uint256 difference = oldMaxLimit - newMaxLimit;
@>  return currentLimit > difference ? currentLimit - difference : 0;
}
```

## Recommendations

Consider treating the `oldMaxLimit` separately when it changes from `type(uint256).max` to any other limit. For example, treat it as a new limit being created.

---
### Example 3

**Auto Label:** Race conditions and improper state transitions lead to inconsistent rate limits, allowing unauthorized bursts or manipulation of allowed inflows and outflows.  

**Original Text Preview:**

The `_inflow()` function is designed to reduce the `amountInFlight` by the `_amount` passed to it (`inflowAmount`), with a lower bound of zero:

```solidity
function _inflow(uint32 _srcEid, uint256 _amount) internal virtual {
    RateLimit storage rl = rateLimits[_srcEid];
    rl.amountInFlight = _amount >= rl.amountInFlight ? 0 : rl.amountInFlight - _amount;
}
```

However, let's imagine the following two scenarios:

- `RateLimiter.RateLimitConfig(1337, 1000, 100);`

**Scenario 1**

- Initially, the system starts with 0 units in flight and 1000 units available to be sent.
- In step 1, an `outflow(500)` occurs, reducing the available units to 500, with 500 units now in flight.
- In step 2, an `inflow(1000)` clears the inflight amount, bringing it back to 0, and restoring 1000 units available to be sent.
- Finally, in step 3, another `outflow(500)` is processed, leaving 500 units in flight and 500 units that can still be sent.

**Scenario 2**

- The initial state has 0 units in flight and 1000 units available to send.
- In step 1, an `outflow(500)` occurs, reducing the available amount to 500, with 500 units now in flight.
- In step 2, another `outflow(500)` fully utilizes the available amount, leaving 1000 units in flight and 0 units available.
- In step 3, an `inflow(1000)` clears the inflight amount, resetting it to 0 and making 1000 units available again.
  Despite both scenarios involving a total of 1000 units of outflow and 1000 units of inflow processed in the same timestamp, the remaining rate limit differs based on the order of operations. This inconsistency proves that the rate limiter behaves differently when the sequence of inflow and outflow operations changes, even though the net effect is the same.

Users performing the same net actions will experience different rate limits based solely on the sequence of their operations. Moreover, it is expected that users will deliberately arrange operations to maximize their rate limits. **A rate limiter should apply limits consistently, regardless of the order of operations, as long as the net effect is the same.**

Consider updating the contract's logic to ensure that the order of operations within the same timestamp does not alter the remaining rate limit.

## Layer Zero comments

Interesting. We have taken the approach that transaction ordering within a given block matters.

The current RateLimiter fixes the range of amountCanBeSent = [0, MAX]. Per timestamp (block), we could track excessive inflows in a separate variable (as there is no Inbound Rate Limit), and prioritize subtracting from that first. However, we cannot do the opposite and allow excessive outflows (as there is an Outbound Rate Limit) that might be compensated for later. The asymmetry of this solution is confusing, and costs additional storage reads/writes.

Given a sufficiently large limit, this shouldn't really matter. As such, we are choosing to just add documentation explaining the limitation.

[LayerZero-Labs/devtools#904](https://github.com/LayerZero-Labs/devtools/pull/904)

---
