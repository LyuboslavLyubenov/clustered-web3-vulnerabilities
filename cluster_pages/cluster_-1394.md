# Cluster -1394

**Rank:** #330  
**Count:** 21  

## Label
Inaccurate state tracking in vesting mechanisms leads to double-counting, incorrect token allocations, and inconsistent claim eligibility, compromising token integrity and user entitlements.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Inaccurate state tracking in vesting mechanisms leads to double-counting, incorrect token allocations, and inconsistent claim eligibility, compromising token integrity and user entitlements.  

**Original Text Preview:**

## Severity

**Impact**: Medium, temporary DOS of funds

**Likelihood**: Medium, occurs when funds are staked / used in the protocol

## Description

The `Vesting` is used to lock funds on behalf of a user while giving them the ability to participate in the protocol. The idea behind the contract is that the amount of funds in the contract will gradually be unlocked over time, while giving the owner the ability to use those tokens to stake or register workers and gateways with. This allows the users to use the tokens in the ecosystem, without having the option to withdraw them all at once.

The `VestingWallet` OpenZeppelin contract tracks a `_erc20Released` variable which keeps track of the tokens already paid out. When trigerring a new payout, the amount of tokens available is calculated as shown below.

```solidity
function releasable() public view virtual returns (uint256) {
    return vestedAmount(uint64(block.timestamp)) - released();
}
```

The issue is that since the contract uses the `erc20.balanceOf` function to track the vesting amounts, this above expression can underflow and revert. This is because the balance in the contract can decrease if the user wishes to allocate some of the vested amount to staking or registering workers and gateways.

This is best demonstrated in the POC below.

The issue is recreated in the following steps

1. The vesting contract is transferred in 8 eth of tokens.
2. At the midpoint, half (4 eth) tokens have been vested out. These tokens are collected by calling `release`. Thus `_erc20Released` is set to 4 eth for the token.
3. Of the remaining 4 eth in the contract, 2 eth is allocated to staking.
4. After some time, more tokens are vested out.
5. Now when `release` is called, the function reverts. This is because `vestedAmount()` returns a value less than 4 eth, and `_erc20Released` is 4 eth. This causes the `releasable` function to underflow and revert.

So even though the contract has funds and can afford to pay out some vested rewards, this function reverts.

```solidity
function test_AttackVesting() public {
    token.transfer(address(vesting), 8 ether);

    // Half (4 eth) is vested out
    vm.warp(vesting.start() + vesting.duration() / 2);
    vesting.release();
    assert(vesting.released(address(token)) == 4 ether);

    // Stake half of rest (2 ETH)
    bytes memory call = abi.encodeWithSelector(
        Staking.deposit.selector,
        0,
        2 ether
    );
    vesting.execute(address(router.staking()), call, 2 ether);

    // pass some time
    vm.warp(vesting.start() + (vesting.duration() * 60) / 100);
    // check rewards
    vm.expectRevert();
    vesting.release();
}
```

This causes a temporary DOS, and users are unable to release vested tokens until their stake or registration is paid out.

Since users lose access to part of the funds they deserve, this is a medium severity issue.

## Recommendations

Consider adding a `depositForVesting(unit amount)` function, and tracking the amount with a storage variable `baseAmount` updated in this function. This way, the vesting rewards will calculate rewards based on this and not the `erc20.balanceOf` value. The result is that we need not decrease the `baseAmount` when tokens are sent out for staking, and then the vested amounts will be correctly calculated based on the value of the contract, instead of just the balances. This will prevent scenarios where claiming can revert even if funds are available.

---
### Example 2

**Auto Label:** Inaccurate state tracking in vesting mechanisms leads to double-counting, incorrect token allocations, and inconsistent claim eligibility, compromising token integrity and user entitlements.  

**Original Text Preview:**

When a vesting is listed, the vesting is transferred to the `SecondSwap_VestingManager` contract. With no previous listings, the contract “inherits” the `stepsClaimed` from the listed vesting:

<https://github.com/code-423n4/2024-12-secondswap/blob/main/contracts/SecondSwap_StepVesting.sol# L288-L290>
```

@>        if (_vestings[_beneficiary].totalAmount == 0) {
            _vestings[_beneficiary] = Vesting({
@>            stepsClaimed: _stepsClaimed,
            ...
```

