# Cluster -1381

**Rank:** #232  
**Count:** 43  

## Label
Failure to properly track or update staking state over time, leading to incorrect balance calculations, reward misallocation, and potential user losses due to flawed state synchronization and lack of checkpoint protection.

## Cluster Information
- **Total Findings:** 43

## Examples

### Example 1

**Auto Label:** **Incorrect state updates due to flawed logic and missing boundaries, leading to erroneous balances, locked assets, and compromised protocol stability.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

- In `processWithdrawals()` function: when the `_withdrawShares = 0`, the `s.totalLiquidity` is increased by the liquidity of the collected-and-deposited fees of the position, where it assumes that the totalLiquidity of the position will not be decreased when `_reviseHedge()` is called:

```javascript
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
             s.totalLiquidity = _totalLiquidity;
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

Where:

```javascript
 function _reviseHedge(
        uint256 _totalLiquidity,
        bytes memory path0,
        bytes memory path1
    ) internal virtual {
        uint256 _newHedgeLiquidity = (_totalLiquidity * s.hedgeSize) / 1e18;

        uint256 _prevHedgeLiquidity = _loanData.liquidity;

        if (_newHedgeLiquidity > _prevHedgeLiquidity) {
            _increaseHedge(
                _newHedgeLiquidity - _prevHedgeLiquidity,
                path0,
                path1
            );
        } else if (_newHedgeLiquidity < _prevHedgeLiquidity) {
            _decreaseHedge(
                _prevHedgeLiquidity - _newHedgeLiquidity,
                path0,
                path1
            );
        }
    }
```

- Given that there's a setter for the `hedgeSize` (where it's used to calculate the `_newHedgeLiquidity`), then the `s.totalLiquidity` should be updated based on the liquidity position in case the `hedgeSize` was updated (also, in `_reviseHedge()`, the hedge liquidity is fetched based on the last updated data without checking if the hedge has accrued interest over time).

- So if the new `hedgeSize` is decreased and results in `_newHedgeLiquidity < _prevHedgeLiquidity` , then the hedge will be decreased, which might result in the hedge incurring losses, hence affecting the liquidity, which will affect the next process that's going to be executed after the withdrawal: if the next process is a deposit, then this incorrect `s.totalLiquidity` will result in minting wrong shares for that deposited period as the liquidity doesn't reflect the actual one (since it's the cashed one + liquidity from the collected-and-deposited fees of the LP position).

- Same issue in `DepositProcess.processDeposits()` function.

## Recommendations

```diff
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
-            s.totalLiquidity = _totalLiquidity;
+            s.totalLiquidity = _getTotalLiquidity(s.tokenId);
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

---
### Example 2

**Auto Label:** Insufficient state validation in unstaking flows leads to incorrect balance updates, silent reverts, and permanent loss of staked assets due to failed epoch and lock expiration checks.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

In the test files, HoneyLocker interacts with the `stakeLocked()` and `withdrawLockedAll()` functions in the Kodiak staking contract.

In the Kodiak staking contract, if `withdrawLockedAll()` is called but the expiration date is not up, the function will simply call `get_rewards()` and skip the `_withdrawLocked()`.

```
function withdrawLockedAll() nonReentrant withdrawalsNotPaused public {
>       _getReward(msg.sender, msg.sender);
        LockedStake[] memory locks = lockedStakes[msg.sender];
        for(uint256 i = 0; i < locks.length; i++) {
>           if(locks[i].liquidity > 0 && block.timestamp >= locks[i].ending_timestamp){
                _withdrawLocked(msg.sender, msg.sender, locks[i].kek_id, false);
            }
        }
    }
```

In the HoneyLocker.unstake function, the `staked[_LPToken][_stakingContract]` value will be subtracted.

```
  function unstake(address _LPToken, address _stakingContract, uint256 _amount, bytes memory _data)
        public
        onlyOwner
        onlyAllowedTargetContract(_stakingContract)
        onlyAllowedSelector(_stakingContract, "unstake", _data)
    {
>       staked[_LPToken][_stakingContract] -= _amount;
        (bool success,) = _stakingContract.call(_data);
        if (!success) revert UnstakeFailed();

        emit Unstaked(_stakingContract, _LPToken, _amount);
    }
```

Let's say the owner stakes 100 LP tokens for 30 days, and `staked[_LPToken][_stakingContract]` is 100e18.

- The 100 LP tokens is now inside Kodiak staking.
- 15 days in, the owner calls `unstake()` with `withdrawLockedAll()` as the function selector.
- The function will collect KDK rewards, but the LP tokens will not be sent back to the HoneyLocker as the expiration is not up.
- In the `HoneyLocker.unstake()` function, `staked[_LPToken][_stakingContract]` will now be 0 but LP tokens is still in Kodiak contract.
- 15 days later, the owner calls `unstake()` again to withdraw the LP tokens.
- The `unstake()` function will revert with underflow because it attempts to deduct from `staked[_LPToken][_stakingContract]`.

