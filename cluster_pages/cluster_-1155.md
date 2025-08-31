# Cluster -1155

**Rank:** #25  
**Count:** 702  

## Label
Inaccurate boost calculation due to flawed arithmetic, improper state tracking, and failure to validate or aggregate user contributions, leading to unfair rewards, incentive misalignment, and potential reward manipulation.

## Cluster Information
- **Total Findings:** 702

## Examples

### Example 1

**Auto Label:** Misconfigured or flawed reward calculation logic leading to inflated, lost, or unbounded rewards, compromising financial accuracy and user incentive alignment.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/factories/BribeFactoryV3.sol# L209-L219>

### Finding description

The `recoverERC20AndUpdateData` function in the `BribeFactoryV3` contract incorrectly calls `emergencyRecoverERC20` instead of `recoverERC20AndUpdateData` on the `IBribe` interface. This misnamed function call results in the failure to update the `tokenRewardsPerEpoch` mapping in the `Bribe` contract, which is critical for maintaining accurate reward accounting.

### Root Cause

In `BribeFactoryV3.sol`, the `recoverERC20AndUpdateData` function is defined as follows:
```

function recoverERC20AndUpdateData(address[] memory _bribe, address[] memory _tokens, uint[] memory _amounts) external onlyOwner {
    uint i = 0;
    require(_bribe.length == _tokens.length, 'MISMATCH_LEN');
    require(_tokens.length == _amounts.length, 'MISMATCH_LEN');

    for(i; i < _bribe.length; i++){
        if(_amounts[i] > 0) IBribe(_bribe[i]).emergencyRecoverERC20(_tokens[i], _amounts[i]);
    }
}
```

The function calls `IBribe(_bribe[i]).emergencyRecoverERC20` instead of `IBribe(_bribe[i]).recoverERC20AndUpdateData`. The correct function, `recoverERC20AndUpdateData` in the `Bribe` contract, updates the `tokenRewardsPerEpoch` mapping to reflect the recovered tokens:
```

function recoverERC20AndUpdateData(address tokenAddress, uint256 tokenAmount) external onlyAllowed {
    require(tokenAmount <= IERC20(tokenAddress).balanceOf(address(this)), "TOO_MUCH");

    uint256 _startTimestamp = IMinter(minter).active_period() + WEEK;
    uint256 _lastReward = tokenRewardsPerEpoch[tokenAddress][_startTimestamp];
    tokenRewardsPerEpoch[tokenAddress][_startTimestamp] = _lastReward - tokenAmount;
    IERC20(tokenAddress).safeTransfer(owner, tokenAmount);
    emit Recovered(tokenAddress, tokenAmount);
}
```

In contrast, `emergencyRecoverERC20` only transfers tokens without updating the mapping:
```

function emergencyRecoverERC20(address tokenAddress, uint256 tokenAmount) external onlyAllowed {
    require(tokenAmount <= IERC20(tokenAddress).balanceOf(address(this)), "TOO_MUCH");
    IERC20(tokenAddress).safeTransfer(owner, tokenAmount);
    emit Recovered(tokenAddress, tokenAmount);
}
```

### Impact

Incorrect reward accounting - failing to update `tokenRewardsPerEpoch` can lead to over-distribution of rewards. Users may claim rewards based on outdated values, potentially draining the `Bribe` contract’s token balance.

### Recommended mitigation steps

Update the function Call in `BribeFactoryV3`:

Modify the `recoverERC20AndUpdateData` function in `BribeFactoryV3.sol` to call the correct `recoverERC20AndUpdateData` function.

