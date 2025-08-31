# Cluster -1046

**Rank:** #140  
**Count:** 100  

## Label
**Insufficient state validation and threshold checks enable attackers to manipulate debt settlements, freeze withdrawals, or exploit pricing inaccuracies, leading to unfair advantages and system instability.**

## Cluster Information
- **Total Findings:** 100

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

**Auto Label:** Inaccurate asset balance tracking due to flawed state updates and inconsistent share-to-asset calculations, leading to user fund misrepresentation and exploitable accounting discrepancies.  

**Original Text Preview:**

**Description:** `BunniQuoter::quoteDeposit` assumes that caller provided the correct vault fee; however, if a vault fee of zero is passed for a vault that has a non-zero vault fee, the returned quote will be incorrect as shown in the PoC below. Additionally, for the case where there is an existing share supply, this function is missing validation against existing token amounts. Specifically, when both token amounts are zero, the `BunniHub` reverts execution whereas the quoter will return success along with a share amount of `type(uint256).max`:
```solidity
    ...
    if (existingShareSupply == 0) {
        // ensure that the added amounts are not too small to mess with the shares math
        if (addedAmount0 < MIN_DEPOSIT_BALANCE_INCREASE && addedAmount1 < MIN_DEPOSIT_BALANCE_INCREASE) {
            revert BunniHub__DepositAmountTooSmall();
        }
        // no existing shares, just give WAD
        shares = WAD - MIN_INITIAL_SHARES;
        // prevent first staker from stealing funds of subsequent stakers
        // see https://code4rena.com/reports/2022-01-sherlock/#h-01-first-user-can-steal-everyone-elses-tokens
        shareToken.mint(address(0), MIN_INITIAL_SHARES, address(0));
    } else {
        // given that the position may become single-sided, we need to handle the case where one of the existingAmount values is zero
@>      if (existingAmount0 == 0 && existingAmount1 == 0) revert BunniHub__ZeroSharesMinted();
        shares = FixedPointMathLib.min(
            existingAmount0 == 0 ? type(uint256).max : existingShareSupply.mulDiv(addedAmount0, existingAmount0),
            existingAmount1 == 0 ? type(uint256).max : existingShareSupply.mulDiv(addedAmount1, existingAmount1)
        );
        if (shares == 0) revert BunniHub__ZeroSharesMinted();
    }
    ...
```

`BunniQuoter::quoteWithdraw` is missing the validation required for queued withdrawals if there exists an am-AMM manager which can be fetched through the `getTopBid()` function that uses an static call:
```diff
    function quoteWithdraw(address sender, IBunniHub.WithdrawParams calldata params)
        external
        view
        override
        returns (bool success, uint256 amount0, uint256 amount1)
    {
        PoolId poolId = params.poolKey.toId();
        PoolState memory state = hub.poolState(poolId);
        IBunniHook hook = IBunniHook(address(params.poolKey.hooks));

++      IAmAmm.Bid memory topBid = hook.getTopBid(poolId);
++      if (hook.getAmAmmEnabled(poolId) && topBid.manager != address(0) && !params.useQueuedWithdrawal) {
++          return (false, 0, 0);
++      }
        ...
    }
```

This function should also validate whether the sender has already an existing queued withdrawal. It is not currently possible to check this because the `BunniHub` does not expose any function to fetch queued withdrawals; however, it should be ensured that if `useQueuedWithdrawal` is true, the user has an existing queued withdrawal that is inside the executable timeframe. In this scenario, the token amount computations should be performed taking the amount of shares from the queued withdrawal.

**Proof of Concept**
First create the following `ERC4626FeeMock` inside `test/mocks/ERC4626Mock.sol`:

```solidity
contract ERC4626FeeMock is ERC4626 {
    address internal immutable _asset;
    uint256 public fee;
    uint256 internal constant MAX_FEE = 10000;

    constructor(IERC20 asset_, uint256 _fee) {
        _asset = address(asset_);
        if(_fee > MAX_FEE) revert();
        fee = _fee;
    }

    function setFee(uint256 newFee) external {
        if(newFee > MAX_FEE) revert();
        fee = newFee;
    }

    function deposit(uint256 assets, address to) public override returns (uint256 shares) {
        return super.deposit(assets - assets * fee / MAX_FEE, to);
    }

    function asset() public view override returns (address) {
        return _asset;
    }

    function name() public pure override returns (string memory) {
        return "MockERC4626";
    }

    function symbol() public pure override returns (string memory) {
        return "MOCK-ERC4626";
    }
}
```

