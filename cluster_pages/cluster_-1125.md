# Cluster -1125

**Rank:** #429  
**Count:** 9  

## Label
Insufficient state validation and incorrect power calculations allow users to manipulate voting power and locked supply, bypassing caps and distorting governance incentives through biased, time-ignoring, or double-counted updates.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Insufficient state validation and incorrect power calculations allow users to manipulate voting power and locked supply, bypassing caps and distorting governance incentives through biased, time-ignoring, or double-counted updates.  

**Original Text Preview:**

## Summary

The veRAACToken::increase function contains a critical vulnerability in the calculation of voting power (bias) when users increase their locked token amount. The issue arises because the function does not account for the decay of the user's existing voting power over time. Instead, it calculates the new bias based on the original locked amount, ignoring the fact that the user's voting power has decreased due to decay.

This flaw allows users to exploit the system by:

Initially locking a small amount of tokens.

Waiting for their voting power to decay partially.

Increasing their locked amount to gain disproportionately higher voting power compared to users who lock the same total amount upfront.

## Vulnerability Details

veRAACToken::increase allows a user with an existing lock to increase the amount they have locked. See below:

```solidity
 /**
     * @notice Increases the amount of locked RAAC tokens
     * @dev Adds more tokens to an existing lock without changing the unlock time
     * @param amount The additional amount of RAAC tokens to lock
     */
    function increase(uint256 amount) external nonReentrant whenNotPaused {
        // Increase lock using LockManager
        _lockState.increaseLock(msg.sender, amount);
        _updateBoostState(msg.sender, locks[msg.sender].amount);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
            msg.sender,
            userLock.amount + amount,
            userLock.end
        );

        // Update checkpoints
        uint256 newPower = uint256(uint128(newBias));
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Transfer additional tokens and mint veTokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);
        _mint(msg.sender, newPower - balanceOf(msg.sender));

        emit LockIncreased(msg.sender, amount);
    }

```

The key line to note is where the newBias is calculated with:

```solidity
 (int128 newBias, int128 newSlope) = _votingState
            .calculateAndUpdatePower(
                msg.sender,
                userLock.amount + amount,
                userLock.end
            );
```

VotingPowerLib::calculateAndUpdatePower updates the user's bias as follows:

```solidity
 /**
     * @notice Calculates and updates voting power for a user
     * @dev Updates points and slope changes for power decay
     * @param state The voting power state
     * @param user The user address
     * @param amount The amount of tokens
     * @param unlockTime The unlock timestamp
     * @return bias The calculated voting power bias
     * @return slope The calculated voting power slope
     */
    function calculateAndUpdatePower(
        VotingPowerState storage state,
        address user,
        uint256 amount,
        uint256 unlockTime
    ) internal returns (int128 bias, int128 slope) {
        if (amount == 0 || unlockTime <= block.timestamp) revert InvalidPowerParameters();
        
        uint256 MAX_LOCK_DURATION = 1460 days; // 4 years
        // FIXME: Get me to uncomment me when able
        // bias = RAACVoting.calculateBias(amount, unlockTime, block.timestamp);
        // slope = RAACVoting.calculateSlope(amount);

        // Calculate initial voting power that will decay linearly to 0 at unlock time
        uint256 duration = unlockTime - block.timestamp;
        uint256 initialPower = (amount * duration) / MAX_LOCK_DURATION; // Normalize by max duration
        
        bias = int128(int256(initialPower));
        slope = int128(int256(initialPower / duration)); // Power per second decay
        
        uint256 oldPower = getCurrentPower(state, user, block.timestamp);
        
        state.points[user] = RAACVoting.Point({
            bias: bias,
            slope: slope,
            timestamp: block.timestamp
        });

        _updateSlopeChanges(state, unlockTime, 0, slope);
        
        emit VotingPowerUpdated(user, oldPower, uint256(uint128(bias)));
        return (bias, slope);
    }

```

The crease function passes userLock.amount + amount. The idea is that we calculate a new bias value based on the amount that the user wants to deposit and the last amount the user deposited. Then the new bias gotten from there using the above function.

