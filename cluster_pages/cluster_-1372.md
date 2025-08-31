# Cluster -1372

**Rank:** #233  
**Count:** 42  

## Label
Precision loss and logic flaws in reward calculations due to improper handling of fixed-point arithmetic, incorrect loop bounds, and inconsistent state updates, leading to systematic underpayment, denial of service, and wealth leakage.

## Cluster Information
- **Total Findings:** 42

## Examples

### Example 1

**Auto Label:** Precision loss and logic flaws in reward calculations due to improper handling of fixed-point arithmetic, incorrect loop bounds, and inconsistent state updates, leading to systematic underpayment, denial of service, and wealth leakage.  

**Original Text Preview:**

In the `Flip` contract, during the `netPayout` calculation in the `flip()` function, it was noticed during testing that the user's `netPayout` is sometimes less than the `betAmount` they paid to participate in the game. For certain coin/heads combinations, the user ends up with a payout lower than the amount they bet, resulting in a loss for the player rather than a win, which makes it non-rewarding for players even if they win.

A `CoinFlipTest` test file is setup with a 10% `feePercentage` (`feePercentage` = 1000) and with a `betAmount` of 100 (where the bet token is assumed to have 18 decimals), to observe at which `numberOfCoins` to `headsRequired` combinations the calculated `netPayout` (reward) starts to be less than the game `betAmount`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;
import "forge-std/Test.sol";

contract CoinFlipTest is Test {
    uint256 public feePercentage = 1000;

    function calculatePayout(
        uint256 betAmount,
        uint256 numberOfCoins,
        uint256 headsRequired
    )
        public
        view
        returns (uint256 grossPayout, uint256 netPayout, uint256 fee)
    {
        require(numberOfCoins > 0, "Number of coins must be greater than 0");
        require(
            headsRequired > 0 && headsRequired <= numberOfCoins,
            "Invalid number of heads required"
        );

        // Calculate total number of possible outcomes (2^numberOfCoins)
        uint256 totalOutcomes = 2 ** numberOfCoins;

        // Calculate favorable outcomes (where headsRequired or more heads occur)
        uint256 favorableOutcomes = 0;

        for (uint256 i = headsRequired; i <= numberOfCoins; i++) {
            favorableOutcomes += binomialCoefficient(numberOfCoins, i);
        }

        uint256 odds = (totalOutcomes * 1e18) / favorableOutcomes;

        // Corrected gross payout calculation
        grossPayout = (betAmount * odds) / 1e18; // Removed betAmount addition

        // Fee based on original bet amount
        fee = (betAmount * feePercentage) / 10000;
        netPayout = grossPayout - fee;

        return (grossPayout, netPayout, fee);
    }

    function binomialCoefficient(
        uint256 n,
        uint256 r
    ) internal pure returns (uint256) {
        if (r > n) {
            return 0;
        }
        if (r == 0 || r == n) {
            return 1;
        }

        uint256 numerator = 1;
        uint256 denominator = 1;

        // Calculate nCr = n! / (r! * (n - r)!)
        for (uint256 i = 0; i < r; i++) {
            numerator = numerator * (n - i);
            denominator = denominator * (i + 1);
        }
        return numerator / denominator;
    }

    function test_PayoutCalculation(
        uint256 numberOfCoins,
        uint256 headsRequired
    ) public {
        // same checks implemented in `Flip.flip()`
        vm.assume(numberOfCoins >= 1 && numberOfCoins <= 20);
        vm.assume(headsRequired >= 1 && headsRequired <= numberOfCoins);
        uint256 betAmount = 100e18;

        (, uint256 netPayout, ) = calculatePayout(
            betAmount,
            numberOfCoins,
            headsRequired
        );
        console.log(
            "numberOfCoins:headsRequired :",
            numberOfCoins,
            headsRequired
        );
        console.log("betAmount:netPayout :", betAmount, netPayout);
        console.log("-------------------");

        assert(netPayout >= betAmount);
    }
}
```

The result shown below that the test starts to fail (`netPayout < betAmount`) with `numberOfCoins = 9 and headsRequired = 3` :

```bash
$ forge test --mt test_PayoutCalculation -vvvvv

Ran 1 test for test/CoinFlipTest.t.sol:CoinFlipTest
[FAIL: panic: assertion failed (0x01); counterexample: calldata=0xe8ea7ea900000000000000000000000000000000000000000000000000000000000000090000000000000000000000000000000000000000000000000000000000000003 args=[9, 3]] test_PayoutCalculation(uint256,uint256) (runs: 1,
μ: 26196, ~: 26196)
Logs:
  numberOfCoins:headsRequired : 9 3
  betAmount:netPayout : 100000000000000000000 99871244635193133000
  -------------------