### Proof of Concept
```

const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");
const { ZERO_ADDRESS } = require("@openzeppelin/test-helpers/src/constants.js");

describe("BribeFactoryV3 Recovery Functions", function () {
    let bribeFactory;
    let mockToken;
    let bribe;
    let owner;
    let voter;
    let gaugeManager;
    let permissionsRegistry;
    let tokenHandler;
    let mockVoter;
    let mockGaugeManager;
    let mockTokenHandler;
    let mockVe;
    let mockMinter;

    beforeEach(async function () {
        // Get signers
        [owner, voter, gaugeManager, permissionsRegistry, tokenHandler] = await ethers.getSigners();

        // Deploy mock token
        const MockToken = await ethers.getContractFactory("MockERC20");
        mockToken = await MockToken.deploy("Mock Token", "MTK", 18);
        await mockToken.deployed();

        // Deploy mock Voter
        const MockVoter = await ethers.getContractFactory("MockVoter");
        mockVoter = await MockVoter.deploy();
        await mockVoter.deployed();

        // Deploy mock GaugeManager
        const MockGaugeManager = await ethers.getContractFactory("MockGaugeManager");
        mockGaugeManager = await MockGaugeManager.deploy();
        await mockGaugeManager.deployed();

        // Deploy mock TokenHandler
        const MockTokenHandler = await ethers.getContractFactory("MockTokenHandler");
        mockTokenHandler = await MockTokenHandler.deploy();
        await mockTokenHandler.deployed();

        // Deploy mock VotingEscrow
        const MockVotingEscrow = await ethers.getContractFactory("MockVotingEscrow");
        mockVe = await MockVotingEscrow.deploy();
        await mockVe.deployed();

        // Deploy mock Minter
        const MockMinter = await ethers.getContractFactory("MockMinter");
        mockMinter = await MockMinter.deploy();
        await mockMinter.deployed();

        // Set up mock Voter to return mock VE
        await mockVoter.setVe(mockVe.address);

        // Set up mock GaugeManager to return minter
        await mockGaugeManager.setMinter(mockMinter.address);

        // Set initial active period
        await mockMinter.setActivePeriod(Math.floor(Date.now() / 1000));

        // Deploy BribeFactoryV3 as upgradeable
        const BribeFactoryV3 = await ethers.getContractFactory("BribeFactoryV3");
        bribeFactory = await upgrades.deployProxy(BribeFactoryV3, [
            mockVoter.address,
            mockGaugeManager.address,
            permissionsRegistry.address,
            mockTokenHandler.address
        ], { initializer: 'initialize' });
        await bribeFactory.deployed();

        // Create a bribe contract
        const tx = await bribeFactory.createBribe(
            owner.address,
            mockToken.address,
            mockToken.address,
            "test"
        );
        const receipt = await tx.wait();

        // Get the bribe address from the factory's last_bribe variable
        const bribeAddress = await bribeFactory.last_bribe();
        bribe = await ethers.getContractAt("Bribe", bribeAddress);

        // Add some tokens to the bribe contract
        const amount = ethers.utils.parseEther("1000");
        await mockToken.mint(bribeAddress, amount);
    });

    describe("recoverERC20AndUpdateData", function () {
        it("Should demonstrate bug in recoverERC20AndUpdateData", async function () {
            // Get initial token rewards per epoch
            const epochStart = await bribe.getEpochStart();
            const initialRewards = await bribe.tokenRewardsPerEpoch(mockToken.address, epochStart);

            // Call recoverERC20AndUpdateData through the factory
            const recoveryAmount = ethers.utils.parseEther("100");
            await bribeFactory.recoverERC20AndUpdateData(
                [bribe.address],
                [mockToken.address],
                [recoveryAmount]
            );

            // Get final token rewards per epoch
            const finalRewards = await bribe.tokenRewardsPerEpoch(mockToken.address, epochStart);

            // This fails because the factory uses emergencyRecoverERC20, which does not update rewards
            expect(finalRewards).to.equal(initialRewards.sub(recoveryAmount),
                "Token rewards per epoch should be updated but weren't");
        });
    });
});
```

**Test Result**: The test fails with the following error, confirming the bug:
```

AssertionError: Expected "0" to be equal -100000000000000000000
+ expected - actual

 {
-  "_hex": "-0x056bc75e2d63100000"
+  "_hex": "0x00"
   "_isBigNumber": true
 }
```

**Explanation**:

The test expects `tokenRewardsPerEpoch` to decrease by `100 ether` after calling `recoverERC20AndUpdateData` on `BribeFactoryV3`. However, because `emergencyRecoverERC20` is called instead, `tokenRewardsPerEpoch` remains unchanged (`0`), causing the test to fail.

This demonstrates that the incorrect function call in `BribeFactoryV3` skips the critical update to `tokenRewardsPerEpoch`, leading to potential reward over-distribution.

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-44), [lonelybones](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-22) and [maxvzuvex](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-35).

The sponsor team requested that the following note be included:

> This issue originates from the upstream codebase, inherited from ThenaV2 fork. Given that ThenaV2 has successfully operated at scale for several months without incident, we assess the severity of this issue as low. The implementation has been effectively battle-tested in a production environment, which significantly reduces the practical risk associated with this finding.
> Reference: <https://github.com/ThenafiBNB/THENA-Contracts/blob/main/contracts/factories/BribeFactoryV3.sol# L199>

---

---
### Example 2

