# Cluster -1336

**Rank:** #412  
**Count:** 11  

## Label
Flawed access control and invalid state validation enable attackers to manipulate pool memberships, siphon rewards, or bypass authorization by misrepresenting sender identity or pool existence.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Flawed access control and invalid state validation enable attackers to manipulate pool memberships, siphon rewards, or bypass authorization by misrepresenting sender identity or pool existence.  

**Original Text Preview:**

## Description

Incorrect node address is recorded as allowed to be staked for, which may result in unexpected and unintended results. On line [305], the address to allow staking for is set as the caller of the function `msg.sender`, however, it should be a `_nodeAddress`, as the call may not necessarily come from the node itself, but it may also be a node’s RPL withdrawal address:

```solidity
setBool(keccak256(abi.encodePacked("node.stake.for.allowed", msg.sender, _caller)), _allowed);
```

## Recommendations

Modify the implementation to record `node.stake.for.allowed` based on node address, e.g.:

```solidity
setBool(keccak256(abi.encodePacked("node.stake.for.allowed", _nodeAddress, _caller)), _allowed);
```

## Resolution

The issue has been addressed in commit `d67fe0c` as per the recommendations.

---
### Example 2

**Auto Label:** Failure to validate pool legitimacy enables attackers to deploy or interact with malicious pools, leading to unauthorized control, griefing, or user exploitation.  

**Original Text Preview:**

##### Description

The `storePoolInformation` function under `storePoolInformation` does allow replacing the pool ID of 0, overriding the `_protectionPool` and `shield` tokens.

Code Location
-------------

#### src/core/PolicyCenter.sol

```
function storePoolInformation(
    address _pool,
    address _token,
    uint256 _poolId
) external {
    // @audit-issue This allows replacing any pool, including the ID 0 with
    // _protectionPool and shield
    require(msg.sender == priorityPoolFactory, "Only factory can store");

    tokenByPoolId[_poolId] = _token;
    priorityPools[_poolId] = _pool;

    _approvePoolToken(_token);
}

```

##### Score

Impact: 3  
Likelihood: 1

##### Recommendation

**SOLVED**: It is verifying using an assert that the protection pool information is never altered.

---
### Example 3

**Auto Label:** Failure to validate pool legitimacy enables attackers to deploy or interact with malicious pools, leading to unauthorized control, griefing, or user exploitation.  

**Original Text Preview:**

## Context
**File:** MstWrapperFactory.sol  
**Line:** 87  

## Description
Although `PoolFactory` ensures that the pool created is from a known implementation, this is not the case with `MstWrapper`. This wrapper can be built with a fake pool as well:

```solidity
function create(address mstWrapperImplementation, address pool, uint128 tick) external returns (address) {
    // ...
}
```

## Recommendation
Show only whitelisted `MstWrapper` on the user interface.

## Metastreet
Resolved in commit `4fe742d`.

## Cantina
This issue was fixed in the aforementioned commit. Now, pool legitimacy is checked via `PoolFactory`.

---
