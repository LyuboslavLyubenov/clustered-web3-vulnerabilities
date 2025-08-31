# Cluster -1292

**Rank:** #19  
**Count:** 733  

## Label
Unbounded loops and improper input validation lead to excessive gas consumption, infinite loops, or invalid state transitions, causing transaction failures, denial-of-service, or revenue loss.

## Cluster Information
- **Total Findings:** 733

## Examples

### Example 1

**Auto Label:** Unbounded loops and improper gas handling lead to excessive gas consumption, causing reverts, denial-of-service, or fund loss through failed transactions or incorrect gas accounting.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L601-L620>

<https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L708-L711>

<https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L750-L754>

When a user **stakes** in `StakingManager`, initially the funds go towards `hypeBuffer` and when it is filled, every deposit is placed in a L1 operation queue.
```

L1Operation[] private _pendingDeposits;
```

Once a day an operator calls `processL1Operations`. The amount of each deposit is delegated towards a validator and once the whole queue is processed it gets deleted.
```

if (_depositProcessingIndex == length) {
	delete _pendingDeposits;
	_depositProcessingIndex = 0;
}
```

The issue is that the whole array gets deleted, which can exceed the [block gas limit](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/dual-block-architecture) (30M gas) if the array is big enough. It is very unlikely that this situation happens on its own, because even the biggest staking protocol Lido has < 3k daily active users during its peak usages for the last 4 years.

However, an attacker can intentionally spam the queue with minimal deposits and cause a DoS. The scenario would be something like this:

1. Attacker takes out a (flash) loan of HYPE
2. Attacker stakes minimum amounts to flood the deposit queue and receives kHYPE in return
3. The attacker then sells the kHYPE and pays loan fees
4. When the `StakingManager` operator calls `processL1Operations` it will fail with out of gas error, because the amount of gas required to delete the array will be > 30M.

We should note that:

* `minStakeAmount` doesn’t stop the attack since HYPE is relatively cheap
* This DoS is done only via staking and is different form DoS caused by withdrawals.

  ##### Impact
* `StakingManager` operations will be disrupted, because processing the last element of the deposit queue will cause delete which will revert with OOG. Only allowing batches (max - 1) element to be processed.
* Last element will never deposit.
* Rebalancing and user deposits both will be affected.
* `resetL1OperationsQueue` will reach block gas limit and revert.

**Root Cause**

The whole pending deposit queue is deleted at once without the possibility of doing it partially.

### Proof of Concept

This PoC demonstrates how `processL1Operations` and `resetL1OperationsQueue` will revert with Out Of Gas when `_pendingDeposits` is too big.

Run `forge install foundry-rs/forge-std` to get latest APIs required for most accurate gas measurement.

We assume the following values for limits and price:
```

### (take note of 2 block types, one's limit is much less), the bigger is 30M
cast block --rpc-url https://rpc.hyperliquid.xyz/evm
HYPE price = 16.00$   # as time of writing
minStakeAmount = 0.1 HYPE
```

To make the `delete` exceed `30M` in gas, about `4480` deposits need to be made.
The amount necessary to execute the attack would be `minStakeAmount * 4480 * HYPE price` or ~`7 168`$ as of today.

Flash loan fees range from 0.09% to 0.3%.
If we assume the higher bound, that would be `21.5`$ in fees, making the attack quite affordable. Any griefer can afford this amount.

Place the code below under `StakingManager.t.sol`:
```

function test_DepositsDoS() public {
	// Set delegation targets for staking managers
	vm.startPrank(manager);
	validatorManager.activateValidator(validator);
	validatorManager.setDelegation(address(stakingManager), validator);

	// Set target buffer to 0
	stakingManager.setTargetBuffer(0);

	// Add many L1 operations in array
	uint256 stakeAmount = 0.1 ether;
	vm.deal(user, 10_000 ether);
	vm.startPrank(user);

	// 0.1 * 4480 (num deposits) * 16 (current HYPE price) = 7168 $ required
	for (uint256 i; i < 4480; ++i) { //
		stakingManager.stake{value: stakeAmount}();
	}

	// Try to process L1 operations
	// block gas limit: cast block --rpc-url https://rpc.hyperliquid.xyz/evm (take note of 2 block types, one's limit is much less)
	vm.startPrank(operator);
	vm.startSnapshotGas("processL1Operations");
	stakingManager.processL1Operations();
	uint256 gasUsed = vm.stopSnapshotGas();

	console.log("gasUsed", gasUsed);

	assertGt(gasUsed, 30_000_000);
}
```

### Recommended Mitigation Steps

Allow deleting `_pendingDeposits` in multiple transactions. On each call, make sure to check that all elements have been processed.

**Kinetiq disputed and commented:**

> We can use `processL1Operations(uint256 batchSize)` to batch process those queued operations, also we are able to reset them at once by using `resetL1OperationsQueue`.

---

---
### Example 2

**Auto Label:** Unbounded loops and improper gas handling lead to excessive gas consumption, causing reverts, denial-of-service, or fund loss through failed transactions or incorrect gas accounting.  

**Original Text Preview:**

When the `to` address is detected as a EIP-7702 EOA wallet by the [`_is7702DelegatedWallet`](https://github.com/across-protocol/contracts/blob/06b14cdfb83d01ceae65b6445a4e6d629686faea/contracts/SpokePool.sol#L1611) function when sending ETH or WETH, the `_unwrapwrappedNativeTokenTo` function from the `SpokePool` contract will first convert the WETH into ETH and then [send](https://github.com/across-protocol/contracts/blob/06b14cdfb83d01ceae65b6445a4e6d629686faea/contracts/SpokePool.sol#L1613) it to the wallet with a low level `.call` call. As such, this low level call does not limit the gas to 2300, being able to perform more complex operations.

Also, as the EIP-7702 EOA wallets could use any implementation, it might be possible that their `receive` or `fallback` method implement malicious logic to reenter the protocol when it is not expected.

Although several functions are protected against reentrancy, consider reducing the attack vector in the protocol by treating the EIP-7702 EOA wallets as contracts by sending WETH. Alternately, consider reducing the gas stipend for the low level `.call` call so it cannot perform any storage change nor external call to another contract.

***Update:** Acknowledged, not resolved. The team stated:*

> *The fill functionality already supports external calls with unlimited gas budgets at the same point in the code. From our perspective, any ETH call to a 7702 wallet could also trigger a callback into the contract via the `handleV3AcrossMessage` callback right after with no state changes in between. We cannot think of any cases where re-entrancy would be a problem in one case, but not the other.*
>
> *For the protocol to function with its current feature set, it must be resilient to re-entrancy, in general. If the protocol is not resilient to re-entrancy, then we think that should be addressed in a way that covers all cases, not just the ETH to 7702 case.*
>
> *We do not think these changes will impact the security of the protocol, so we acknowledge, but choose not to make the suggested changes.*

---
### Example 3

**Auto Label:** Unbounded loops and improper gas handling lead to excessive gas consumption, causing reverts, denial-of-service, or fund loss through failed transactions or incorrect gas accounting.  

**Original Text Preview:**

In `StrategyPassiveManagerKittenswap._chargeFees`, we call `safeTransfer` without checking that the values of `callFeeAmount`, `strategistFeeAmount`, and `beefyFeeAmount` are zero. Since we are sending a native token(WHYPE), sending zero will not cause the transaction to fail, but it will waste gas unnecessarily.

---