The 100 LP tokens will be stuck in the Kodiak staking contract.

## Recommendations

```diff
-   staked[_LPToken][_stakingContract] -= _amount;
+   if (_amount >= staked[_LPToken][_stakingContract]) {
+       staked[_LPToken][_stakingContract] -= _amount;
+   }
```

---
### Example 3

**Auto Label:** Insufficient state validation in unstaking flows leads to incorrect balance updates, silent reverts, and permanent loss of staked assets due to failed epoch and lock expiration checks.  

**Original Text Preview:**

## Summary

The `Fjord` protocol team allows for users to stake their Sablier streams and make additional gains on their Sablier NFTs. However, whenever a Sablier stream sender decides to `cancel()` a recipient's stream, the staked position of the recipient in `FjordStaking` will remain active due to a revert in the `FjordStaking::onStreamCanceled(...)` hook callback. When a stream is canceled, the Fjord staking contract tries to unstake the recipient's position by invoking `_unstakeVested(...)`, however, the unstaking function incorrectly passes the `msg.sender` to the `FjordPoints::onUnstaked(...)` function, which in the event of a canceled stream will be the Sablier contract address, instead of the Sablier stream recipient.

## Vulnerability Details

As per the [Sablier Docs](https://docs.sablier.com/concepts/protocol/cancelability):

> When creating a Sablier stream, users have the ability to set the stream as cancelable, or uncancelable. If cancelable, the stream can be stopped at any time by the stream creator, with the unstreamed funds being returned over to the stream creator.

From the above, we can see that a recipient's stream can be canceled anytime, and their funds stopped while doing so. This is why Sablier promotes the use of special hooks, allowing stream recipients to execute required actions and update their states accordingly. `Fjord` does this as well by inheriting from the `ISablierV2LockupRecipient` interface and including the `onStreamCanceled` hook. When a stream is canceled, `Fjord` tries to unstake the recipient's position and process any rewards applicable up to the cancelation. However, with the current configuration, the unstaking will fail silently, as the protocol passes the incorrect address to the `FjordPoints::onUnstaked(...)` function.

This issue is possible because Sablier stream cancelations will not revert if the `onStreamCanceled(...)` hook reverts:

```solidity
function _cancel(uint256 streamId) internal override {
__SNIP__
        // Interactions: if `msg.sender` is the sender and the recipient is a contract, try to invoke the cancel
        // hook on the recipient without reverting if the hook is not implemented, and without bubbling up any
        // potential revert.
        if (msg.sender == sender) {
            if (recipient.code.length > 0) {
                try ISablierV2LockupRecipient(recipient).onStreamCanceled({
                    streamId: streamId,
                    sender: sender,
                    senderAmount: senderAmount,
                    recipientAmount: recipientAmount
                }) { } catch { }
            }
        }
        // Interactions: if `msg.sender` is the recipient and the sender is a contract, try to invoke the cancel
        // hook on the sender without reverting if the hook is not implemented, and also without bubbling up any
        // potential revert.
        else {
            if (sender.code.length > 0) {
                try ISablierV2LockupSender(sender).onStreamCanceled({
                    streamId: streamId,
                    recipient: recipient,
                    senderAmount: senderAmount,
                    recipientAmount: recipientAmount
                }) { } catch { }
            }
        }
__SNIP__
    }
```

As seen from the code above, if a user stakes his/her position in `FjordStaking`, and the Sablier stream sender decides to cancel it, any reverts that happen in `FjordStaking::onStreamCanceled(...)` will be silenced.

When the cancelation hook is processed in `FjordStaking`, the contract correctly passes the `streamOwner` address to the [\_unstakeVested(...)](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordStaking.sol#L840) function:

```solidity
function onStreamCanceled(
        uint256 streamId,
        address sender,
        uint128 senderAmount,
        uint128 /*recipientAmount*/
    ) external override onlySablier checkEpochRollover {
        address streamOwner = _streamIDOwners[streamId];
__SNIP__

@>        _unstakeVested(streamOwner, streamId, amount); // Stream owner is correctly propagated

        emit SablierCanceled(streamOwner, streamId, sender, amount);
    }
```

However, later in that function, instead of passing the `streamOwner`, the protocol incorrectly passes `msg.sender` to the `FjordPoints::onUnstaked(...)` function, which will fail as it tries to unstake from an unexistent position.

```solidity
 function _unstakeVested(address streamOwner, uint256 _streamID, uint256 amount) internal {
__SNIP__
@>        points.onUnstaked(msg.sender, amount); // incorrectly passing msg.sender

        emit VestedUnstaked(streamOwner, epoch, amount, _streamID);
    }
```

This, in turn, will revert, as the [onUnstaked(...)](https://github.com/Cyfrin/2024-08-fjord/blob/0312fa9dca29fa7ed9fc432fdcd05545b736575d/src/FjordPoints.sol#L214) function will throw `UnstakingAmountExceedsStakedAmount` error, and the whole unstaking flow will revert (but the stream cancelation will not). The canceled recipient's position in the `FjordStaking` contract will remain intact, allowing the user to later unstake and claim rewards for the full amount of his/her stake, even though there are no active funds in his/her Sablier stream.

## Impact

Canceled vested stakers will incorrectly receive rewards for staked funds that they do not own. What is worse they can also claim `FjordPoints` which they can use in the auctions and benefit even more.

## Proof of concept

1. Alice and Bob have Sablier streams containing FjordTokens.
2. They both stake them in the `FjordStaking` contract.
3. Alice's stream gets canceled shortly after the stake, as she gets fired.
4. The Fjord protocol tries to unstake Alice's position upon receiving the `onStreamCanceled(...)` hook from Sablier, but fails to do so.
5. Alice patiently waits for the unstaking cycle, unstakes, and afterwards claims her rewards as if she owned the funds for the whole cycle.

<details>

<summary> Coded PoC </summary>

_I have created a separate testing suit, as the protocol team's one used improper Mocks, which do not invoke the erroneous flow described above_

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import { Test, console } from "forge-std/Test.sol";
import { FjordStaking } from "../src/FjordStaking.sol";
import { FjordPoints } from "../src/FjordPoints.sol";
import { FjordToken } from "../src/FjordToken.sol";
import { ISablierV2LockupLinear } from "lib/v2-core/src/interfaces/ISablierV2LockupLinear.sol";
import { ISablierV2Lockup } from "lib/v2-core/src/interfaces/ISablierV2Lockup.sol";
import { Broker, LockupLinear } from "lib/v2-core/src/types/DataTypes.sol";
import { ud60x18 } from "@prb/math/src/UD60x18.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FjordAuditTests is Test {
    FjordStaking fjordStaking;
    FjordToken token;
    address minter = makeAddr("minter");
    address newMinter = makeAddr("new_minter");
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address internal constant SABLIER_ADDRESS = address(0xB10daee1FCF62243aE27776D7a92D39dC8740f95);
    address points;
    bool isMock = false;

    ISablierV2LockupLinear SABLIER = ISablierV2LockupLinear(SABLIER_ADDRESS);
    address authorizedSender = address(this);

    function setUp() public {
        token = new FjordToken();
        points = address(new FjordPoints());

        vm.createSelectFork({ urlOrAlias: "mainnet", blockNumber: 19_595_905 });

        fjordStaking =
            new FjordStaking(address(token), minter, SABLIER_ADDRESS, authorizedSender, points);

        FjordPoints(points).setStakingContract(address(fjordStaking));

        deal(address(token), address(this), 10_000 ether);
        token.approve(address(fjordStaking), 10_000 ether);

        deal(address(token), minter, 10_000 ether);
        vm.prank(minter);
        token.approve(address(fjordStaking), 10_000 ether);

        deal(address(token), alice, 10_000 ether);
        vm.prank(alice);
        token.approve(address(fjordStaking), 10_000 ether);

        deal(address(token), bob, 10_000 ether);
        vm.prank(bob);
        token.approve(address(fjordStaking), 10_000 ether);
    }

    function createStream(
        address sender,
        address recipient,
        FjordToken asset,
        bool isCancelable,
        uint256 amount
    ) internal returns (uint256 streamID) {
        deal(address(asset), sender, amount);
        vm.prank(sender);
        asset.approve(address(SABLIER), amount);

        LockupLinear.CreateWithRange memory params;

        params.sender = sender;
        params.recipient = recipient;
        params.totalAmount = uint128(amount);
        params.asset = IERC20(address(asset));
        params.cancelable = isCancelable;
        params.range = LockupLinear.Range({
            start: uint40(vm.getBlockTimestamp() + 1 days),
            cliff: uint40(vm.getBlockTimestamp() + 2 days),
            end: uint40(vm.getBlockTimestamp() + 11 days)
        });
        params.broker = Broker(address(0), ud60x18(0));

        assertEq(asset.balanceOf(sender), amount);
        vm.prank(sender);
        streamID = SABLIER.createWithRange(params);
        assertEq(asset.balanceOf(sender), 0);
    }

    function testSablierCancleFails() public {
        console.log("Alice's balance before stake: %d", token.balanceOf(address(alice)));
        console.log("Bob's balance before stake: %d", token.balanceOf(address(bob)));
        uint256 aliceStream = createStream(address(this), alice, token, true, 100 ether);
        uint256 bobStream = createStream(address(this), bob, token, true, 100 ether);

        vm.prank(alice);
        SABLIER.approve(address(fjordStaking), aliceStream);

        vm.prank(bob);
        SABLIER.approve(address(fjordStaking), bobStream);

        vm.prank(alice);
        fjordStaking.stakeVested(aliceStream);

        vm.prank(bob);
        fjordStaking.stakeVested(bobStream);

        console.log(
            "Sablier sender balance before Alice's cancel: %d", token.balanceOf(address(this))
        );
        assertEq(token.balanceOf(address(this)), 0);
        SABLIER.cancel(aliceStream); // fails silently
        console.log(
            "Sablier sender balance after Alice's cancel: %d", token.balanceOf(address(this))
        );
        assertEq(token.balanceOf(address(this)), 100 ether); // The stream is canceled and the sender retrieves his funds

        // However due to the silent revert in the contract, Alice's stake is not removed
        // And Alice will still be able to unstake and claim

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.prank(minter);
        fjordStaking.addReward(1 ether);

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.prank(alice);
        fjordStaking.unstakeVested(aliceStream);

        vm.prank(bob);
        fjordStaking.unstakeVested(bobStream);

        vm.prank(alice);
        fjordStaking.claimReward(false);

        vm.prank(bob);
        fjordStaking.claimReward(false);

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.warp(vm.getBlockTimestamp() + fjordStaking.epochDuration());

        vm.prank(alice);
        fjordStaking.completeClaimRequest();

        vm.prank(bob);
        fjordStaking.completeClaimRequest();

        console.log("Alice's balance after stake: %d", token.balanceOf(address(alice)));
        console.log("Bob's balance after stake: %d", token.balanceOf(address(bob)));

        assertEq(token.balanceOf(address(alice)), token.balanceOf(address(bob))); // Alice receives the same reward as Bob, even though she has no actual funds
    }
}

```

```bash

├─ [97410] 0xB10daee1FCF62243aE27776D7a92D39dC8740f95::cancel(1402)
    │   ├─ [2906] FjordToken::transfer(FjordAuditTests: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 100000000000000000000 [1e20])
    │   │   ├─ emit Transfer(from: 0xB10daee1FCF62243aE27776D7a92D39dC8740f95, to: FjordAuditTests: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], value: 100000000000000000000 [1e20])
    │   │   └─ ← [Return] true
    │   ├─ [65699] FjordStaking::onStreamCanceled(1402, FjordAuditTests: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 100000000000000000000 [1e20], 0)
    │   │   ├─ [24515] 0xB10daee1FCF62243aE27776D7a92D39dC8740f95::transferFrom(FjordStaking: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], 1402)
    │   │   │   ├─ emit Transfer(from: FjordStaking: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], to: alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], tokenId: 1402)
    │   │   │   └─ ← [Return] 
    │   │   ├─ [28265] FjordPoints::onUnstaked(0xB10daee1FCF62243aE27776D7a92D39dC8740f95, 100000000000000000000 [1e20])
    │   │   │   └─ ← [Revert] UnstakingAmountExceedsStakedAmount()
    │   │   └─ ← [Revert] UnstakingAmountExceedsStakedAmount()
    │   ├─ emit CancelLockupStream(streamId: 1402, sender: FjordAuditTests: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], recipient: FjordStaking: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], senderAmount: 100000000000000000000 [1e20], recipientAmount: 0)
    │   ├─ emit MetadataUpdate(: 1402)