**Auto Label:** Misconfigured or flawed reward calculation logic leading to inflated, lost, or unbounded rewards, compromising financial accuracy and user incentive alignment.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/AlgebraCLVe33/GaugeCL.sol# L38>

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/AlgebraCLVe33/GaugeCL.sol# L157-L183>

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/AlgebraCLVe33/GaugeCL.sol# L257-L259>

### Finding description

The `GaugeCL::notifyRewardAmount` function attempts to update the `rewardRate` state variable, which is meant to track the reward emission rate for the current distribution period. However, the function declares a return variable named `rewardRate` (as part of the function’s return signature), which **shadows** the contract-level storage variable. As a result, assignments within the function update only the local return variable, and the **contract’s storage `rewardRate` is never updated**.

This leads to the reward rate always remaining at its initial value (`0`), and any queries to `rewardForDuration()` or other on-chain users referencing `rewardRate` will receive inaccurate results.
```

// @audit-issue Storage rewardRate is shadowed and never updated
function notifyRewardAmount(address token, uint256 reward)
    external nonReentrant isNotEmergency onlyDistribution
    returns (IncentiveKey memory incentivekey, uint256 rewardRate, uint128 bonusRewardRate)
{
    ...
    if (block.timestamp >= _periodFinish) {
        rewardRate = reward / DURATION;
    } else {
        uint256 remaining = _periodFinish - block.timestamp;
        uint256 leftover = remaining * rewardRate;
        rewardRate = (reward + leftover) / DURATION;
    }
    ...
}
```

### Issue identified

* The function’s return signature declares a local variable `rewardRate`, which **shadows** the contract’s storage variable of the same name.
* All assignments to `rewardRate` within the function only affect this local variable, not the contract storage.
* The contract storage variable `rewardRate` remains **permanently zero**, and is never updated by any function in the contract.
* Consequently, the function `rewardForDuration()` always returns `0`, misleading dApps, explorers, and UIs that rely on this state variable for reward calculations.

### Risk and impact

**Likelihood**:

* This is a deterministic bug and will always occur if the function signature is not fixed.
* Any user or protocol that queries `rewardForDuration()` or the public `rewardRate` will receive incorrect values.

**Impact**:

* Protocol dashboards, explorers, or analytic scripts may display incorrect or misleading reward information.
* Third-party tools or automated scripts that rely on on-chain `rewardRate` data could behave incorrectly.
* Does **not** directly cause loss of funds or user assets, but can lead to confusion or improper reward tracking.

### Recommended mitigation steps

Rename the function return variable to avoid shadowing, or directly assign to the storage variable inside the function:
```

function notifyRewardAmount(address token, uint256 reward)
    external
    nonReentrant
    isNotEmergency
    onlyDistribution
-   returns (IncentiveKey memory incentivekey, uint256 rewardRate, uint128 bonusRewardRate)
+   returns (IncentiveKey memory, uint256, uint128)
{
    require(token == address(rewardToken), "not rew token");
    if (block.timestamp >= _periodFinish) {
        rewardRate = reward / DURATION;
    } else {
        uint256 remaining = _periodFinish - block.timestamp;
        uint256 leftover = remaining * rewardRate;
        rewardRate = (reward + leftover) / DURATION;
    }
    _periodFinish = block.timestamp + DURATION;
    (IERC20Minimal rewardTokenAdd, IERC20Minimal bonusRewardTokenAdd, IAlgebraPool pool, uint256 nonce) =
        algebraEternalFarming.incentiveKeys(poolAddress);
-   incentivekey = IncentiveKey(rewardTokenAdd, bonusRewardTokenAdd, pool, nonce);
+   IncentiveKey memory incentivekey = IncentiveKey(rewardTokenAdd, bonusRewardTokenAdd, pool, nonce);
    bytes32 incentiveId = IncentiveId.compute(incentivekey);

    (,, address virtualPoolAddress,,,) = algebraEternalFarming.incentives(incentiveId);
-   (,bonusRewardRate) = IAlgebraEternalVirtualPool(virtualPoolAddress).rewardRates();
+   (,uint128 bonusRewardRate) = IAlgebraEternalVirtualPool(virtualPoolAddress).rewardRates();

    rewardToken.safeTransferFrom(DISTRIBUTION, address(this), reward);
    IERC20(token).safeApprove(farmingParam.algebraEternalFarming, reward);
    algebraEternalFarming.addRewards(incentivekey, uint128(reward), 0);
    emit RewardAdded(reward);
+   return(incentivekey, rewardRate, bonusRewardRate);
}
```