And add it into the `test/BaseTest.sol` imports:

```diff
--  import {ERC4626Mock} from "./mocks/ERC4626Mock.sol";
++  import {ERC4626Mock, ERC4626FeeMock} from "./mocks/ERC4626Mock.sol";
```

The following test can now be run inside `test/BunniHub.t.sol`:
```solidity
function test_QuoterAssumingCorrectVaultFee() public {
    ILiquidityDensityFunction uniformDistribution = new UniformDistribution(address(hub), address(bunniHook), address(quoter));
    Currency currency0 = Currency.wrap(address(token0));
    Currency currency1 = Currency.wrap(address(token1));
    ERC4626FeeMock feeVault0 = new ERC4626FeeMock(token0, 0);
    ERC4626 vault0_ = ERC4626(address(feeVault0));
    ERC4626 vault1_ = ERC4626(address(0));
    IBunniToken bunniToken;
    PoolKey memory key;
    (bunniToken, key) = hub.deployBunniToken(
        IBunniHub.DeployBunniTokenParams({
            currency0: currency0,
            currency1: currency1,
            tickSpacing: TICK_SPACING,
            twapSecondsAgo: TWAP_SECONDS_AGO,
            liquidityDensityFunction: uniformDistribution,
            hooklet: IHooklet(address(0)),
            ldfType: LDFType.DYNAMIC_AND_STATEFUL,
            ldfParams: bytes32(abi.encodePacked(ShiftMode.STATIC, int24(-5) * TICK_SPACING, int24(5) * TICK_SPACING)),
            hooks: bunniHook,
            hookParams: abi.encodePacked(
                FEE_MIN,
                FEE_MAX,
                FEE_QUADRATIC_MULTIPLIER,
                FEE_TWAP_SECONDS_AGO,
                POOL_MAX_AMAMM_FEE,
                SURGE_HALFLIFE,
                SURGE_AUTOSTART_TIME,
                VAULT_SURGE_THRESHOLD_0,
                VAULT_SURGE_THRESHOLD_1,
                REBALANCE_THRESHOLD,
                REBALANCE_MAX_SLIPPAGE,
                REBALANCE_TWAP_SECONDS_AGO,
                REBALANCE_ORDER_TTL,
                true, // amAmmEnabled
                ORACLE_MIN_INTERVAL,
                MIN_RENT_MULTIPLIER
            ),
            vault0: vault0_,
            vault1: vault1_,
            minRawTokenRatio0: 0.20e6,
            targetRawTokenRatio0: 0.30e6,
            maxRawTokenRatio0: 0.40e6,
            minRawTokenRatio1: 0,
            targetRawTokenRatio1: 0,
            maxRawTokenRatio1: 0,
            sqrtPriceX96: TickMath.getSqrtPriceAtTick(0),
            name: bytes32("BunniToken"),
            symbol: bytes32("BUNNI-LP"),
            owner: address(this),
            metadataURI: "metadataURI",
            salt: bytes32(0)
        })
    );

    // make initial deposit to avoid accounting for MIN_INITIAL_SHARES
    uint256 depositAmount0 = 1e18 + 1;
    uint256 depositAmount1 = 1e18 + 1;
    address firstDepositor = makeAddr("firstDepositor");
    vm.startPrank(firstDepositor);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, firstDepositor, depositAmount0 * 100);
    _mint(key.currency1, firstDepositor, depositAmount1 * 100);

    // deposit tokens
    IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        recipient: firstDepositor,
        refundRecipient: firstDepositor,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });

    vm.prank(firstDepositor);
    (uint256 sharesFirstDepositor, uint256 firstDepositorAmount0In, uint256 firstDepositorAmount1In) = hub.deposit(depositParams);

    IdleBalance idleBalanceBefore = hub.idleBalance(key.toId());
    (uint256 idleAmountBefore, bool isToken0Before) = IdleBalanceLibrary.fromIdleBalance(idleBalanceBefore);
    feeVault0.setFee(1000);     // 10% fee

    depositAmount0 = 1e18;
    depositAmount1 = 1e18;
    address secondDepositor = makeAddr("secondDepositor");
    vm.startPrank(secondDepositor);
    token0.approve(address(PERMIT2), type(uint256).max);
    token1.approve(address(PERMIT2), type(uint256).max);
    PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
    PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
    vm.stopPrank();

    // mint tokens
    _mint(key.currency0, secondDepositor, depositAmount0);
    _mint(key.currency1, secondDepositor, depositAmount1);

    // deposit tokens
    depositParams = IBunniHub.DepositParams({
        poolKey: key,
        amount0Desired: depositAmount0,
        amount1Desired: depositAmount1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        recipient: secondDepositor,
        refundRecipient: secondDepositor,
        vaultFee0: 0,
        vaultFee1: 0,
        referrer: address(0)
    });
    (bool success, uint256 previewedShares, uint256 previewedAmount0, uint256 previewedAmount1) = quoter.quoteDeposit(address(this), depositParams);

    vm.prank(secondDepositor);
    (uint256 sharesSecondDepositor, uint256 secondDepositorAmount0In, uint256 secondDepositorAmount1In) = hub.deposit(depositParams);

    console.log("Quote deposit will be successful?", success);
    console.log("Quoted shares to mint", previewedShares);
    console.log("Quoted token0 amount to use", previewedAmount0);
    console.log("Quoted token1 amount to use", previewedAmount1);
    console.log("---------------------------------------------------");
    console.log("Actual shares minted", sharesSecondDepositor);
    console.log("Actual token0 amount used", secondDepositorAmount0In);
    console.log("Actual token1 amount used", secondDepositorAmount1In);
}
```

