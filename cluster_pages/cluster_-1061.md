# Cluster -1061

**Rank:** #349  
**Count:** 17  

## Label
Failure of external calls or predictable randomness leads to denial-of-service via unbounded minting or blocked execution, causing resource exhaustion or system lockout.

## Cluster Information
- **Total Findings:** 17

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

**Auto Label:** Loop-based gas explosions from unbounded iterations and external calls, leading to denial-of-service or transaction failure due to excessive gas consumption.  

**Original Text Preview:**

`runValidationHooks` and `runExecutionHooks` iterate through the registered `validationHooks` and `executionHooks` to perform the defined operations. However, these operations could run out of gas if there are too many registered hooks, potentially leading to unexpected reverts and a poor user experience. Consider restricting the number of registered hooks within the wallet.

---
