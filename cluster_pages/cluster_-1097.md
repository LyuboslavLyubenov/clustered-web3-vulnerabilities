# Cluster -1097

**Rank:** #151  
**Count:** 93  

## Label
Failure to account for dynamic token supply changes leads to stale balance tracking, incorrect entitlement calculations, and flawed distribution—causing unfair rewards, claim failures, and governance misallocations.

## Cluster Information
- **Total Findings:** 93

## Examples

### Example 1

**Auto Label:** Race conditions in reward calculation and state synchronization lead to inaccurate or lost rewards, causing economic inefficiencies and user loss.  

**Original Text Preview:**

`Gauge` updates rewards whenever a new action is performed (deposit, withdraw, claim rewards, or notify rewards). If the `totalSupply` is zero, the `rewardPerToken` will not be updated, while the last updated timestamp will be updated.

```solidity
    modifier updateReward(address _account) {
        rewardPerTokenStored = rewardPerToken();
        updatedAt = lastTimeRewardApplicable();

(...)

    function rewardPerToken() public view returns (uint256) {
        if (totalSupply == 0) {
            return rewardPerTokenStored;
        }
```

This will cause that for periods where the total supply is zero, the rewards to be distributed will get locked. While there are economic incentives for at least one user to have a non-zero balance in the gauge, this is not guaranteed, especially at the moment of creation of the gauge or after reviving it, as users may not deposit LP tokens immediately.

Proof of concept

Add the following code to the file `TestGauge.t.sol` and run `forge test -f https://rpc.hyperliquid.xyz/evm --mt test_audit_GaugeLockedRewards --mc TestGauge`.

```solidity
function test_audit_GaugeLockedRewards() public {
	// setup
	test_CreateGauge();
	Gauge _gauge = Gauge(gauge.get(pairListVolatile[0]));
	address user1 = userList[0];
	address lpToken = address(_gauge.lpToken());
	vm.prank(kitten.minter());
	kitten.mint(address(voter), 2 ether);
	deal(lpToken, user1, 1 ether);

	// voter distributes 1e18 KITTEN to the gauge
	vm.prank(address(voter));
	_gauge.notifyRewardAmount(1 ether);

	// after 1 week, user1 deposits LP tokens into the gauge
	vm.warp(block.timestamp + 1 weeks);
	vm.startPrank(user1);
	IERC20(lpToken).approve(address(_gauge), 1 ether);
	_gauge.deposit(1 ether);
	vm.stopPrank();

	// voter distributes another 1e18 KITTEN to the gauge
	vm.prank(address(voter));
	_gauge.notifyRewardAmount(1 ether);

	// after another week, user1 claims rewards
	vm.warp(block.timestamp + 1 weeks);
	vm.prank(user1);
	_gauge.getReward(user1);

	// gauge has 1e18 KITTEN balance, but no rewards left
	assertGe(kitten.balanceOf(address(_gauge)), 1 ether);
	assertEq(_gauge.left(), 0);
}
```

**Recommendations**
Handle the distribution or recovery of rewards not distributed due to the total supply being zero. This can be done in different ways, such as:

- Sending the non-distributed rewards to the VotingReward contract.
- Sending the non-distributed rewards to a specific address designated for this purpose.
- Adding the non-distributed rewards to the next distribution cycle.

---
### Example 2

**Auto Label:** Race conditions in reward calculation and state synchronization lead to inaccurate or lost rewards, causing economic inefficiencies and user loss.  

**Original Text Preview:**

While the `CLGauge` contract has a token recovery function, the `Gauge`, `VotingReward`, and `RebaseReward` contracts do not.

This prevents the recovery of rewards not distributed, dust amounts resulting from rounding errors, or tokens sent by mistake to these contracts.

It is recommended to implement a token recovery function in these contracts.

---
### Example 3

**Auto Label:** Race conditions in reward calculation and state synchronization lead to inaccurate or lost rewards, causing economic inefficiencies and user loss.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

When rewards are distributed to `Gauge`, `CLGauge`, and Bribes contracts via `notifyRewardAmount`, it's possible that the total supply is 0 during that reward period/epoch, resulting in no rewards being distributed.

```solidity
    function notifyRewardAmount(address token, uint amount) external lock {
        require(amount > 0);
        if (!isReward[token]) {
            require(
                IVoter(voter).isWhitelisted(token),
                "bribe tokens must be whitelisted"
            );
            require(
                rewards.length < MAX_REWARD_TOKENS,
                "too many rewards tokens"
            );
        }
        // bribes kick in at the start of next bribe period
        uint adjustedTstamp = getEpochStart(block.timestamp);
        uint epochRewards = tokenRewardsPerEpoch[token][adjustedTstamp];

        _safeTransferFrom(token, msg.sender, address(this), amount);
        tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;

        periodFinish[token] = adjustedTstamp + DURATION;

        if (!isReward[token]) {
            isReward[token] = true;
            rewards.push(token);
        }

        emit NotifyReward(msg.sender, token, adjustedTstamp, amount);
    }
```

However, those contracts lack functionality to rescue the rewards, causing the distributed rewards to become permanently stuck inside them.

## Recommendations

Add functionality to rescue the rewards.

---
