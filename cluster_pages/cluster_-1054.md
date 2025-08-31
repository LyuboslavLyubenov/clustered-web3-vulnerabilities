# Cluster -1054

**Rank:** #167  
**Count:** 75  

## Label
Failure to properly route or validate funds during liquidation leads to unaddressed bad debt, undercollateralization, and protocol instability through incorrect entity authorization or missing token inclusion.

## Cluster Information
- **Total Findings:** 75

## Examples

### Example 1

**Auto Label:** Inadequate validation and flawed state transitions lead to incorrect asset calculations, unauthorized access, or unfair value distribution during withdrawals.  

**Original Text Preview:**

**Description:** After transitioning to the yield phase, redemptions of both pUSDe and yUSDe are processed by `pUSDeVault::_withdraw` such that they are both paid out in sUSDe. This is achieved by computing the sUSDe balance corresponding to the required USDe amount by calling its `previewWithdraw()` function:

```solidity
    function _withdraw(address caller, address receiver, address owner, uint256 assets, uint256 shares) internal override {

            if (PreDepositPhase.YieldPhase == currentPhase) {
                // sUSDeAssets = sUSDeAssets + user_yield_sUSDe
@>              assets += previewYield(caller, shares);

@>              uint sUSDeAssets = sUSDe.previewWithdraw(assets); // @audit - this rounds up because sUSDe requires the amount of sUSDe burned to receive assets amount of USDe to round up, but below we are transferring this rounded value out to the receiver which actually rounds against the protocol/yUSDe depositors!

                _withdraw(
                    address(sUSDe),
                    caller,
                    receiver,
                    owner,
                    assets, // @audit - this should not include the yield, since it is decremented from depositedBase
                    sUSDeAssets,
                    shares
                );
                return;
            }
        ...
    }
```

The issue with this is that `previewWithdraw()` returns the required sUSDe balance that must be burned to receive the specified USDe amount and so rounds up accordingly; however, here this rounded sUSDe amount is being transferred out of the protocol. This means that the redemption actually rounds in favour of the receiver and against the protocol/yUSDe depositors.

**Impact:** Value can leak from the system in favour of pUSDe redemptions at the expense of other yUSDe depositors.

**Proof of Concept:** Note that the following test will revert due to underflow when attempting to determine the fully redeemed amounts unless the mitigation from C-01 is applied:

```solidity
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC4626} from "@openzeppelin/contracts/interfaces/IERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import {MockUSDe} from "../contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "../contracts/test/MockStakedUSDe.sol";
import {MockERC4626} from "../contracts/test/MockERC4626.sol";

import {pUSDeVault} from "../contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "../contracts/predeposit/yUSDeVault.sol";

import {console2} from "forge-std/console2.sol";

contract RoundingTest is Test {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe public USDe;
    MockStakedUSDe public sUSDe;
    pUSDeVault public pUSDe;
    yUSDeVault public yUSDe;

    address account;

    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        address owner = msg.sender;

        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function test_rounding() public {
        uint256 userDeposit = 100 ether;

        // fund users
        USDe.mint(alice, userDeposit);
        USDe.mint(bob, userDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), userDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(userDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), userDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(userDeposit, bob);
        vm.stopPrank();

        // setup assertions
        assertEq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        assertEq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            account = msg.sender;
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(account);
            USDe.mint(account, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        uint256 sUSDeYieldAmount = 1_000 ether;
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe_actual = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        // bob redeems from yUSDe
        uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
        vm.prank(bob);
        uint256 bobRedeemed_pUSDe_reported = yUSDe.redeem(bobShares_yUSDe, bob, bob);
        uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
        uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

        console2.log("Alice redeemed sUSDe: %s", aliceRedeemed_sUSDe);
        console2.log("Alice redeemed USDe (reported): %s", aliceRedeemed_USDe_reported);
        console2.log("Alice redeemed USDe (actual): %s", aliceRedeemed_USDe_actual);

        console2.log("Bob redeemed pUSDe (reported): %s", bobRedeemed_pUSDe_reported);
        console2.log("Bob redeemed pUSDe (actual): %s", bobShares_pUSDe);
        console2.log("Bob redeemed sUSDe: %s", bobRedeemed_sUSDe);
        console2.log("Bob redeemed USDe: %s", bobRedeemed_USDe);

        // post-redemption assertions
        assertEq(
            aliceRedeemed_USDe_reported,
            aliceRedeemed_USDe_actual,
            "Alice's reported and actual USDe redemption amounts should match"
        );

        assertGe(
            bobRedeemed_pUSDe_reported,
            bobShares_pUSDe,
            "Bob should redeem at least the same amount of pUSDe as his original deposit"
        );

        assertGe(
            bobRedeemed_USDe, userDeposit, "Bob should redeem at least the same amount of USDe as his initial deposit"
        );

        assertLe(
            aliceRedeemed_USDe_actual,
            userDeposit,
            "Alice should redeem no more than the same amount of USDe as her initial deposit"
        );
    }
}
```