Output:
```
Ran 1 test for test/BunniHub.t.sol:BunniHubTest
[PASS] test_QuoterAssumingCorrectVaultFee() (gas: 4773069)
Logs:
  Quote deposit will be successful? true
  Quoted shares to mint 1000000000000000000
  Quoted token0 amount to use 1000000000000000000
  Quoted token1 amount to use 1000000000000000000
  ---------------------------------------------------
  Actual shares minted 930000000000000000
  Actual token0 amount used 930000000000000000
  Actual token1 amount used 1000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.71s (4.45ms CPU time)

Ran 1 test suite in 1.71s (1.71s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

**Recommended Mitigation:** Implement the missing validation as described above.

**Bacon Labs:** Fixed in [PR \#122](https://github.com/timeless-fi/bunni-v2/pull/122).

**Cyfrin:** Verified, additional validation has been added to `BunniQuoter` to match the behavior of `BunniHub`.

---
### Example 3

**Auto Label:** **Insufficient state validation and threshold checks enable attackers to manipulate debt settlements, freeze withdrawals, or exploit pricing inaccuracies, leading to unfair advantages and system instability.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-yearn-ybold-judging/issues/90 

## Found by 
Robert

### Summary

Because there is no maximum amount on `dustThreshold`, management can increase it to an absurdly high number. By doing this, they stop all auctions from happening. Users will continue to lose BOLD to liquidations but will not be able to receive anything in return, therefore leading to permanent locking of funds users are owed.

### Root Cause

[Strategy.sol#L155:](https://github.com/sherlock-audit/2025-05-yearn-ybold/blob/main/yv3-liquityv2-sp-strategy/src/Strategy.sol#L155)

/// @notice Set the dust threshold for the strategy
/// @param _dustThreshold New dust threshold
function setDustThreshold(
        uint256 _dustThreshold
    ) external onlyManagement {
        require(_dustThreshold >= MIN_DUST_THRESHOLD, "!minDust");
        dustThreshold = _dustThreshold;
    }

This function allows management to set dust threshold. It has a minimum amount, but no maximum. If a maximum is added it will stop management from being able to lock user funds.

### Internal Pre-conditions

1. Admin sets `dustThreshold` absurdly high
2. Users funds owed to them from liquidations are locked as collateral in the strategy contract

### External Pre-conditions

1. Liquidations must occur for it to have an actual effect

### Attack Path

1. Management sees a large amount of incoming liquidations
2. Management sets dustThreshold very high so auctions cannot occur
3. User funds taken by liquidations are permanently locked

### Impact

The attacker doesn't inherently gain anything from the attacker but users suffer from locking of their funds. If done directly before a large amount of liquidations, this could be a significant amount of user funds.

### PoC

_No response_

### Mitigation

Add a maximum amount to dust threshold such as 1e18

---