### Proof of Concept
```

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

contract ShadowedStorageExample {
    uint256 public rewardRate;

    function incrementRewardRate(uint256 amount) public returns (uint256 rewardRate) {
        rewardRate += amount;
    }
}
```

No matter how you call `incrementRewardRate` with different values, the storage variable `rewardRate` will **never change**. This is because the function’s return variable `rewardRate` shadows the contract’s storage variable of the same name.

All updates inside the function only affect the local return variable, not the storage.
As a result, `rewardRate()` (the public getter) will always return `0`, regardless of how many times you call the function or what arguments you provide.

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [lonelybones](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-19), [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-47) and [maxvzuvex](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-58).

---

---

---
### Example 3

**Auto Label:** Misconfigured or flawed reward calculation logic leading to inflated, lost, or unbounded rewards, compromising financial accuracy and user incentive alignment.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/main/contracts/RewardsDistributor.sol# L196-L247>

### Finding description

The RewardsDistributor contract contains an inconsistency in how it handles tokens managed by the Auto Voting Escrow Manager (AVM) system. In the `claim()` function, there’s code to check if a token is managed by AVM and, if so, to retrieve the original owner for sending rewards. However, this critical check is missing in the `claim_many()` function.

This creates a discrepancy in reward distribution where:

* `claim()` correctly sends rewards to the original owner of an AVM-managed token
* `claim_many()` incorrectly sends rewards to the current NFT owner (the AVM contract itself)

Relevant code from `claim()`:
```

if (_locked.end < block.timestamp && !_locked.isPermanent) {
    address _nftOwner = IVotingEscrow(voting_escrow).ownerOf(_tokenId);
    if (address(avm) != address(0) && avm.tokenIdToAVMId(_tokenId) != 0) {
        _nftOwner = avm.getOriginalOwner(_tokenId);
    }
    IERC20(token).transfer(_nftOwner, amount);
}
```

Relevant code from `claim_many()`:
```

if(_locked.end < block.timestamp && !_locked.isPermanent){
    address _nftOwner = IVotingEscrow(_voting_escrow).ownerOf(_tokenId);
    IERC20(token).transfer(_nftOwner, amount);
} else {
    IVotingEscrow(_voting_escrow).deposit_for(_tokenId, amount);
}
```

### Impact

This inconsistency has significant impact for users who have delegated their tokens to the AVM system:

1. Users who have expired locks and whose tokens are managed by AVM will lose their rewards if `claim_many()` is called instead of `claim()`.
2. The rewards will be sent to the AVM contract, which has no mechanism to forward these tokens to their rightful owners.
3. This results in permanent loss of rewards for affected users.
4. Since `claim_many()` is more gas efficient for claiming multiple tokens, it’s likely to be frequently used, increasing the likelihood and impact of this issue.

### Recommended mitigation steps

1. Add the AVM check to the `claim_many()` function to match the behavior in `claim()`:
```

if(_locked.end < block.timestamp && !_locked.isPermanent){
    address _nftOwner = IVotingEscrow(_voting_escrow).ownerOf(_tokenId);
    if (address(avm) != address(0) && avm.tokenIdToAVMId(_tokenId) != 0) {
        _nftOwner = avm.getOriginalOwner(_tokenId);
    }
    IERC20(token).transfer(_nftOwner, amount);
} else {
    IVotingEscrow(_voting_escrow).deposit_for(_tokenId, amount);
}
```

2. For better code maintainability, consider refactoring the owner resolution logic to a separate internal function:
```

function _getRewardRecipient(uint256 _tokenId) internal view returns (address) {
    address _nftOwner = IVotingEscrow(voting_escrow).ownerOf(_tokenId);
    if (address(avm) != address(0) && avm.tokenIdToAVMId(_tokenId) != 0) {
        _nftOwner = avm.getOriginalOwner(_tokenId);
    }
    return _nftOwner;
}
```

Then, use this function in both `claim()` and `claim_many()` to ensure consistent behavior.

### Proof of Concept

Scenario:

1. User A delegates their tokenId # 123 to the AVM system.
2. The lock for tokenId # 123 expires.
3. Someone calls `claim(123)`:

   * AVM check triggers and rewards go to User A (correct behavior)
4. Someone calls `claim_many([123])`:

   * No AVM check happens, rewards go to the AVM contract (incorrect behavior)
5. User A permanently loses their rewards.

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-31) and [maxvzuvex](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-25).

---

---