The following Echidna optimization test can also be run to maximise this discrepancy:

```solidity
// SPDX-License-Identifier: GPL-2.0
pragma solidity ^0.8.0;

import {BaseSetup} from "@chimera/BaseSetup.sol";
import {CryticAsserts} from "@chimera/CryticAsserts.sol";
import {vm} from "@chimera/Hevm.sol";

import {pUSDeVault} from "contracts/predeposit/pUSDeVault.sol";
import {yUSDeVault} from "contracts/predeposit/yUSDeVault.sol";
import {MockUSDe} from "contracts/test/MockUSDe.sol";
import {MockStakedUSDe} from "contracts/test/MockStakedUSDe.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

// echidna . --contract CryticRoundingTester --config echidna_rounding.yaml --format text --workers 16 --test-limit 1000000
contract CryticRoundingTester is BaseSetup, CryticAsserts {
    uint256 constant MIN_SHARES = 0.1 ether;

    MockUSDe USDe;
    MockStakedUSDe sUSDe;
    pUSDeVault pUSDe;
    yUSDeVault yUSDe;

    address owner;
    address alice = address(uint160(uint256(keccak256(abi.encodePacked("alice")))));
    address bob = address(uint160(uint256(keccak256(abi.encodePacked("bob")))));
    uint256 severity;

    constructor() payable {
        setup();
    }

    function setup() internal virtual override {
        owner = msg.sender;

        USDe = new MockUSDe();
        sUSDe = new MockStakedUSDe(USDe, owner, owner);

        pUSDe = pUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new pUSDeVault()),
                    abi.encodeWithSelector(pUSDeVault.initialize.selector, owner, USDe, sUSDe)
                )
            )
        );

        yUSDe = yUSDeVault(
            address(
                new ERC1967Proxy(
                    address(new yUSDeVault()),
                    abi.encodeWithSelector(yUSDeVault.initialize.selector, owner, USDe, sUSDe, pUSDe)
                )
            )
        );

        vm.startPrank(owner);
        pUSDe.setDepositsEnabled(true);
        pUSDe.setWithdrawalsEnabled(true);
        pUSDe.updateYUSDeVault(address(yUSDe));

        // deposit USDe and burn minimum shares to avoid reverting on redemption
        uint256 initialUSDeAmount = pUSDe.previewMint(MIN_SHARES);
        USDe.mint(owner, initialUSDeAmount);
        USDe.approve(address(pUSDe), initialUSDeAmount);
        pUSDe.mint(MIN_SHARES, address(0xdead));
        vm.stopPrank();

        if (pUSDe.balanceOf(address(0xdead)) != MIN_SHARES) {
            revert("address(0xdead) should have MIN_SHARES shares of pUSDe");
        }
    }

    function target(uint256 aliceDeposit, uint256 bobDeposit, uint256 sUSDeYieldAmount) public {
        aliceDeposit = between(aliceDeposit, 1, 100_000 ether);
        bobDeposit = between(bobDeposit, 1, 100_000 ether);
        sUSDeYieldAmount = between(sUSDeYieldAmount, 1, 500_000 ether);
        precondition(aliceDeposit <= 100_000 ether);
        precondition(bobDeposit <= 100_000 ether);
        precondition(sUSDeYieldAmount <= 500_000 ether);

        // fund users
        USDe.mint(alice, aliceDeposit);
        USDe.mint(bob, bobDeposit);

        // alice deposits into pUSDe
        vm.startPrank(alice);
        USDe.approve(address(pUSDe), aliceDeposit);
        uint256 aliceShares_pUSDe = pUSDe.deposit(aliceDeposit, alice);
        vm.stopPrank();

        // bob deposits into pUSDe
        vm.startPrank(bob);
        USDe.approve(address(pUSDe), bobDeposit);
        uint256 bobShares_pUSDe = pUSDe.deposit(bobDeposit, bob);
        vm.stopPrank();

        // setup assertions
        eq(pUSDe.balanceOf(alice), aliceShares_pUSDe, "Alice should have shares equal to her deposit");
        eq(pUSDe.balanceOf(bob), bobShares_pUSDe, "Bob should have shares equal to his deposit");

        {
            // phase change
            uint256 initialAdminTransferAmount = 1e6;
            vm.startPrank(owner);
            USDe.mint(owner, initialAdminTransferAmount);
            USDe.approve(address(pUSDe), initialAdminTransferAmount);
            pUSDe.deposit(initialAdminTransferAmount, address(yUSDe));
            pUSDe.startYieldPhase();
            yUSDe.setDepositsEnabled(true);
            yUSDe.setWithdrawalsEnabled(true);
            vm.stopPrank();
        }

        // bob deposits into yUSDe
        vm.startPrank(bob);
        pUSDe.approve(address(yUSDe), bobShares_pUSDe);
        uint256 bobShares_yUSDe = yUSDe.deposit(bobShares_pUSDe, bob);
        vm.stopPrank();

        // simulate sUSDe yield transfer
        USDe.mint(address(sUSDe), sUSDeYieldAmount);

        // alice redeems from pUSDe
        uint256 aliceBalanceBefore_sUSDe = sUSDe.balanceOf(alice);
        vm.prank(alice);
        uint256 aliceRedeemed_USDe_reported = pUSDe.redeem(aliceShares_pUSDe, alice, alice);
        uint256 aliceRedeemed_sUSDe = sUSDe.balanceOf(alice) - aliceBalanceBefore_sUSDe;
        uint256 aliceRedeemed_USDe_actual = sUSDe.previewRedeem(aliceRedeemed_sUSDe);

        // bob redeems from yUSDe
        uint256 bobBalanceBefore_sUSDe = sUSDe.balanceOf(bob);
        vm.prank(bob);
        uint256 bobRedeemed_pUSDe_reported = yUSDe.redeem(bobShares_yUSDe, bob, bob);
        uint256 bobRedeemed_sUSDe = sUSDe.balanceOf(bob) - bobBalanceBefore_sUSDe;
        uint256 bobRedeemed_USDe = sUSDe.previewRedeem(bobRedeemed_sUSDe);

        // optimize
        if (aliceRedeemed_USDe_actual > aliceDeposit) {
            uint256 diff = aliceRedeemed_USDe_actual - aliceDeposit;
            if (diff > severity) {
                severity = diff;
            }
        }
    }

    function echidna_opt_severity() public view returns (uint256) {
        return severity;
    }
}
```

