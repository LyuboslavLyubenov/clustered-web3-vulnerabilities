# Cluster -1130

**Rank:** #49  
**Count:** 361  

## Label
Failure to synchronize or validate state during critical operations leads to irreversible fund loss, incorrect balances, or denial-of-service, enabling irreversible or erroneous token transfers.

## Cluster Information
- **Total Findings:** 361

## Examples

### Example 1

**Auto Label:** Insecure token approval mechanisms leading to unauthorized transfers, silent failures, or denial-of-service due to improper allowance handling and lack of validation.  

**Original Text Preview:**

## Severity

**Impact**: Medium, because functionality won't work

**Likelihood**: Medium, because USDT is a common token

## Description

Code uses the `approve` method to set allowance for ERC20 tokens in `setSwapperAllowance`. This will cause revert if the target ERC20 was a non-standard token that has different function signature for `approve()` function. Tokens like USDT will cause revert for this function, so they can't be used as reward token, input token and underlying asset.

```solidity
    function setSwapperAllowance(uint256 _amount) public onlyAdmin {
        address swapperAddress = address(swapper);

        for (uint256 i = 0; i < rewardLength; i++) {
            if (rewardTokens[i] == address(0)) break;
            IERC20Metadata(rewardTokens[i]).approve(swapperAddress, _amount);
        }
        for (uint256 i = 0; i < inputLength; i++) {
            if (address(inputs[i]) == address(0)) break;
            inputs[i].approve(swapperAddress, _amount);
        }
        asset.approve(swapperAddress, _amount);
    }
```

## Recommendations

Use `SafeERC20`'s `forceApprove` method instead to support all the ERC20 tokens.

---
### Example 2

**Auto Label:** Insecure token approval mechanisms leading to unauthorized transfers, silent failures, or denial-of-service due to improper allowance handling and lack of validation.  

**Original Text Preview:**

**Description:** Use [`SafeERC20::forceApprove`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L101-L108) when dealing with a range of potential tokens instead of standard `IERC20::approve`:
```solidity
predeposit/yUSDeDepositor.sol
58:        pUSDe.approve(address(yUSDe), amount);

predeposit/pUSDeVault.sol
178:        USDe.approve(address(sUSDe), USDeAssets);

predeposit/pUSDeDepositor.sol
86:            asset.approve(address(vault), amount);
98:        sUSDe.approve(address(pUSDe), amount);
110:        USDe.approve(address(pUSDe), amount);
122:        token.approve(swapInfo.router, amount);
```