Ran 1 test for test/FjordAuditTests.t.sol:FjordAuditTests
[PASS] testSablierCancleFails() (gas: 1788195)
Logs:
  Alice's balance before stake: 10000000000000000000000
  Bob's balance before stake: 10000000000000000000000
  Sablier sender balance before Alice's cancel: 0
  Sablier sender balance after Alice's cancel: 100000000000000000000
  Alice's balance after stake: 10000500000000000000000
  Bob's balance after stake: 10000500000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.03s (7.12ms CPU time)

Ran 1 test suite in 1.04s (1.03s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>

## Tools Used

Manual review

## Recommendations

Pass the `streamOwner` address instead of the `msg.sender` to the `onUnstaked(...)` function in the `FjordStaking.sol` contract:

```diff
diff --git a/src/FjordStaking.sol b/src/FjordStaking.sol
index 46e93c4..aac894f 100644
--- a/src/FjordStaking.sol
+++ b/src/FjordStaking.sol
@@ -558,7 +558,7 @@ contract FjordStaking is ISablierV2LockupRecipient {
             sablier.transferFrom({ from: address(this), to: streamOwner, tokenId: _streamID });
         }
 
-        points.onUnstaked(msg.sender, amount);
+        points.onUnstaked(streamOwner, amount);
 
         emit VestedUnstaked(streamOwner, epoch, amount, _streamID);
     }
```

---
