# Cluster -1313

**Rank:** #76  
**Count:** 221  

## Label
Redundant array length reads in loops cause excessive gas costs due to repeated memory or storage accesses, leading to performance degradation and potential denial-of-service through high transaction costs.

## Cluster Information
- **Total Findings:** 221

## Examples

### Example 1

**Auto Label:** Gas optimization vulnerabilities through inefficient storage reads, redundant computations, and poor state management, leading to higher transaction costs and reduced scalability.  

**Original Text Preview:**

...

Export to GitHub ...

Set external GitHub Repo ...

Export to Clipboard (json)

Export to Clipboard (text)

#### Resolution

USDi team has acknowledged this finding with the following note:

> We don’t anticipate many small clients, rather few large clients (for the whitelist).

#### Description

The `scheduleWhitelist` function schedules a single address for whitelisting by setting its ready time and emitting a `WhitelistRequested` event. However, the function does not support batch processing, which can lead to higher gas costs when adding multiple users. This becomes a scalability limitation, as each user must be added in a separate transaction, each incurring a base cost of 21,000 gas, in addition to computation and storage costs.

#### Examples

**contracts/USDiCoin.sol:L248-L255**

```
/// @notice Initiates the whitelist schedule for an address; after `whitelistChangeDelay` has elapsed, address can transact
/// Now only a MANAGER or ADMIN can call this.
function scheduleWhitelist(address account) external onlyManagerOrAdmin {
    require(account != address(0), "Invalid address");
    uint256 readyTime = block.timestamp + whitelistChangeDelay;
    whitelistStartTime[account] = readyTime;
    emit WhitelistRequested(account, readyTime);
}

```

#### Recommendation

We recommend implementing a batch version of the `scheduleWhitelist` function to efficiently process multiple addresses in a single transaction. This will significantly reduce the overall gas cost and improve the scalability of the whitelisting mechanism.

---
### Example 2

**Auto Label:** Gas optimization vulnerabilities through inefficient storage reads, redundant computations, and poor state management, leading to higher transaction costs and reduced scalability.  

**Original Text Preview:**

##### Description

In the `CHNGovernance` contract the `queue()` function iterates over all of the contract targets of the proposal and then calls the `_queueOrRevert()` function in order to queue all of the functions that have to be executed later. However the length of the `proposal.targets` is not cached before all of the targets are looped trough. This increases the gas consumed by the `queue()` function. There is a similar scenario in the `execute()` function.

##### BVSS

[AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N (0.0)](/bvss?q=AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N)

##### Recommendation

Consider caching the length of the `proposal.targets` array before you loop over them.

##### Remediation Comment

**ACKNOWLEDGED:** The **Onyx team** has acknowledged the risk.

##### References

<https://etherscan.io/address/0xdec2f31c3984f3440540dc78ef21b1369d4ef767#code#L99>

<https://etherscan.io/address/0xdec2f31c3984f3440540dc78ef21b1369d4ef767#code#L115>

---
### Example 3

**Auto Label:** Gas optimization vulnerabilities through inefficient storage reads, redundant computations, and poor state management, leading to higher transaction costs and reduced scalability.  

**Original Text Preview:**

##### Description

In multiple functions across the codebase, the `for` loops use `i++` (post-increment) for incrementing the loop counter. In Solidity, post-increment (`i++`) is slightly less efficient than pre-increment (`++i`) because `i++` requires storing the original value of `i` in a temporary variable before incrementing, which consumes more gas. Although the gas difference is minimal, especially in recent Solidity versions, it becomes noticeable in larger loops or frequently executed functions, leading to inefficiencies in contract execution. The affected functions are the following:

* `StakeHub.init`
* `StakeHub.claimReward`
* `StakeHub.proxyClaimReward`
* `StakeHub.calculateReward`
* `SystemReward.receiveRewards`
* `SystemReward._checkPercentage`

##### BVSS

[AO:A/AC:L/AX:H/R:N/S:U/C:N/A:N/I:N/D:L/Y:N (0.8)](/bvss?q=AO:A/AC:L/AX:H/R:N/S:U/C:N/A:N/I:N/D:L/Y:N)

##### Recommendation

To optimize gas usage, especially when iterating over large arrays or loops, it is recommendd to replace `i++` with `++i`. Pre-increment (`++i`) does not require storing the old value of `i`, making it slightly more efficient in terms of gas consumption.

##### Remediation

**ACKNOWLEDGED:** The **CoreDAO team** acknowledged this issue and stated the following:

> *We will consider in some future releases.*

##### References

contracts/StakeHub.sol#L115, L233, L243, L259, L269, L290

contracts/SystemReward.sol#L68, L198

---
