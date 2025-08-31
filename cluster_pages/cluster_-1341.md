# Cluster -1341

**Rank:** #402  
**Count:** 12  

## Label
Inadequate arithmetic validation and flawed logic in interest rate and parameter calculations lead to incorrect rate caps, negative values, or invalid state, enabling exploitable financial misbehavior.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Inadequate arithmetic validation and flawed logic in interest rate and parameter calculations lead to incorrect rate caps, negative values, or invalid state, enabling exploitable financial misbehavior.  

**Original Text Preview:**

## RateLimit Analysis

## Context
**File:** RateLimit.sol  
**Lines:** 73-76  

## Description
The `removeCaller` function only resets the callers mapping value for the removed caller. It does not change the `maxAllowances`, `allowances`, `allowancesLastSet`, and `intervals` states of the caller. This leads to some unintended scenarios:

1. The `allowanceCurrent` function still works for the removed caller.
2. The `estimatedAllowance` and `allowanceStored` functions return non-zero allowances for the removed caller.

Any external agent relying on these functions will observe incorrect properties for the caller.

## Recommendation
Consider resetting all allowance states for the caller who is getting removed.

## Acknowledgments
- **Coinbase:** Acknowledged.
- **Cantina Managed:** Acknowledged.

---
### Example 2

**Auto Label:** Inadequate arithmetic validation and flawed logic in interest rate and parameter calculations lead to incorrect rate caps, negative values, or invalid state, enabling exploitable financial misbehavior.  

**Original Text Preview:**

## Context
**File:** IRMSynth.sol  
**Lines:** L88-L94

## Description
In the IRMSynth documentation and EVK-78 specifications, it states that if the synth is trading at the target quote or below, the rate should be lowered by 10% (proportional to the previous rate). However, this does not match the code implementation, as the else part (decrease the rate) executes when the synth is trading at or above the target quote.

## Recommendation
It seems that the documentation statement is incorrect and should be updated as follows:

- "If the synth is trading below the target quote, raise the rate by 10% (proportional to the previous rate)."
- "If the synth is trading at or above the target quote, lower the rate by 10% (proportional to the previous rate)."

---
### Example 3

**Auto Label:** Improper access control leading to unauthorized manipulation of borrower balances and rates, enabling attackers to alter loan states and distort interest calculations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-02-tapioca-judging/issues/89 

## Found by 
0xadrii, bin2chen, ctf\_sec, hyh
## Summary

Balancer's `rebalance()` controls access rights by requesting `msg.sender` to simultaneously be owner and `rebalancer`, which blocks it whenever this role is assigned to any other address besides owner's (that should be the case for production use).

## Vulnerability Detail

Balancer's core operation can be blocked due to structuring of the access control check, which requires `msg.sender` to have both roles instead of either one of them.

## Impact

Rebalancing, which is core functionality for mTOFT workflow, becomes inaccessible once owner transfers the `rebalancer` role elsewhere. To unblock the functionality the role has to be returned to the owner address and kept there, so rebalancing will have to be performed only directly from owner, which brings in operational risks as keeper operations will have to be run from owner account permanently, which can be compromised with higher probability this way.

Also, there is an impact of having `rebalancer` role set to a keeper bot and being unable to perform the rebalancing for a while until protocol will have role reassigned and the scripts run from owner account. This additional time needed can be crucial for user operations and in some situations lead to loss of funds.

Likelihood: Low + Impact: High = Severity: Medium.

## Code Snippet

Initially `owner` and `rebalancer` are set to the same address:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L101-L110

```solidity
    constructor(address _routerETH, address _router, address _owner) {
        ...

        transferOwnership(_owner);
        rebalancer = _owner;
        emit RebalancerUpdated(address(0), _owner);
    }
```

Owner can then transfer `rebalancer` role to some other address, e.g. some keeper contract:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L142-L149

```solidity
    /**
     * @notice set rebalancer role
     * @param _addr the new address
     */
    function setRebalancer(address _addr) external onlyOwner {
 >>     rebalancer = _addr;
        emit RebalancerUpdated(rebalancer, _addr);
    }
```

Once owner transfers `rebalancer` role to anyone else, it will be impossible to rebalance as it's always `(msg.sender != owner() || msg.sender != rebalancer) == true`:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L160-L176

```solidity
    /**
     * @notice performs a rebalance operation
     * @dev callable only by the owner
     * @param _srcOft the source TOFT address
     * @param _dstChainId the destination LayerZero id
     * @param _slippage the destination LayerZero id
     * @param _amount the rebalanced amount
     * @param _ercData custom send data
     */
    function rebalance(
        address payable _srcOft,
        uint16 _dstChainId,
        uint256 _slippage,
        uint256 _amount,
        bytes memory _ercData
    ) external payable onlyValidDestination(_srcOft, _dstChainId) onlyValidSlippage(_slippage) {
>>      if (msg.sender != owner() || msg.sender != rebalancer) revert NotAuthorized();
```

## Tool used

Manual Review

## Recommendation

Consider updating the access control to allow either owner or `rebalancer`, e.g.:

https://github.com/sherlock-audit/2024-02-tapioca/blob/main/TapiocaZ/contracts/Balancer.sol#L169-L176

```diff
    function rebalance(
        ...
    ) external payable onlyValidDestination(_srcOft, _dstChainId) onlyValidSlippage(_slippage) {
-       if (msg.sender != owner() || msg.sender != rebalancer) revert NotAuthorized();
+       if (msg.sender != owner() && msg.sender != rebalancer) revert NotAuthorized();
```



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  the natspec says "callable only by the owner", which means the rebalancer role should be both owner and rebalancer which makes this invalid



**cryptotechmaker**

Low.
That was the initial intention. However, we'll fix it. 

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/Tapioca-DAO/TapiocaZ/pull/179.

**nevillehuang**

@cryptotechmaker Why was this the initial intention? I am inclined to keep medium severity given a direct code change was made to unblock DoS.

---