Suppose the `stepsClaimed` amount is positive. In that case, further listing allocations will be mixed with the previous one, meaning the “inherited” `stepsClaimed` amount will be present in the listings transferred from the `SecondSwap_VestingManager` contract to users with no allocation that buy listings through `SecondSwap_Marketplace::spotPurchase`.

This condition creates two scenarios that affect how much the user can claim:

Assuming for both scenarios that there are no listings yet for a given vesting plan.

Scenario 1:

1. First listing has no `claimedSteps`
2. Second listing has `claimedSteps`

Since the first listing has no `claimedSteps`, users with no previous vestings allocation can buy any of the listings and their listing won’t have claimed steps, allowing them to claim immediately after their purchase.

Scenario 2:

1. First listing has `claimedSteps`
2. Second listing has no claimedSteps

Due to the first listing having a positive `claimedSteps` amount, users with no previous vesting allocations will have their vestings inherit the `claimedSteps`, meaning they won’t be able to claim if they are on the current step corresponding to `claimedSteps`.

### Proof of Concept
```

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test, console} from "../lib/forge-std/src/Test.sol";
import {Vm} from "../lib/forge-std/src/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {SecondSwap_Marketplace} from "../contracts/SecondSwap_Marketplace.sol";
import {SecondSwap_MarketplaceSetting} from "../contracts/SecondSwap_MarketplaceSetting.sol";
import {SecondSwap_VestingManager} from "../contracts/SecondSwap_VestingManager.sol";
import {SecondSwap_VestingDeployer} from "../contracts/SecondSwap_VestingDeployer.sol";
import {SecondSwap_WhitelistDeployer} from "../contracts/SecondSwap_WhitelistDeployer.sol";
import {SecondSwap_StepVesting} from "../contracts/SecondSwap_StepVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SecondSwap_MarketplaceTest is Test {
    ERC1967Proxy marketplaceProxy;
    ERC1967Proxy vestingManagerProxy;
    ERC1967Proxy vestingDeployerProxy;
    SecondSwap_Marketplace marketplaceImplementation;
    SecondSwap_VestingManager vestingManagerImplementation;
    SecondSwap_VestingDeployer vestingDeployerImplementation;
    SecondSwap_VestingManager vestingManager;
    SecondSwap_Marketplace marketplace;
    SecondSwap_VestingDeployer vestingDeployer;
    SecondSwap_MarketplaceSetting marketplaceSetting;
    SecondSwap_WhitelistDeployer whitelistDeployer;
    MockERC20 marketplaceToken;
    MockERC20 vestingToken;
    MockUSDT usdt;
    address vestingSeller1 = makeAddr("vestingSeller1");
    address vestingSeller2 = makeAddr("vestingSeller2");
    address vestingBuyer = makeAddr("vestingBuyer");

    function setUp() public {
        marketplaceImplementation = new SecondSwap_Marketplace();
        vestingManagerImplementation = new SecondSwap_VestingManager();
        vestingDeployerImplementation = new SecondSwap_VestingDeployer();
        whitelistDeployer = new SecondSwap_WhitelistDeployer();

        marketplaceToken = new MockERC20("Marketplace Token", "MTK");
        vestingToken = new MockERC20("Vesting Token", "VTK");
        usdt = new MockUSDT();

        vestingManagerProxy = new ERC1967Proxy(
            address(vestingManagerImplementation),
            abi.encodeWithSignature("initialize(address)", address(this))
        );

        vestingManager = SecondSwap_VestingManager(
            address(vestingManagerProxy)
        );

        vestingDeployerProxy = new ERC1967Proxy(
            address(vestingDeployerImplementation),
            abi.encodeWithSignature(
                "initialize(address,address)",
                address(this),
                address(vestingManager)
            )
        );

        vestingDeployer = SecondSwap_VestingDeployer(
            address(vestingDeployerProxy)
        );

        marketplaceSetting = new SecondSwap_MarketplaceSetting({
            _feeCollector: address(this),
            _s2Admin: address(this),
            _whitelistDeployer: address(whitelistDeployer),
            _vestingManager: address(vestingManager),
            _usdt: address(usdt)
        });
        marketplaceProxy = new ERC1967Proxy(
            address(marketplaceImplementation),
            abi.encodeWithSignature(
                "initialize(address,address)",
                address(marketplaceToken),
                address(marketplaceSetting)
            )
        );
        marketplace = SecondSwap_Marketplace(address(marketplaceProxy));

        vestingDeployer.setTokenOwner(address(vestingToken), address(this));
        vestingManager.setVestingDeployer(address(vestingDeployer));
        vestingManager.setMarketplace(address(marketplace));
        vestingToken.mint(address(this), 10e18 * 2);
        marketplaceToken.mint(vestingBuyer, 10e18);

    }

    // 1. First listing has no claimedSteps
    // 2. Second listing has claimedSteps
    function testCase1() public {
        vm.recordLogs();

        vestingDeployer.deployVesting({
            tokenAddress: address(vestingToken),
            startTime: block.timestamp,
            endTime: block.timestamp + 200 days,
            steps: 200,
            vestingId: "1"
        });

        Vm.Log[] memory entries = vm.getRecordedLogs();
        assertEq(entries.length, 3);

        // Check the event signature
        assertEq(
            entries[2].topics[0],
            keccak256("VestingDeployed(address,address,string)")
        );
        assertEq(entries[2].emitter, address(vestingDeployer));

        (address deployedVesting, ) = abi.decode(
            entries[2].data,
            (address, string)
        );

        SecondSwap_StepVesting vesting = SecondSwap_StepVesting(
            deployedVesting
        );

        vestingToken.approve(address(vesting), 10e18);

        vesting.createVesting({
            _beneficiary: vestingSeller1,
            _totalAmount: 10e18
        });

        // After 10 days, vestingSeller1 lists 10% of the total amount
        vm.warp(block.timestamp + 10 days);
        uint256 amount = (10e18 * 1000) / 10000; // 10 percent of the total amount

        vm.startPrank(vestingSeller1);
        marketplace.listVesting({
            _vestingPlan: address(vesting),
            _amount: amount,
            _price: 1e18,
            _discountPct: 0,
            _listingType: SecondSwap_Marketplace.ListingType.SINGLE,
            _discountType: SecondSwap_Marketplace.DiscountType.NO,
            _maxWhitelist: 0,
            _currency: address(marketplaceToken),
            _minPurchaseAmt: amount,
            _isPrivate: false
        });

        // vestingSeller1 claims
        vesting.claim();

        //vestingSeller1 lists another 10% of the total amount
        marketplace.listVesting({
            _vestingPlan: address(vesting),
            _amount: amount,
            _price: 1e18,
            _discountPct: 0,
            _listingType: SecondSwap_Marketplace.ListingType.SINGLE,
            _discountType: SecondSwap_Marketplace.DiscountType.NO,
            _maxWhitelist: 0,
            _currency: address(marketplaceToken),
            _minPurchaseAmt: amount,
            _isPrivate: false
        });
        vm.stopPrank();

        // At this point, the SecondSwap_VestingManager contract has 0 stepsClaimed

// vestingBuyer buys the second listed vesting
        vm.startPrank(vestingBuyer);
        marketplaceToken.approve(address(marketplace), amount + (amount * marketplaceSetting.buyerFee()));
        marketplace.spotPurchase({
            _vestingPlan: address(vesting),
            _listingId: 1,
            _amount: amount,
            _referral: address(0)
        });
        vm.stopPrank();

        (uint256 vestingBuyerClaimableAmount, ) = vesting.claimable(vestingBuyer);

        console.log("Buyer claimable amount: ", vestingBuyerClaimableAmount);
    }

    // 1. First listing has claimedSteps
    // 2. Second listing has no claimedSteps
    function testCase2() public {
        vm.recordLogs();

        vestingDeployer.deployVesting({
            tokenAddress: address(vestingToken),
            startTime: block.timestamp,
            endTime: block.timestamp + 200 days,
            steps: 200,
            vestingId: "1"
        });

        Vm.Log[] memory entries = vm.getRecordedLogs();
        assertEq(entries.length, 3);

        // Check the event signature
        assertEq(
            entries[2].topics[0],
            keccak256("VestingDeployed(address,address,string)")
        );
        assertEq(entries[2].emitter, address(vestingDeployer));

        (address deployedVesting, ) = abi.decode(
            entries[2].data,
            (address, string)
        );

        SecondSwap_StepVesting vesting = SecondSwap_StepVesting(
            deployedVesting
        );

        vestingToken.approve(address(vesting), 10e18 * 2);

        vesting.createVesting({
            _beneficiary: vestingSeller1,
            _totalAmount: 10e18
        });

        vesting.createVesting({
            _beneficiary: vestingSeller2,
            _totalAmount: 10e18
        });

        // After 10 days, vestingSeller1 claims and then lists 10% of the total amount
        vm.warp(block.timestamp + 10 days);
        uint256 amount = (10e18 * 1000) / 10000; // 10 percent of the total amount

        vm.startPrank(vestingSeller1);
        // vestingSeller1 claims
        vesting.claim();

        marketplace.listVesting({
            _vestingPlan: address(vesting),
            _amount: amount,
            _price: 1e18,
            _discountPct: 0,
            _listingType: SecondSwap_Marketplace.ListingType.SINGLE,
            _discountType: SecondSwap_Marketplace.DiscountType.NO,
            _maxWhitelist: 0,
            _currency: address(marketplaceToken),
            _minPurchaseAmt: amount,
            _isPrivate: false
        });
        vm.stopPrank();

        //vestingSeller2 lists 10% of the total amount. Has not claimed yet
        vm.prank(vestingSeller2);
        marketplace.listVesting({
            _vestingPlan: address(vesting),
            _amount: amount,
            _price: 1e18,
            _discountPct: 0,
            _listingType: SecondSwap_Marketplace.ListingType.SINGLE,
            _discountType: SecondSwap_Marketplace.DiscountType.NO,
            _maxWhitelist: 0,
            _currency: address(marketplaceToken),
            _minPurchaseAmt: amount,
            _isPrivate: false
        });

        // At this point, the SecondSwap_VestingManager contract has stepsClaimed

        // vestingBuyer buys the second listed vesting
        vm.startPrank(vestingBuyer);
        marketplaceToken.approve(address(marketplace), amount + (amount * marketplaceSetting.buyerFee()));
        marketplace.spotPurchase({
            _vestingPlan: address(vesting),
            _listingId: 1,
            _amount: amount,
            _referral: address(0)
        });
        vm.stopPrank();

        (uint256 vestingBuyerClaimableAmount, ) = vesting.claimable(vestingBuyer);

        console.log("Buyer claimable amount: ", vestingBuyerClaimableAmount);
    }
}

contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}

    function mint(address account, uint amount) external {
        _mint(account, amount);
    }
}

contract MockUSDT is ERC20 {
    constructor() ERC20("Tether USD", "USDT") {}

    function mint(address account, uint amount) external {
        _mint(account, amount);
    }

    function decimals() public pure override returns (uint8) {
        return 6;
    }
}
```