The error occurs in the calculation of the new bias. Since veRAACToken::increase uses userLock.amount, it assumes the initial value that the user deposited into the protocol. This doesn't take the rate of decay into account. The idea of ve mechanics is that the tokens lose value over time as their power tends towards 0. So when a user increases their locked amount, the previous balance they deposited should have been decaying over time and the amount that should be considered when a user increases their lock is the current amount factoring decay + new amount user wants to deposit.  In the current implementation, the rate of decay is not considered which can lead to a host of issues including that a user could opt to initially lock a small amount and come back and increase the lock by a larger amount and get more voting power than a user who just locked the same amount at once using veRAACToken::lock.

\##Proof Of Code (POC)

The following tests were run in the veRAACToken.test.js file in the "Lock Mechanism" describe block.

```javascript
 it("new bias skew in increase function", async () => {
      //c for testing purposes
      const initialAmount = ethers.parseEther("1000");
      const additionalAmount = ethers.parseEther("500");
      const duration = 365 * 24 * 3600; // 1 year

      await veRAACToken.connect(users[0]).lock(initialAmount, duration);

      //c wait for some time to pass so the user's biases to change
      await time.increase(duration / 2); //c half a year has passed

      //c get user biases
      const user0BiasPreIncrease = await veRAACToken.getVotingPower(
        users[0].address
      );
      console.log("User 0 Bias: ", user0BiasPreIncrease);

      //c calculate expected bias after increase taking the rate of decay of user's tokens into account
      const amount = user0BiasPreIncrease + additionalAmount;
      console.log("Amount: ", amount);

      const positionPreIncrease = await veRAACToken.getLockPosition(
        users[0].address
      );
      const positionEndTime = positionPreIncrease.end;
      console.log("Position End Time: ", positionEndTime);

      const currentTimestamp = await time.latest();
      const duration1 = positionEndTime - BigInt(currentTimestamp);
      console.log("Duration 1: ", duration1);

      const user0expectedBiasAfterIncrease =
        (amount * duration1) / (await veRAACToken.MAX_LOCK_DURATION());
      console.log(
        "Expected Bias After Increase: ",
        user0expectedBiasAfterIncrease
      );

      await veRAACToken.connect(users[0]).increase(additionalAmount);

      //c get actual user biases after increase
      const user0actualBiasPostIncrease = await veRAACToken.getVotingPower(
        users[0].address
      );
      console.log("User 0 Post Increase Bias: ", user0actualBiasPostIncrease);

      //c since the rate of decay was not taken into account when calculating the user's bias, the expected bias after increase will be less than the actual bias after increase which gives the user more voting power than expected which can be used to skew voting results and reward distribution
      assert(user0expectedBiasAfterIncrease < user0actualBiasPostIncrease);

      const position = await veRAACToken.getLockPosition(users[0].address);
      expect(position.amount).to.equal(initialAmount + additionalAmount);
    });

    it("2 users locking same amount with different methods end up with different voting power", async () => {
      //c for testing purposes
      const initialAmount = ethers.parseEther("1000");

      const duration = 365 * 24 * 3600; // 1 year

      //c user0 locks tokens using the lock function
      await veRAACToken.connect(users[0]).lock(initialAmount, duration);

      //c user 1 knows about the exploit and deposits only half the amount of user 0 for same duration
      await veRAACToken
        .connect(users[1])
        .lock(BigInt(initialAmount) / BigInt(2), duration);

      //c wait for some time to pass so the user's biases to change
      await time.increase(duration / 2); //c half a year has passed

      //c user1 waits half a year and then deposits the next half of the amount without adjusting their lock duration
      await veRAACToken
        .connect(users[1])
        .increase(BigInt(initialAmount) / BigInt(2));

      //c get user biases
      const user0BiasPostIncrease = await veRAACToken.getVotingPower(
        users[0].address
      );
      console.log("User 0 Bias: ", user0BiasPostIncrease);

      const user1BiasPostIncrease = await veRAACToken.getVotingPower(
        users[1].address
      );
      console.log("User 1 Bias: ", user1BiasPostIncrease);

      //c at this point, user0 and user1 should have the same voting power since they have the same amount of tokens locked for the same duration but this isnt the case due to the bug in the above test and user1 will end up with more tokens than user 0
      assert(user0BiasPostIncrease < user1BiasPostIncrease);
    });

```