Config:
```yaml
testMode: "optimization"
prefix: "echidna_"
coverage: true
corpusDir: "echidna_rounding"
balanceAddr: 0x1043561a8829300000
balanceContract: 0x1043561a8829300000
filterFunctions: []
cryticArgs: ["--foundry-compile-all"]
deployer: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496"
contractAddr: "0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496"
shrinkLimit: 100000
```

Output:
```bash
echidna_opt_severity: max value: 444330
```

**Recommended Mitigation:** Rather than calling `previewWithdraw()` which rounds up, call `convertToShares()` which rounds down:

```solidity
function previewWithdraw(uint256 assets) public view virtual override returns (uint256) {
    return _convertToShares(assets, Math.Rounding.Up);
}

function convertToShares(uint256 assets) public view virtual override returns (uint256) {
    return _convertToShares(assets, Math.Rounding.Down);
}
```

**Strata:** Fixed in commit [59fcf23](https://github.com/Strata-Money/contracts/commit/59fcf239a9089d14f02621a7f692bcda6c85690e).

**Cyfrin:** Verified. The sUSDe to transfer out to the receiver is now calculated using `convertToShares()` which rounds down.

\clearpage

---
### Example 2

**Auto Label:** Failure to validate asset balances before initiating withdrawals, leading to reverts due to zero-amount validation checks in lending protocols.  

**Original Text Preview:**

**Severity:** Low

**Path:** src/stHYPEWithdrawalModule.sol#L312

**Description:** [AAVE V3 Pool](https://github.com/aave/aave-v3-core/blob/782f51917056a53a2c228701058a6c3fb233684a/contracts/protocol/libraries/logic/ValidationLogic.sol#L66-L71) reverts when withdrawing zero amount:
```
function validateWithdraw(
    DataTypes.ReserveCache memory reserveCache,
    uint256 amount,
    uint256 userBalance
  ) internal view {
    require(amount != 0, Errors.INVALID_AMOUNT);
```
Thus, when updating a lending module (`AaveLendingModule.sol`) with `assetBalance() = 0` via `stHYPEWithdrawalModule::setProposedLendingModule` it will revert:
```
if (address(lendingModule) != address(0)) {
    lendingModule.withdraw(lendingModule.assetBalance(), address(this));
}
```
```
function setProposedLendingModule() external onlyOwner {
    if (lendingModuleProposal.startTimestamp > block.timestamp) {
        revert stHYPEWithdrawalModule__setProposedLendingModule_ProposalNotActive();
    }

    if (lendingModuleProposal.startTimestamp == 0) {
        revert stHYPEWithdrawalModule__setProposedLendingModule_InactiveProposal();
    }

    if (address(lendingModule) != address(0)) {
        lendingModule.withdraw(lendingModule.assetBalance(), address(this));
    }

    lendingModule = ILendingModule(lendingModuleProposal.lendingModule);
    delete lendingModuleProposal;
    emit LendingModuleSet(address(lendingModule));
}
```

**Remediation:**  Consider checking the `assetBalance()` to not being 0 before calling `lendingModule.withdraw()` inside `stHYPEWithdrawalModule::setProposedLendingModule`.

For example:
```
if (address(lendingModule) != address(0) && lendingModule.assetBalance() != 0) {
    lendingModule.withdraw(lendingModule.assetBalance(), address(this));
}
```

**Status:** Fixed


- - -

---
### Example 3

**Auto Label:** Failure to validate asset types or liquidation states leads to unauthorized collateral transfers or loss of user control, enabling permanent asset lock or unintended value loss.  

**Original Text Preview:**

## Summary

The `LendingPool::depositNFT` function does not check if a user is under liquidation and if their grace period has expired. This allows users to deposit additional NFTs in an attempt to increase their health factor, but if they fail to close the liquidation because of the grace period expiration, they will lose all collateral including the newly deposited NFTs.

## Vulnerability Details

The `LendingPool::depositNFT` function lacks checks for:

1. Whether the user is under liquidation (`isUnderLiquidation[msg.sender]`)
2. Whether their grace period has expired (`block.timestamp > liquidationStartTime[msg.sender] + liquidationGracePeriod`)

This allows users to deposit additional NFTs even when their position is already eligible for liquidation and after the grace period has expired, resulting in the loss of more collateral than necessary.

## Impact

Users under liquidation can lose additional collateral by attempting to save their position after the grace period has expired. Since `finalizeLiquidation()` transfers all NFTs to the Stability Pool, any newly deposited NFTs will also be liquidated, causing users to lose more value than their original debt.

For example:

1. User has 100k debt and 125k in NFT collateral
2. Position becomes liquidatable and grace period expires
3. User deposits additional 50k NFT trying to save position
4. Liquidation executes, user loses 175k collateral to cover 100k debt

## Tools Used

Manual review

## Proof of Concept

Add the following test case to the `test/unit/core/pools/LendingPool/LendingPool.test.js` file in the `Liquidation` section:

```javascript
it("should demonstrate loss of additional collateral after grace period", async function () {
    // Set Stability Pool address (using owner for this test)
    await lendingPool.connect(owner).setStabilityPool(owner.address);
    await token.mint(owner.address, ethers.parseEther("100"));

    // Decrease house price and initiate liquidation
    await raacHousePrices.setHousePrice(1, ethers.parseEther("90"));

    // Initiate liquidation
    await lendingPool.connect(user2).initiateLiquidation(user1.address);

    // Advance time beyond grace period (72 hours)
    await ethers.provider.send("evm_increaseTime", [72 * 60 * 60 + 1]);
    await ethers.provider.send("evm_mine");

    // Verify the health factor is below the liquidation threshold
    const healthFactor = await lendingPool.calculateHealthFactor(user1.address);
    const healthFactorLiquidationThreshold = await lendingPool.healthFactorLiquidationThreshold();
    expect(healthFactor).to.be.lt(healthFactorLiquidationThreshold);

    // Mint new NFT to User1
    const tokenId = 2;
    await raacHousePrices.setHousePrice(tokenId, ethers.parseEther("100"));
    const housePrice = await raacHousePrices.tokenToHousePrice(tokenId);
    await token.mint(user1.address, housePrice);
    await token.connect(user1).approve(raacNFT.target, housePrice);
    await raacNFT.connect(user1).mint(tokenId, housePrice);

    // Deposit new NFT to collateral
    await raacNFT.connect(user1).approve(lendingPool.target, tokenId);
    await lendingPool.connect(user1).depositNFT(tokenId);

    // Verify the new health factor is below the liquidation threshold
    const healthFactorAfterDeposit = await lendingPool.calculateHealthFactor(user1.address);
    expect(healthFactorAfterDeposit).to.be.gt(healthFactorLiquidationThreshold);

    // User1 is still under liquidation 
    expect(await lendingPool.isUnderLiquidation(user1.address)).to.be.true;

    // User1 is not able to close liquidation
    await expect(lendingPool.connect(user1).closeLiquidation())
    .to.be.revertedWithCustomError(lendingPool, "GracePeriodExpired");

    // Stability Pool closes liquidation
    await expect(lendingPool.connect(owner).finalizeLiquidation(user1.address))
    .to.emit(lendingPool, "LiquidationFinalized")
    
    // Verify that the user is no longer under liquidation
    expect(await lendingPool.isUnderLiquidation(user1.address)).to.be.false;

    // Verify that the NFT has been transferred to the Stability Pool
    expect(await raacNFT.ownerOf(1)).to.equal(owner.address);
    expect(await raacNFT.ownerOf(tokenId)).to.equal(owner.address);
});
```

## Recommendations

Add liquidation status checks to the `depositNFT` function:

```diff
function depositNFT(uint256 tokenId) external nonReentrant whenNotPaused {
+   if (isUnderLiquidation[msg.sender] && 
+       block.timestamp > liquidationStartTime[msg.sender] + liquidationGracePeriod) {
+       revert CannotDepositAfterGracePeriod();
+   }
    // Rest of the function...
}
```

---
