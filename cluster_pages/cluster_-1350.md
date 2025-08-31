# Cluster -1350

**Rank:** #207  
**Count:** 54  

## Label
Reentrancy vulnerabilities caused by improper state updates and premature state checks, enabling attackers to manipulate token supply or trigger invalid operations through external callbacks.

## Cluster Information
- **Total Findings:** 54

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
### Example 3

**Auto Label:** Reentrancy vulnerabilities enable attackers to drain funds via recursive calls, exploiting missing checks and state updates, leading to total fund loss and system failure.  

**Original Text Preview:**

Admin Can not add already removed strategy, because
`addStrategy` function calls `connect` in strategy but it will revert since can be called once:

```solidity
    function addStrategy(IStrategy s) external override onlyRole(AccessControlStorage.DEFAULT_ADMIN_ROLE) {
        OperationalTreasuryStorage.SetUp storage setUp = OperationalTreasuryStorage.layout().setUp;

        if (setUp.acceptedStrategy[s]) revert StrategyAlreadyAdded();
        _connect(s, setUp);
    }

    function _connect(IStrategy s, OperationalTreasuryStorage.SetUp storage strg) internal {
        s.connect();
        strg.acceptedStrategy[s] = true;
    }
````

`connect` function in `AdminStrategy` contract can be called only once:

```solidity

    function connect() external override {
        StrategyStorage.SetUp storage setUp = StrategyStorage.layout().setUp;
        IOperationalTreasury treasury_ = IOperationalTreasury(msg.sender);
        if (address(setUp.treasury) != address(0)) revert TreasuryAlreadySet();
        setUp.treasury = treasury_;
    }
```

---