## Impact

The vulnerability in the veRAACToken::increase function has significant implications for the fairness and integrity of the protocol. Specifically:

Unfair Voting Power Distribution: Users who exploit this vulnerability by initially locking a small amount and later increasing it can gain disproportionately higher voting power compared to users who lock the same total amount upfront. This skews voting outcomes and undermines the fairness of governance decisions.

Reward Distribution Manipulation:  Exploiting this bug allows users to manipulate their voting power to claim a larger share of rewards than they are entitled to.

Economic Imbalance: The exploit could lead to an imbalance in the protocol's tokenomics, as users who exploit the bug gain an unfair advantage over honest participants.

## Tools Used

Manual Review, Hardhat

## Recommendations

Update the increase Function: Modify the increase function to pass the current decayed balance of the user's existing lock to calculateAndUpdatePower.

Update the increase function to:

```solidity
function increase(uint256 amount) external nonReentrant whenNotPaused {
    // Increase lock using LockManager
    _lockState.increaseLock(msg.sender, amount);
    _updateBoostState(msg.sender, locks[msg.sender].amount);

    // Get the current decayed balance of the user's existing lock
    uint256 currentBalance = getCurrentPower(_votingState, msg.sender, block.timestamp);

    // Update voting power
    LockManager.Lock memory userLock = _lockState.locks[msg.sender];
    (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
        msg.sender,
        currentBalance + amount, // Use current decayed balance + new amount
        userLock.end
    );

    // Update checkpoints
    uint256 newPower = uint256(uint128(newBias));
    _checkpointState.writeCheckpoint(msg.sender, newPower);

    // Transfer additional tokens and mint veTokens
    raacToken.safeTransferFrom(msg.sender, address(this), amount);
    _mint(msg.sender, newPower - balanceOf(msg.sender));

    emit LockIncreased(msg.sender, amount);
}
```

---
### Example 2

**Auto Label:** Insufficient state validation and incorrect power calculations allow users to manipulate voting power and locked supply, bypassing caps and distorting governance incentives through biased, time-ignoring, or double-counted updates.  

**Original Text Preview:**

## Summary

Incorrect bias update in `veRAACToken.increase` method allows a malicious user receive more voting power and veToken than deserved by locking and then increasing.

## Vulnerability Details

### Root Cause Analysis

Users can lock raacToken for voting power and veRAACToken. 

