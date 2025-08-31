# Cluster -1120

**Rank:** #421  
**Count:** 10  

## Label
Inconsistent state tracking and improper validation of position status across loan cycles lead to incorrect calculations, fund locking, and denial of exits, enabling malicious actors to manipulate or withhold funds and disrupt liquidity.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** Inconsistent state tracking and improper validation of position status across loan cycles lead to incorrect calculations, fund locking, and denial of exits, enabling malicious actors to manipulate or withhold funds and disrupt liquidity.  

**Original Text Preview:**

## Context
MstWrapper.sol#L186-L188

## Description
There is a `tick()` function to fetch the encoded deposit tick. For convenience, consider adding another function with decoded information (price limit, loan duration, APR).

## Recommendation
```solidity
function decodedTick() external view returns (uint128, uint256, uint256, uint64, uint64) {
    (uint256 limit, uint256 durationIndex, uint256 rateIndex,) = _tick.decode();
    (uint64[] memory result) = _pool.durations();
    uint64 duration = result[durationIndex];
    result = _pool.rates();
    uint64 rate = result[rateIndex]; // APR = annual rate, perhaps it should * 365 * 86_400
    return (uint128(limit), durationIndex, rateIndex, duration, rate);
}
```

## Metastreet
Resolved in commit `f45f5ad`. We opted to split it out into separate getters: `limit()`, `duration()`, `rate()`.

## Cantina
The issue has been fixed in the aforementioned commit.

---
### Example 2

**Auto Label:** Inconsistent state tracking and improper validation of position status across loan cycles lead to incorrect calculations, fund locking, and denial of exits, enabling malicious actors to manipulate or withhold funds and disrupt liquidity.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

### Target: TickLogic.sol

## Description
If a position has been detached during a previous loan cycle and is borrowed in a subsequent loan cycle, that position cannot be exited. The protocol allows positions that can be fully matched to exit the loan, withdrawing the full amount of their deposit and the accumulated accruals. However, a position that has been detached during a previous lending cycle will not be able to exit; instead, the function will revert with an arithmetic underflow:

```solidity
634    tick.withdrawnAmounts.toBeAdjusted -= endOfLoanPositionValue;
```

**Figure 21.1**: registerExit function in TickLogic.sol

This is because the `prepareTickForNextLoan` function in the TickLogic contract is called on each repayment, which in turn sets the `tick.withdrawnAmounts` to zero:

```solidity
104    delete tick.withdrawnAmounts;
```

**Figure 21.2**: prepareTickForNextLoan function in TickLogic.sol

Due to this error, the lender will be prevented from exiting their position and will have to wait for the loan to be repaid to withdraw their assets.

## Exploit Scenario
A Revolving Credit Line is created for Alice with a maximum borrowable amount of 100 ether and a duration of 10 weeks.

1. Bob deposits 20 ether into the pool at a 10% rate, and Alice borrows 10 ether.
2. Bob assumes Alice will not borrow any more assets and detaches his position so he can utilize the assets elsewhere. He receives 10 ether from the position.
3. Alice repays the loan and again borrows 10 ether.
4. Charlie deposits 10 ether into the pool at a 10% rate.
5. Bob decides to exit his position. However, when he executes the `exit` function, it reverts with an arithmetic underflow.

## Recommendations
Short term, consider revising how detached positions are considered in the system during multiple loan cycles. Long term, improve unit test coverage and create a list of system properties that can be tested with smart contract fuzzing. For example, this issue could have been discovered by implementing a property test that checks that calling `exit` does not revert if all the preconditions are met.

---
### Example 3

**Auto Label:** Inconsistent state tracking and improper validation of position status across loan cycles lead to incorrect calculations, fund locking, and denial of exits, enabling malicious actors to manipulate or withhold funds and disrupt liquidity.  

**Original Text Preview:**

## Vulnerability Report

## Difficulty
**Medium**

## Type
**Data Validation**

## Target
**TickLogic.sol**

## Description
Lenders who detach their position earn less interest in subsequent loan cycles due to an incorrect calculation in `getAdjustedAmount`. The protocol allows users whose position has not been fully borrowed to detach their position and withdraw their leftover unborrowed assets. To avoid storing the state of each individual position, the protocol derives the position’s borrowed and unborrowed amounts. This is done via the `adjustedAmount` and the total available and borrowed amount of that interest rate, as shown below:

```solidity
uint256 adjustedAmount = getAdjustedAmount(tick, position, referenceLoan);
unborrowedAmount =
if (position.epochId <= tick.loanStartEpochId) {
    tick.baseEpochsAmounts.available.mul(adjustedAmount).div(tick.baseEpochsAmounts.adjustedDeposits);
}
return (unborrowedAmount, borrowedAmount);
borrowedAmount = tick.baseEpochsAmounts.borrowed.mul(adjustedAmount).div(tick.baseEpochsAmounts.adjustedDeposits);
```

*Figure 13.1: The `getPositionRepartition` function in `TickLogic.sol`*

The `getAdjustedAmount` function has a special case for calculating detached positions using the position information and the `endOfLoanYieldFactor` of the loan in which the position was detached:

```solidity
if (position.withdrawLoanId > 0) {
    uint256 lateRepayFees = accrualsFor(
        position.withdrawn.borrowed,
        referenceLoan.lateRepayTimeDelta,
        referenceLoan.lateRepayFeeRate
    ).mul(referenceLoan.repaymentFeesRate);
    uint256 protocolFees = (position.withdrawn.expectedAccruals + lateRepayFees);
}
adjustedAmount = (position.withdrawn.borrowed + position.withdrawn.expectedAccruals + lateRepayFees - protocolFees).div(tick.endOfLoanYieldFactors[position.withdrawLoanId]);
```

*Figure 13.2: The `getAdjustedAmount` function in `TickLogic.sol`*

However, this calculation returns an incorrect value, causing the protocol to consider the position’s borrowed amount to be less than it actually is. This will result in the lender earning less interest than they should.

## Exploit Scenario
1. A Revolving Credit Line is created for Alice with a maximum borrowable amount of 10 ether and a duration of 10 weeks.
2. Bob deposits 15 ether at a 10% rate.
3. Alice borrows 10 ether from the pool.
4. Bob detaches his position, withdrawing the unborrowed 5 ether to his account.
5. Alice repays the loan on time and takes another loan of 10 ether.
6. Although Alice took a 10 ether loan, the protocol considers Bob’s position to be borrowed for 9.9998 ether.
7. Bob earns less accruals than he should for the duration of the loan and all subsequent loans.

## Recommendations
- **Short Term:** Investigate the ways in which detached position values are calculated and fees are subtracted in the system to ensure position values are derived correctly.
- **Long Term:** Consider redesigning how fees are charged throughout the system so that derived values are not incorrectly impacted. Improve unit test coverage and create a list of system properties that can be tested with smart contract fuzzing. For example, this issue could have been discovered by implementing a property test that checks that the total borrowed amount should be equal to the sum of the borrowed amounts of all positions.

---
