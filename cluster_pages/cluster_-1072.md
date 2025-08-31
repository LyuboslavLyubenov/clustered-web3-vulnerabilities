# Cluster -1072

**Rank:** #377  
**Count:** 14  

## Label
Failure to refund users upon order cancellation or amendment, leading to fund loss and broken financial accountability due to improper state updates and missing refund mechanisms.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** Failure to refund users upon order cancellation or amendment, leading to fund loss and broken financial accountability due to improper state updates and missing refund mechanisms.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-11-oku-judging/issues/331 

## Found by 
covey0x07, vinica\_boy
### Summary
When a user creates an order in the `OracleLess` contract, he can add a malicous token contract that reverts when tokens are transferred from the `OracleLess` contract. This order can't be canceled by admin. A malicious attacker can create this kind of order as many as he can to grife the protocol.

### Root Cause
At [OracleLess.sol#L38](https://github.com/sherlock-audit/2024-11-oku/blob/ee3f781a73d65e33fb452c9a44eb1337c5cfdbd6/oku-custom-order-types/contracts/automatedTrigger/OracleLess.sol#L38), there is no restrictions for `tokenIn`.
Any contract that implements `IERC20` can be `tokenIn`.

### Internal pre-conditions
N/A

### External pre-conditions
N/A

### Attack Path
- Alice creates a malicous token contract that reverts if token is transferred from the `OracleLess` contract.
- Alice creates orders by using this fake token contract.
- This order can't be cancelable as it reverts at [L160](https://github.com/sherlock-audit/2024-11-oku/blob/ee3f781a73d65e33fb452c9a44eb1337c5cfdbd6/oku-custom-order-types/contracts/automatedTrigger/OracleLess.sol#L160).
```solidity
    function _cancelOrder(Order memory order) internal returns (bool) { 
        //refund tokenIn amountIn to recipient
 @>     order.tokenIn.safeTransfer(order.recipient, order.amountIn);
```

### Impact
- A malicous attacker can grief the protocol by making a lot of uncancelable orders.
- All users of the protocol wastes significant gas in whenever they fill or cancel orders.

### Mitigation
It is recommended to add mechanism to whitelist tokens.

---
### Example 2

**Auto Label:** Failure to refund user funds during operation cancellation or failure, leading to permanent loss of user assets and undermining financial integrity and accountability.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-08-flayer-judging/issues/261 

## Found by 
0xAlix2, 0xHappy, 0xc0ffEE, Audinarey, Aymen0909, BADROBINX, BugPull, Hearmen, Limbooo, McToady, Ragnarok, almantare, araj, asui, blockchain555, cawfree, ctf\_sec, dany.armstrong90, g, merlinboii, onthehunt, steadyman, stuart\_the\_minion, utsav, ydlee, zzykxx
## Summary

When a user votes for collection shutdown, the `CollectionShutdown` contract gathers the whole balance from the user. However, when cancelling the shutdown process, the contract doesn't refund the user's votes.

## Vulnerability Detail

The [`CollectionShutdown::_vote()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L191-L214) function gathers the whole balance of the collection token from a voter.

```solidity
    function _vote(address _collection, CollectionShutdownParams memory params) internal returns (CollectionShutdownParams memory) {
        uint userVotes = params.collectionToken.balanceOf(msg.sender);
        if (userVotes == 0) revert UserHoldsNoTokens();

        // Pull our tokens in from the user
        params.collectionToken.transferFrom(msg.sender, address(this), userVotes);
        ... ...
    }
```

But in the [`CollectionShutdown::cancel()`](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L390-L405) function, it does not refund the voters tokens.

```solidity
    function cancel(address _collection) public whenNotPaused {
        // Ensure that the vote count has reached quorum
        CollectionShutdownParams memory params = _collectionParams[_collection];
        if (!params.canExecute) revert ShutdownNotReachedQuorum();

        // Check if the total supply has surpassed an amount of the initial required
        // total supply. This would indicate that a collection has grown since the
        // initial shutdown was triggered and could result in an unsuspected liquidation.
        if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** locker.collectionToken(_collection).denomination()) {
            revert InsufficientTotalSupplyToCancel();
        }

        // Remove our execution flag
        delete _collectionParams[_collection];
        emit CollectionShutdownCancelled(_collection);
    }
```

### Proof-Of-Concept

Here is the testcase of the POC:

To bypass the [total supply vs shutdown votes restriction](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L398-L400), added the following line to the test case:

```solidity
    collectionToken.mint(address(10), _additionalAmount);
```

The whole test case is:

```solidity
    function test_CancelShutdownNotRefund() public withQuorumCollection {
        uint256 _additionalAmount = 1 ether;
        // Confirm that we can execute with our quorum-ed collection
        assertCanExecute(address(erc721b), true);

        vm.prank(address(locker));
        collectionToken.mint(address(10), _additionalAmount); 

        // Cancel our shutdown
        collectionShutdown.cancel(address(erc721b));

        // Now that we have cancelled the shutdown process, we should no longer
        // be able to execute the shutdown.
        assertCanExecute(address(erc721b), false);

        console.log("Address 1 balance after:", collectionToken.balanceOf(address(1)));
        console.log("Address 2 balance after:", collectionToken.balanceOf(address(2)));
    }
```

Here are the logs after running the test:
```bash
$ forge test --match-test test_CancelShutdownNotRefund -vv
[⠒] Compiling...
[⠃] Compiling 1 files with Solc 0.8.26
[⠊] Solc 0.8.26 finished in 8.81s
Compiler run successful!

Ran 1 test for test/utils/CollectionShutdown.t.sol:CollectionShutdownTest
[PASS] test_CancelShutdownNotRefund() (gas: 390566)
Logs:
  Address 1 balance after: 0
  Address 2 balance after: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.29s (454.80µs CPU time)

Ran 1 test suite in 8.29s (8.29s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As can be seen from the logs, the voters(1, 2) were not refunded their tokens.

## Impact

Shutdown Voters will be ended up losing their whole collection tokens by cancelling the shutdown.

## Code Snippet

[utils/CollectionShutdown.sol#L390-L405](https://github.com/sherlock-audit/2024-08-flayer/blob/main/flayer/src/contracts/utils/CollectionShutdown.sol#L390-L405)

## Tool used

Manual Review

## Recommendation

The problem can be fixed by implementing following:

1. Add new state variable to the contract that records all voters

```solidity
    address[] public votersList;
```

2. Update the `_vote()` function like below:
```diff
    function _vote(address _collection, CollectionShutdownParams memory params) internal returns (CollectionShutdownParams memory) {
        ... ...
        // Register the amount of votes sent as a whole, and store them against the user
        params.shutdownVotes += uint96(userVotes);

        // Register the amount of votes for the collection against the user
+       if (shutdownVoters[_collection][msg.sender] == 0)
+           votersList.push(msg.sender);
        unchecked { shutdownVoters[_collection][msg.sender] += userVotes; }
        ... ...
    }
```

3. Add the new code section to the `reclaimVote()` function, that removes the sender from the `votersList`.

4. Update the `cancel()` function like below:
```diff
    function cancel(address _collection) public whenNotPaused {
        ... ...
        if (params.collectionToken.totalSupply() <= MAX_SHUTDOWN_TOKENS * 10 ** locker.collectionToken(_collection).denomination()) {
            revert InsufficientTotalSupplyToCancel();
        }

+       uint256 i;
+       uint256 votersLength = votersList.length;
+       for (; i < votersLength; i ++) {
+           params.collectionToken.transfer(
+               votersList[i], 
+               shutdownVoters[_collection][votersList[i]]
+           );
+       }

        // Remove our execution flag
        delete _collectionParams[_collection];
+       delete votersList;
        emit CollectionShutdownCancelled(_collection);
    }
```

After running the testcase on the above update, the user voters are able to get their own votes:

```bash
$ forge test --match-test test_CancelShutdownNotRefund -vv
[⠒] Compiling...
[⠘] Compiling 3 files with Solc 0.8.26
[⠃] Solc 0.8.26 finished in 8.70s
Compiler run successful!

Ran 1 test for test/utils/CollectionShutdown.t.sol:CollectionShutdownTest
[PASS] test_CancelShutdownNotRefund() (gas: 486318)
Logs:
  Address 1 balance after: 1000000000000000000
  Address 2 balance after: 1000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.46s (526.80µs CPU time)

Ran 1 test suite in 3.46s (3.46s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

---
### Example 3

**Auto Label:** Failure to refund users upon order cancellation or amendment, leading to fund loss and broken financial accountability due to improper state updates and missing refund mechanisms.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-05-elfi-protocol-judging/issues/50 

## Found by 
mstpr-brainbot, pashap9990
## Summary
When a position is decreased fully or partially, all the stop orders for that particular position will be canceled. Normally, the cancel order flow returns the execution fee paid by the user. However, this type of cancellation does not do that. As a result, all the STOP_LOSS and TAKE_PROFIT order execution fees are lost for the user.
## Vulnerability Detail
When a position is decreased partially or in full, the DecreasePositionProcess::decreasePosition function will remove all the hanging close orders with the following line:
```solidity
CancelOrderProcess.cancelStopOrders(
            cache.position.account,
            symbolProps.code,
            cache.position.marginToken,
            cache.position.isCrossMargin,
            CancelOrderProcess.CANCEL_ORDER_POSITION_CLOSE,
            params.requestId
        );
```

As we can observe in CancelOrderProcess::cancelStopOrders function the order removed from the accounts storage and global storage:
```solidity
function cancelStopOrders(
        address account,
        bytes32 symbol,
        address marginToken,
        bool isCrossMargin,
        bytes32 reasonCode,
        uint256 excludeOrder
    ) external {
        .
        .
            if (
                orderInfo.symbol == symbol &&
                orderInfo.marginToken == marginToken &&
                Order.Type.STOP == orderInfo.orderType &&
                orderInfo.isCrossMargin == isCrossMargin
            ) {
                -> accountProps.delOrder(orderIds[i]);
                -> orderPros.remove(orderIds[i]);
               .
            }
        }
    }
```

When the order is removed from storage user can no longer cancel it and get back the execution fee. All the stop loss and take profit orders execution fees are lost for the user.
## Impact
When user decides to close positions they will lose the execution fees. It can interpreted as user mistake however, If the account is liquidated than it can't be users mistake and the execution fees are lost regardless.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/CancelOrderProcess.sol#L64-L94
## Tool used

Manual Review

## Recommendation
Refund the execution fees as it's done in a normal cancel order



## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/0xCedar/elfi-perp-contracts/pull/22

---