**Strata:** Fixed in commit [f258bdc](https://github.com/Strata-Money/contracts/commit/f258bdcc49b87a2f8658b150bc3e3597a5187816).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Race conditions in withdrawal mechanisms due to insufficient state validation, timing flaws, or lack of locking, enabling unauthorized or premature token extraction and denial-of-service.  

**Original Text Preview:**

**Description:** Unchecked math is used within `BunniHubLogic::queueWithdraw` to wrap around if the sum of the block timestamp with `WITHDRAW_DELAY` exceeds the maximum `uint56`:

```solidity
    // update queued withdrawal
    // use unchecked to get unlockTimestamp to overflow back to 0 if overflow occurs
    // which is fine since we only care about relative time
    uint56 newUnlockTimestamp;
    unchecked {
@>      newUnlockTimestamp = uint56(block.timestamp) + WITHDRAW_DELAY;
    }
    if (queued.shareAmount != 0) {
        // requeue expired queued withdrawal
@>      if (queued.unlockTimestamp + WITHDRAW_GRACE_PERIOD >= block.timestamp) {
            revert BunniHub__NoExpiredWithdrawal();
        }
        s.queuedWithdrawals[id][msgSender].unlockTimestamp = newUnlockTimestamp;
    } else {
        // create new queued withdrawal
        if (params.shares == 0) revert BunniHub__ZeroInput();
        s.queuedWithdrawals[id][msgSender] =
            QueuedWithdrawal({shareAmount: params.shares, unlockTimestamp: newUnlockTimestamp});
    }
```

This yields a `newUnlockTimestamp` that is modulo the max `uint56`; however, note the addition of `WITHDRAW_GRACE_PERIOD` to the unlock timestamp of an existing queued withdrawal that is also present in `BunniHubLogic::withdraw`:

```solidity
    if (params.useQueuedWithdrawal) {
        // use queued withdrawal
        // need to withdraw the full queued amount
        QueuedWithdrawal memory queued = s.queuedWithdrawals[poolId][msgSender];
        if (queued.shareAmount == 0 || queued.unlockTimestamp == 0) revert BunniHub__QueuedWithdrawalNonexistent();
@>      if (block.timestamp < queued.unlockTimestamp) revert BunniHub__QueuedWithdrawalNotReady();
@>      if (queued.unlockTimestamp + WITHDRAW_GRACE_PERIOD < block.timestamp) revert BunniHub__GracePeriodExpired();
        shares = queued.shareAmount;
        s.queuedWithdrawals[poolId][msgSender].shareAmount = 0; // don't delete the struct to save gas later
        state.bunniToken.burn(address(this), shares); // BunniTokens were deposited to address(this) earlier with queueWithdraw()
    }
```

This logic is not implemented correctly and has a couple of implications in various scenarios:
* If the unlock timestamp for a given queued withdrawal with the `WITHDRAW_DELAY` does not overflow, but with the addition of the `WITHDRAW_GRACE_PERIOD` it does, then queued withdrawals will revert due to overflowing `uint56` outside of the unchecked block and it will not be possible to re-queue expired withdrawals for the same reasoning.
* If the unlock timestamp for a given queued withdrawal with the `WITHDRAW_DELAY` overflows and wraps around, it is possible to immediately re-queue an “expired” withdrawal, since by comparison the block timestamp in `uint256` will be a lot larger than the unlock timestamp the `WITHDRAW_GRACE_PERIOD` applied; however, it will not be possible to execute such a withdrawal (even without replacement) since the block timestamp will always be significantly larger after wrapping around.

**Impact:** While it is unlikely `block.timestamp` will reach close to overflowing a `uint56` within the lifetime of the Sun, the intended unchecked logic is implemented incorrectly and would prevent queued withdrawals from executing correctly. This could be especially problematic if the width of the data type were reduced assuming no issues are present.

**Proof of Concept:** The following tests can be run from within `test/BunniHub.t.sol`:
```solidity
function test_queueWithdrawPoC1() public {
    uint256 depositAmount0 = 1 ether;
    uint256 depositAmount1 = 1 ether;
    (IBunniToken bunniToken, PoolKey memory key) = _deployPoolAndInitLiquidity();

    // make deposit
    (uint256 shares,,) = _makeDepositWithFee({
        key_: key,
        depositAmount0: depositAmount0,
        depositAmount1: depositAmount1,
        depositor: address(this),
        vaultFee0: 0,
        vaultFee1: 0,
        snapLabel: ""
    });

    // bid in am-AMM auction
    PoolId id = key.toId();
    bunniToken.approve(address(bunniHook), type(uint256).max);
    uint128 minRent = uint128(bunniToken.totalSupply() * MIN_RENT_MULTIPLIER / 1e18);
    uint128 rentDeposit = minRent * 2 days;
    bunniHook.bid(id, address(this), bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), minRent * 2, rentDeposit);
    shares -= rentDeposit;

    // wait until address(this) is the manager
    skipBlocks(K);
    assertEq(bunniHook.getTopBid(id).manager, address(this), "not manager yet");

    vm.warp(type(uint56).max - 1 minutes);

    // queue withdraw
    bunniToken.approve(address(hub), type(uint256).max);
    hub.queueWithdraw(IBunniHub.QueueWithdrawParams({poolKey: key, shares: shares.toUint200()}));
    assertEqDecimal(bunniToken.balanceOf(address(hub)), shares, DECIMALS, "didn't take shares");

    // wait 1 minute
    skip(1 minutes);

    // withdraw
    IBunniHub.WithdrawParams memory withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: address(this),
        shares: shares,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: true
    });
    vm.expectRevert();
    hub.withdraw(withdrawParams);
}

function test_queueWithdrawPoC2() public {
    uint256 depositAmount0 = 1 ether;
    uint256 depositAmount1 = 1 ether;
    (IBunniToken bunniToken, PoolKey memory key) = _deployPoolAndInitLiquidity();

    // make deposit
    (uint256 shares,,) = _makeDepositWithFee({
        key_: key,
        depositAmount0: depositAmount0,
        depositAmount1: depositAmount1,
        depositor: address(this),
        vaultFee0: 0,
        vaultFee1: 0,
        snapLabel: ""
    });

    // bid in am-AMM auction
    PoolId id = key.toId();
    bunniToken.approve(address(bunniHook), type(uint256).max);
    uint128 minRent = uint128(bunniToken.totalSupply() * MIN_RENT_MULTIPLIER / 1e18);
    uint128 rentDeposit = minRent * 2 days;
    bunniHook.bid(id, address(this), bytes6(abi.encodePacked(uint24(1e3), uint24(2e3))), minRent * 2, rentDeposit);
    shares -= rentDeposit;

    // wait until address(this) is the manager
    skipBlocks(K);
    assertEq(bunniHook.getTopBid(id).manager, address(this), "not manager yet");

    vm.warp(type(uint56).max);

    // queue withdraw
    bunniToken.approve(address(hub), type(uint256).max);
    hub.queueWithdraw(IBunniHub.QueueWithdrawParams({poolKey: key, shares: shares.toUint200()}));
    assertEqDecimal(bunniToken.balanceOf(address(hub)), shares, DECIMALS, "didn't take shares");

    // wait 1 minute
    skip(1 minutes);

    // re-queue before expiry
    hub.queueWithdraw(IBunniHub.QueueWithdrawParams({poolKey: key, shares: shares.toUint200()}));

    // withdraw
    IBunniHub.WithdrawParams memory withdrawParams = IBunniHub.WithdrawParams({
        poolKey: key,
        recipient: address(this),
        shares: shares,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp,
        useQueuedWithdrawal: true
    });
    vm.expectRevert();
    hub.withdraw(withdrawParams);
}
```

**Recommended Mitigation:** Unsafely downcast all other usage of `block.timestamp` to `uint56` such that it is allowed to silently overflow when compared with unlock timestamps computed in the same way.

**Bacon Labs:** Fixed in [PR \#126](https://github.com/timeless-fi/bunni-v2/pull/126).

**Cyfrin:** Verified, `uint56` overflows are now handled during queued withdrawal.

---