veRAACToken minted amount represents initial voting power, and it is calculated as the following [ref](https://github.com/Cyfrin/2025-02-raac/blob/fe17cc3fee28e04fd45e5102167e3642a193b2ca/contracts/libraries/governance/VotingPowerLib.sol#L90): 

```Solidity
power = lockAmount * duration / MAX_LOCK_DURATION
```

However, when users [increase their position](https://github.com/Cyfrin/2025-02-raac/blob/fe17cc3fee28e04fd45e5102167e3642a193b2ca/contracts/core/tokens/veRAACToken.sol#L252-L262), total lock amount is calculated incorrectly:

```solidity
    // Increase lock using LockManager
@>  _lockState.increaseLock(msg.sender, amount); // @audit amount is already added
    _updateBoostState(msg.sender, locks[msg.sender].amount);

    // Update voting power
    LockManager.Lock memory userLock = _lockState.locks[msg.sender];
    (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
        msg.sender,
@>      userLock.amount + amount, // @audit new amount is added again
        userLock.end
    );
```

&#x20;

As we can see in the comment, `userLock.amount` is already incremented by increased amount. But increased amount is added to `userLock.amount` again when calculating new bias.

So the following thing happens on increase:

```Solidity
newBias = (existingLockAmount + increasedAmount * 2) * duration / MAX_LOCK_DURATION
```

This means that, for given raacToken amount, if you keep `existingLockAmount` to almost zero, and keep `increasedAmount` to total raacToken amount you have, you will mint almost double veToken and voting power.

Consider the following scenario:

* Eve has 100 raacToken
* Eve locks dust amount `1e-18` raccToken to create an initial position
* Eve immediately increases the position by locking the rest of raacToken
* Eve will receive around 200 veToken and 200 voting power

### POC

**Scenario**

POC scenario is similar to the one in the Vulnerability Details section.

To illustrate the vulnerabilty more clearly, POC has two actors: alice and eve

* `alice` locks 100 raacToken and receives 100 veToken and voting power
* `eve` exploits the vulnerability and mints nearly 200 veToken and voting power with the same amount of raacToken

**How to run the POC**

* First, follow the steps in [hardhat-foundry integration tutorial](https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry#adding-foundry-to-a-hardhat-project)
* Then, create a file `test/poc.t.sol` and put the following content and run `forge test poc.t.sol -vvv`:

```solidity
pragma solidity ^0.8.19;

import "../lib/forge-std/src/Test.sol";
import {veRAACToken} from "../contracts/core/tokens/veRAACToken.sol";
import {RAACToken} from "../contracts/core/tokens/RAACToken.sol";

contract veRAACTokenTest is Test {
    RAACToken raacToken;
    veRAACToken veToken;

    address alice = makeAddr("alice");
    address eve = makeAddr("eve");

    uint256 raacTokenAmount = 100e18;

    function setUp() external {
        raacToken = new RAACToken(address(this), 0, 0);
        raacToken.setMinter(address(this));
        veToken = new veRAACToken(address(raacToken));
        veToken.setMinter(address(this));
    }

    function testLockAndIncrease() external {
        // alice just locks 100 raacToken
        _userLocks(alice, raacTokenAmount, veToken.MAX_LOCK_DURATION());
        emit log_named_decimal_uint("alice voting power", veToken.getVotingPower(alice), 18);
        emit log_named_decimal_uint("alice veToken balance", veToken.balanceOf(alice), 18);

        // eve first locks dust amount
        _userLocks(eve, 1, veToken.MAX_LOCK_DURATION());
        // eve then increases the lock position by using rest of the raacToken
        _userIncreases(eve, raacTokenAmount - 1);
        emit log_named_decimal_uint("eve voting power", veToken.getVotingPower(eve), 18);
        emit log_named_decimal_uint("eve veToken balance", veToken.balanceOf(eve), 18);

        // eve has nearly double voting power and veToken balance as alice
        assertGt(veToken.getVotingPower(eve), veToken.getVotingPower(alice));
        assertGt(veToken.balanceOf(eve), veToken.balanceOf(alice));
    }

    function _userLocks(address user, uint256 amount, uint256 duration) internal {
        raacToken.mint(user, amount);
        vm.startPrank(user);
        raacToken.approve(address(veToken), amount);
        veToken.lock(amount, duration);
        vm.stopPrank();
    }

    function _userIncreases(address user, uint256 amount) internal {
        raacToken.mint(user, amount);
        vm.startPrank(user);
        raacToken.approve(address(veToken), amount);
        veToken.increase(amount);
        vm.stopPrank();
    }
}

```

Console Output

```console
[PASS] testLockAndIncrease() (gas: 907293)
Logs:
  alice voting power: 100.000000000000000000
  alice veToken balance: 100.000000000000000000
  eve voting power: 199.999999999999999999
  eve veToken balance: 199.999999999999999999
```

## Impact

* Attackers can have double voting power and veToken balance for given fund
* Increase-and-extend will grant more power and veToken than extend-and-increase, because increase-and-extend will inflate incresed amount by multiple of extended duration

## Tools Used

Manaual Review, Foundry

## Recommendations

The following diff will solve the problem:

```diff
diff --git a/contracts/core/tokens/veRAACToken.sol b/contracts/core/tokens/veRAACToken.sol
index 2dbeefc..f60aca9 100644
--- a/contracts/core/tokens/veRAACToken.sol
+++ b/contracts/core/tokens/veRAACToken.sol
@@ -257,7 +257,7 @@ contract veRAACToken is ERC20, Ownable, ReentrancyGuard, IveRAACToken {
         LockManager.Lock memory userLock = _lockState.locks[msg.sender];
         (int128 newBias, int128 newSlope) = _votingState.calculateAndUpdatePower(
             msg.sender,
-            userLock.amount + amount,
+            userLock.amount,
             userLock.end
         );

```

---
### Example 3

**Auto Label:** Insufficient state validation and incorrect power calculations allow users to manipulate voting power and locked supply, bypassing caps and distorting governance incentives through biased, time-ignoring, or double-counted updates.  

**Original Text Preview:**

## Summary

In the `veRAACToken` contract, several functions are designed to manage the locking, unlocking, increasing, and extending of RAAC tokens, as well as their lock durations. The protocol utilizes a dual-gauge system for rewards distribution, where user rewards are determined based on both their boost factors and internally recorded boost balances.

However, there is a critical issue in how these functions update boost balances. Specifically, the functions `lock`, `increase`, `extend`, `withdraw`, and `emergencyWithdraw` do not update the users’ boost balances appropriately. Notably, the `extend` function fails to call `_updateBoostState`, and while the `lock` and `increase` functions do call `_updateBoostState`, that function itself does not invoke `_boostState.updateUserBalance`. Moreover, the current implementation passes the locked RAAC token amount (`locks[msg.sender].amount`) to update the boost state, even though the boost state should instead be updated using the newly calculated voting power (`newPower`), which accounts for time-weighted decay. This discrepancy can lead to significant reward manipulation, where users may not receive the rewards they are entitled to, thereby destabilizing the incentive structure and eroding trust in the protocol.

## Vulnerability Details

1.  **Boost State Update Shortcomings:**

    - The `_updateBoostState` function is designed to update global boost parameters such as overall voting power, total voting power, and total weight based on locked tokens. It also updates the boost period by calling `_boostState.updateBoostPeriod()`.
    - **Missing Call:** The function does not invoke `_boostState.updateUserBalance(user, newAmount)`, which is necessary for updating user-specific boost balances.
    - **Incorrect Parameter:** Additionally, the function is called with `locks[msg.sender].amount` (the raw locked RAAC token amount) rather than the calculated voting power (`newPower`) that reflects the current time-weighted state. As a result, the boost state does not correctly represent the actual voting power of the user, leading to inaccurate reward and governance weight calculations.

    ```solidity
         function _updateBoostState(address user, uint256 newAmount) internal {
             // Update boost calculator state
             _boostState.votingPower = _votingState.calculatePowerAtTimestamp(user, block.timestamp);
             _boostState.totalVotingPower = totalSupply();
             _boostState.totalWeight = _lockState.totalLocked;

             _boostState.updateBoostPeriod();
     @>      // @info: doesn't call _boostState.updateUserBalance(user, newAmount);
         }
    ```

2.  **Function-Specific Issues:**

    - **`lock` Function:**  
      After creating a lock via `_lockState.createLock(msg.sender, amount)`, the function calls `_updateBoostState(msg.sender, amount)`. Since this call does not forward the new voting power and omits the necessary user balance update in the boost state, the user's boost balance remains outdated.

    ```solidity
        function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
            //...
            //...
            //...

            // Create lock position
            _lockState.createLock(msg.sender, amount, duration);
    @>      _updateBoostState(msg.sender, amount);

            //...
            //...
            //...

            emit LockCreated(msg.sender, amount, unlockTime);
        }
    ```

    - **`increase` Function:**  
      When a user increases their locked amount using `_lockState.increaseLock(msg.sender, amount)`, `_updateBoostState` is invoked with `locks[msg.sender].amount` rather than the new calculated voting power. This misrepresentation leads to an inaccurately low boost factor, adversely affecting both reward allocations and governance influence.

    ```solidity
        function increase(uint256 amount) external nonReentrant whenNotPaused {
            _lockState.increaseLock(msg.sender, amount);
    @>      _updateBoostState(msg.sender, locks[msg.sender].amount);

            //...
            //...
            //...

            emit LockIncreased(msg.sender, amount);
        }
    ```

    - **`extend` Function:**  
      The `extend` function does not call `_updateBoostState` at all. Instead, it manually updates voting power by recalculating and then minting or burning tokens accordingly. This omission means that changes in lock duration do not reflect in the global boost state, leaving user-specific boost balances stale.

      ```solidity
        function extend(uint256 newDuration) external nonReentrant whenNotPaused {
             // Extend lock using LockManager
             uint256 newUnlockTime = _lockState.extendLock(msg.sender, newDuration);

             // Update voting power
             LockManager.Lock memory userLock = _lockState.locks[msg.sender];
             (int128 newBias, int128 newSlope) =
                 _votingState.calculateAndUpdatePower(msg.sender, userLock.amount, newUnlockTime);

             // Update checkpoints
             uint256 oldPower = balanceOf(msg.sender);
             uint256 newPower = uint256(uint128(newBias));
             _checkpointState.writeCheckpoint(msg.sender, newPower);

        @>  // @info: doesn't call the \_updateBoostState function to update boost factors.

             // Update veToken balance
             if (newPower > oldPower) {
                 _mint(msg.sender, newPower - oldPower);
             } else if (newPower < oldPower) {
                 _burn(msg.sender, oldPower - newPower);
             }

             emit LockExtended(msg.sender, newUnlockTime);
        }
      ```

3.  **Potential Exploitation:**
    - **Reward Manipulation:**  
      Inaccurate boost balances can be exploited by malicious actors to either underclaim or overclaim rewards. By carefully timing lock operations and relying on stale boost state data, attackers could skew the reward distribution mechanism in their favor.
    - **Governance Discrepancies:**  
      Since governance weight is tied to the voting power (which is derived from boost factors), any inaccuracy in boost state updates may result in misrepresentation of voting power. This can lead to governance decisions being influenced by parties whose actual stake is lower than what is recorded.

## Proof of Concept

To demonstrate this vulnerability, the following Proof of Concept (PoC) is provided. The PoC is written using the Foundry tool.

1. **Step 1:** Create a Foundry project and place all the contracts in the `src` directory.

2. **Step 2:** Create a `test` directory and a `mocks` folder within the `src` directory (or use an existing mocks folder).

3. **Step 3:** Create all necessary mock contracts, if required.

4. **Step 4:** Create a test file (with any name) in the `test` directory.

```solidity
    // SPDX-License-Identifier: MIT

    pragma solidity ^0.8.19;

    import {Test, console} from "forge-std/Test.sol";

    import {veRAACToken} from "../src/core/tokens/veRAACToken.sol";
    import {RAACToken} from "../src/core/tokens/RAACToken.sol";
    import {TimeWeightedAverage} from "../src/libraries/math/TimeWeightedAverage.sol";
    import {LockManager} from "../src/libraries/governance/LockManager.sol";
    import {IveRAACToken} from "../src/interfaces/core/tokens/IveRAACToken.sol";

    contract VeRAACTokenTest is Test {
        veRAACToken veRaacToken;
        RAACToken raacToken;

        address RAAC_OWNER = makeAddr("RAAC_OWNER");
        address RAAC_MINTER = makeAddr("RAAC_MINTER");
        uint256 initialRaacSwapTaxRateInBps = 200; // 2%, 10000 - 100%
        uint256 initialRaacBurnTaxRateInBps = 150; // 1.5%, 10000 - 100%

        address VE_RAAC_OWNER = makeAddr("VE_RAAC_OWNER");
        address ALICE = makeAddr("ALICE");
        address BOB = makeAddr("BOB");
        address CHARLIE = makeAddr("CHARLIE");
        address DEVIL = makeAddr("DEVIL");

        function setUp() public {
            raacToken = new RAACToken(RAAC_OWNER, initialRaacSwapTaxRateInBps, initialRaacBurnTaxRateInBps);

            vm.startPrank(VE_RAAC_OWNER);
            veRaacToken = new veRAACToken(address(raacToken));
            vm.stopPrank();
        }
    }
```

5. **Step 5:** Add the following test PoC in the test file, after the `setUp` function.

```solidity
    function testUpdateBoostStateDoesNotUpdateUsersBoostBalanceAndOtherParameters() public {
        uint256 LOCK_AMOUNT = 10_000_000e18;
        uint256 LOCK_DURATION = 365 days;

        vm.startPrank(RAAC_OWNER);
        raacToken.setMinter(RAAC_MINTER);
        vm.stopPrank();

        vm.startPrank(RAAC_MINTER);
        raacToken.mint(ALICE, LOCK_AMOUNT);
        vm.stopPrank();

        uint256 veRaacRaacTokenBalance = raacToken.balanceOf(address(veRaacToken));

        // Step 1: Alice approves and locks tokens.
        vm.startPrank(ALICE);
        raacToken.approve(address(veRaacToken), LOCK_AMOUNT);
        veRaacToken.lock(LOCK_AMOUNT / 2, LOCK_DURATION);
        vm.stopPrank();

        veRaacRaacTokenBalance = raacToken.balanceOf(address(veRaacToken));

        vm.warp(block.timestamp + 1 days);

        // Step 2: ALICE increases their lock to update internal state (including checkpoints and boost state).
        vm.startPrank(ALICE);
        veRaacToken.increase(LOCK_AMOUNT / 2);
        vm.stopPrank();

        TimeWeightedAverage.Period memory aliceBoostTimePeriod = veRaacToken.getUserBoostTimePeriods(ALICE);

        console.log("Alice's data persists even after withdrawal...");
        console.log("startTime: ", aliceBoostTimePeriod.startTime);
        console.log("endTime: ", aliceBoostTimePeriod.endTime);
        console.log("lastUpdateTime: ", aliceBoostTimePeriod.lastUpdateTime);
        console.log("value: ", aliceBoostTimePeriod.value);
        console.log("weightedSum: ", aliceBoostTimePeriod.weightedSum);
        console.log("totalDuration: ", aliceBoostTimePeriod.totalDuration);
        console.log("weight: ", aliceBoostTimePeriod.weight);
    }
```

6. **Step 6:** To run the test, execute the following commands in your terminal:

```bash
forge test --mt testUpdateBoostStateDoesNotUpdateUsersBoostBalanceAndOtherParameters -vv
```

7. **Step 7:** Review the output. The expected output should indicate that users internal balance and users specific parameters or members haven't updated.

```log
    [⠒] Compiling...
    No files changed, compilation skipped

    Ran 1 test for test/VeRAACTokenTest.t.sol:VeRAACTokenTest
    [PASS] testUpdateBoostStateDoesNotUpdateUsersBoostBalanceAndOtherParameters() (gas: 711963)
    Logs:
    Alice's data persists even after withdrawal...
    startTime:  0
    endTime:  0
    lastUpdateTime:  0
    value:  0
    weightedSum:  0
    totalDuration:  0
    weight:  0

    Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.99ms (829.20µs CPU time)

    Ran 1 test suite in 12.82ms (3.99ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As demonstrated, the test confirms that the users boost internal balance and other specific parameters or members haven't updated.

**Notice:** Some modifications have been made to let you show the issue.

`veRAACToken::getUserBoostTimePeriods:` (newly introduced getter for ease)

```solidity
    // * @dev temporary not exist in actual
    function getUserBoostTimePeriods(address user) external view returns (TimeWeightedAverage.Period memory) {
        return _boostState.userPeriods[user];
    }
```

## Impact

- **Distorted Voting Power Calculations:**  
  The failure to update user-specific boost balances accurately leads to an inaccurate representation of voting power. Since boost factors are a key determinant in voting weight, any miscalculation undermines the integrity of governance outcomes.

- **Inconsistent Reward Distribution:**  
  Rewards based on boost factors will be misallocated because the internal state does not reflect the true, updated voting power. Users may receive lower rewards than they are entitled to, or malicious actors might exploit this discrepancy to skew the reward system.

- **Global Parameter Inconsistencies:**  
  The omission of proper boost state updates creates discrepancies in the overall governance system. Historical and real-time data used for governance decision-making become unreliable, making audits and performance tracking challenging.

- **Increased Vulnerability to Exploitation:**  
  Attackers can exploit stale or inaccurate boost data to manipulate governance proposals and reward distribution. This manipulation can lead to governance takeovers or biased proposals that favor certain participants over others.

- **Erosion of User Trust:**  
  Persistent inaccuracies in voting power and reward distribution will erode user confidence in the protocol, potentially reducing long-term engagement and participation in governance.

## Tools Used

- Manual Review
- Foundry
- Console Log (foundry)
- SRC Mutation

## Recommendations

- Modify the `_updateBoostState` function to include a call to `_boostState.updateUserBalance(user, newAmount)` so that user-specific boost balances are accurately updated whenever a lock is created or increased.

- In the extend function, add a call to `_updateBoostState` after extending the lock and calculating new voting power. This will ensure that changes to the lock duration correctly reflect in the global boost state and user boost balances.

One possible solution is as follows:

`veRAACToken::_updateBoostState:`

```diff
    function _updateBoostState(address user, uint256 newAmount) internal {
        // Update boost calculator state
        _boostState.votingPower = _votingState.calculatePowerAtTimestamp(user, block.timestamp);
        _boostState.totalVotingPower = totalSupply();
        _boostState.totalWeight = _lockState.totalLocked;

        _boostState.updateBoostPeriod();
+       _boostState.updateUserBalance(user, newAmount);
    }
```

`veRAACToken::extend:`

```diff
    function extend(uint256 newDuration) external nonReentrant whenNotPaused {
        // Extend lock using LockManager
        uint256 newUnlockTime = _lockState.extendLock(msg.sender, newDuration);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        (int128 newBias, int128 newSlope) =
            _votingState.calculateAndUpdatePower(msg.sender, userLock.amount, newUnlockTime);

        // Update checkpoints
        uint256 oldPower = balanceOf(msg.sender);
        uint256 newPower = uint256(uint128(newBias));
+       _updateBoostState(msg.sender, newPower);
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Update veToken balance
        if (newPower > oldPower) {
            _mint(msg.sender, newPower - oldPower);
        } else if (newPower < oldPower) {
            _burn(msg.sender, oldPower - newPower);
        }

        emit LockExtended(msg.sender, newUnlockTime);
    }
```

`veRAACToken::lock:`

```diff
    function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
        if (amount == 0) revert InvalidAmount();
        if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
        if (totalSupply() + amount > MAX_TOTAL_SUPPLY) revert TotalSupplyLimitExceeded();
        if (duration < MIN_LOCK_DURATION || duration > MAX_LOCK_DURATION) {
            revert InvalidLockDuration();
        }

        // Do the transfer first - this will revert with ERC20InsufficientBalance if user doesn't have enough tokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);

        // Calculate unlock time
        uint256 unlockTime = block.timestamp + duration;

        // Create lock position
        _lockState.createLock(msg.sender, amount, duration);
-       _updateBoostState(msg.sender, amount);

        // Calculate initial voting power
        (int128 bias, int128 slope) = _votingState.calculateAndUpdatePower(msg.sender, amount, unlockTime);

        // Update checkpoints
        uint256 newPower = uint256(uint128(bias));
+       _updateBoostState(msg.sender, newPower);
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Mint veTokens
        _mint(msg.sender, newPower);

        emit LockCreated(msg.sender, amount, unlockTime);
    }
```

`veRAACToken::increase:`

```diff
    function increase(uint256 amount) external nonReentrant whenNotPaused {
        // Increase lock using LockManager
        _lockState.increaseLock(msg.sender, amount);
-       _updateBoostState(msg.sender, locks[msg.sender].amount);

        // Update voting power
        LockManager.Lock memory userLock = _lockState.locks[msg.sender];
        // @info: adding amount in already updated locked amount
        (int128 newBias, int128 newSlope) =
            _votingState.calculateAndUpdatePower(msg.sender, userLock.amount + amount, userLock.end);

        // Update checkpoints
        uint256 newPower = uint256(uint128(newBias));
+       _updateBoostState(msg.sender, newPower);
        _checkpointState.writeCheckpoint(msg.sender, newPower);

        // Transfer additional tokens and mint veTokens
        raacToken.safeTransferFrom(msg.sender, address(this), amount);

        _mint(msg.sender, newPower - balanceOf(msg.sender));

        emit LockIncreased(msg.sender, amount);
    }
```

---