Steps to reproduce:

1. Run `npm i --save-dev @nomicfoundation/hardhat-foundry` in the terminal to install the hardhat-foundry plugin.
2. Add `require("@nomicfoundation/hardhat-foundry");` to the top of the hardhat.config.js file.
3. Run `npx hardhat init-foundry` in the terminal.
4. Create a file “StepVestingTest.t.sol” in the “test/” directory and paste the provided PoC.
5. Run `forge test --mt testCase1` in the terminal.
6. Run `forge test --mt testCase2` in the terminal.

### Recommended mitigation steps

Add a virtual total amount to the manager contract on each vesting plan deployed.

**TechticalRAM (SecondSwap) confirmed**

---

---
### Example 3

**Auto Label:** Inaccurate state tracking in vesting mechanisms leads to double-counting, incorrect token allocations, and inconsistent claim eligibility, compromising token integrity and user entitlements.  

**Original Text Preview:**

**Severity**: Critical

**Status**: Resolved

**Description**

In the updateUserVesting function of ERC20Vesting, the condition if(user.claimedAmount < newAmount) is incorrect. This condition erroneously checks if the claimedAmount is less than newAmount. The intended logic should verify that the claimedAmount does not exceed the newAmount. Using the wrong comparison allows reducing the vesting amount below what has already been claimed, potentially resulting in an underflow condition when calculating the claim amount and it will always revert.

**Recommendation**:

Update the condition to if(user.claimedAmount > newAmount) revert()  to correctly enforce that the new vesting amount must be greater than or equal to the already claimed amount.

---