Traces:
  [49533] CoinFlipTest::test_PayoutCalculation(9, 3)
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← [Return]
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← [Return]
    ├─ [0] console::log("numberOfCoins:headsRequired :", 9, 3) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("betAmount:netPayout :", 100000000000000000000 [1e20], 99871244635193133000 [9.987e19]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] console::log("-------------------") [staticcall]
    │   └─ ← [Stop]
    └─ ← [Revert] panic: assertion failed (0x01)

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 32.16ms (31.60ms CPU time)

Ran 1 test suite in 55.82ms (32.16ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/CoinFlipTest.t.sol:CoinFlipTest
[FAIL: panic: assertion failed (0x01); counterexample: calldata=0xe8ea7ea900000000000000000000000000000000000000000000000000000000000000090000000000000000000000000000000000000000000000000000000000000003 args=[9, 3]] test_PayoutCalculation(uint256,uint256) (runs: 1,
μ: 26196, ~: 26196)

Encountered a total of 1 failing tests, 0 tests succeeded
```

When tested separately for `numberOfCoins = 10  and headsRequired = 3`

```solidity
    function test_testPayoutForASpecificValues() public {
        uint256 betAmount = 100e18;
        uint256 numberOfCoins = 10;
        uint256 headsRequired = 3;
        (, uint256 netPayout, ) = calculatePayout(
            betAmount,
            numberOfCoins,
            headsRequired
        );
        console.log("betAmount:netPayout :", betAmount/1e18, netPayout/1e18);
    }
```

the result is (`netPayout = 95e18` while ` betAmount = 100e18` ):

```bash
$ forge test --mt test_testPayoutForASpecificValues -vvvvv

Ran 1 test for test/CoinFlipTest.t.sol:CoinFlipTest
[PASS] test_testPayoutForASpecificValues() (gas: 53257)
Logs:
  betAmount:netPayout : 100 95

Traces:
  [53257] CoinFlipTest::test_testPayoutForASpecificValues()
    ├─ [0] console::log("betAmount:netPayout :", 100, 95) [staticcall]
    │   └─ ← [Stop]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 12.12ms (11.56ms CPU time)

Ran 1 test suite in 34.93ms (12.12ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Recommendations:

Consider updating the `flip()` function to revert the transaction if the calculated `netPayout` is less than the `betAmount`.

---
### Example 2

**Auto Label:** Precision loss and logic flaws in reward calculations due to improper handling of fixed-point arithmetic, incorrect loop bounds, and inconsistent state updates, leading to systematic underpayment, denial of service, and wealth leakage.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-03-symm-io-stacking-judging/issues/575 

## Found by 
0xBecket, 0xCNX, A\_Failures\_True\_Power, Artur, Audinarey, Breeje, CL001, Fortis\_Audits, Kyosi, Pablo, SlayerSecurity, The\_Rezolvers, Victor\_TheOracle, X12, aslanbek, bladeee, durov, farismaulana, godwinudo, lls, newspacexyz, onthehunt, stuart\_the\_minion, wickie, zraxx

### Summary


https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/main/token/contracts/staking/SymmStaking.sol#L402-L423

_updateRewardsStates can be triggered as often as each block (2 seconds) via deposit/withdraw/claim/notifyRewardAmount

e.g. if there's 1209.6e6 USDC rewards for one week (604800 seconds)


https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/main/token/contracts/staking/SymmStaking.sol#L374

rate = 1209_600000 / 604800 = 2000 "usdc units" per second


https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/main/token/contracts/staking/SymmStaking.sol#L194-L202

if SYMM total staked supply is 1_000_000e18 (~26560 usd), and we call `deposit` each block, then [`perTokenStored` will be increased by](https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/main/token/contracts/staking/SymmStaking.sol#L412):

2 * 2000 * 1e18 / 1_000_000e18 = 4_000 / 1_000_000 = 0

Therefore, `perTokenStored` will not increase, but [`lastUpdated` will be increased](https://github.com/sherlock-audit/2025-03-symm-io-stacking/blob/main/token/contracts/staking/SymmStaking.sol#L413), therefore users will not receive any USDC rewards for staking.

In this particular example, triggering `_updateRewardsStates` once in 249 blocks would be sufficient, as it would still result in rewards rounding down to zero.


### Root Cause

Lack of upscaling for tokens with less than 18 decimals for reward calculations.

### Attack Path

1. Attacker calls deposit/withdraw/notifyRewardAmount with any non-zero amount every block (or less often as long as the calculation will still round down to zero)

### Impact

High: stakers do not receive rewards in tokens with low decimals (e.g. USDC, USDT).

### PoC

1. SYMM total staked supply = 1_000_000e18
2. `notifyRewardAmount` is called with 1209.6 USDC
3. griefer calls `deposit/withdraw` 1 wei of SYMM each 249 blocks for 1 week
4. USDC rewards are stuck in the contract, instead of being distributed to stakers (but can be rescued by admin)

### Mitigation

Introduce 1e12 multiplier for reward calculation, and divide the accumulated rewards by 1e12 when they are being claimed.

---
### Example 3

**Auto Label:** Precision loss and logic flaws in reward calculations due to improper handling of fixed-point arithmetic, incorrect loop bounds, and inconsistent state updates, leading to systematic underpayment, denial of service, and wealth leakage.  

**Original Text Preview:**

## Severity: Low Risk

## Context
- `ReserveLogic.sol#L259`
- `ValidationLogic.sol#L330`
- `ValidationLogic.sol#L335`
- `ValidationLogic.sol#L342`
- `ValidationLogic.sol#L349`
- `MiniPoolLiquidationLogic.sol#L48`
- `MiniPoolLiquidationLogic.sol#L57`
- `MiniPoolReserveLogic.sol#L268`
- `MiniPoolValidationLogic.sol#L335`
- `MiniPoolValidationLogic.sol#L340`
- `MiniPoolValidationLogic.sol#L347`
- `MiniPoolValidationLogic.sol#L354`
- `RewardsDistributor.sol#L457`
- `RewardsDistributor.sol#L458`
- `RewardsDistributor.sol#L465`
- `RewardsDistributor.sol#L470`
- `RewardsDistributor6909.sol#L485`
- `RewardsDistributor6909.sol#L486`
- `RewardsDistributor6909.sol#L493`
- `RewardsDistributor6909.sol#L498`
- `AToken.sol#L267`
- `VariableDebtToken.sol#L278`
- `VariableDebtToken.sol#L305`
- `VariableDebtToken.sol#L307`
- `VariableDebtToken.sol#L308`

## Description
1. In `ReserveLogic` and `MiniPoolReserveLogic`, we have:
   ```solidity
   // Calculate the last principal variable debt.
   vars.previousVariableDebt = scaledVariableDebt.rayMul(previousVariableBorrowIndex);
   ```
   Here, principal means the actual debt principal plus the accrued interest.
   
2. In liquidation flows, the word principal is used in conjunction with reserve to mean the debt or borrow reserve.

3. In `RewardsDistributor` and `RewardsDistributor6909`, we have:
   ```solidity
   /**
    * @dev Calculates rewards based on principal balance and index difference.
    * @param principalUserBalance User's principal balance.
    * @param reserveIndex Current reserve index.
    * @param userIndex User's stored index.
    * @param decimals Number of decimals.
    * @return The calculated reward amount.
    */
   function _getRewards(
       uint256 principalUserBalance,
       uint256 reserveIndex,
       uint256 userIndex,
       uint8 decimals
   ) internal pure returns (uint256) {
       return (principalUserBalance * (reserveIndex - userIndex)) / 10 ** decimals;
   }
   ```
   Here, principal is supposed to represent the amount of shares, which is actually not the initial principal deposited or borrowed by the user.

4. In `AToken`, the word principal actually matches what we would consider as principal:
   ```solidity
   // The balance of the user: principal balance + interest generated by the principal.
   super.balanceOf(user).rayMul(
       _pool.getReserveNormalizedIncome(_underlyingAsset, RESERVE_TYPE)
   )
   ```

5. In `VariableDebtToken.sol#L277-L284` and `VariableDebtToken.sol#L304-L317`, the word principal refers to the shares and total shares in this token.

## Recommendation
In most cases, the comments or variable names need to be updated to reflect the actual value computed or stored. However, in case 3, the confusion can result in computing incorrect reward amounts for the user. The question comes into mind: should the user be awarded based on:
- The principal deposited or borrowed (the true meaning of the word) or...
- The shares of the `aToken` or `debtToken` the user holds.

If it's the first point, then the reward distribution flow needs to be updated, and the tokens would also need to store principal values along with the shares to be able to provide them to the incentive controllers.

## Cod3x
Acknowledged.

## Spearbit
Acknowledged.

---
