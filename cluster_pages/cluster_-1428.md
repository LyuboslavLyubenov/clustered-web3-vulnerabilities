# Cluster -1428

**Rank:** #360  
**Count:** 17  

## Label
Insufficient state validation and flawed access control allow unauthorized full liquidity extraction during active periods, enabling attackers to drain funds through bypassed balance and period checks.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Inconsistent state validation leads to incorrect fund returns or unauthorized token gains via dynamic parameter misuse or unverified liquidity operations.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-06-superfluid-locker-system-judging/issues/96 

This issue has been acknowledged by the team but won't be fixed at this time.

## Found by 
0x73696d616f, zxriptor

### Summary

The `FluidEPProgramManager.cancelProgram()` function incorrectly assumes that governance parameters (minimum deposit and liquidation period) remain constant throughout a program's lifecycle. When these parameters change after program creation, the calculated `initialDeposit` for treasury return will be incorrect, potentially causing transaction reverts or wrong accounting.

### Root Cause

The vulnerability lies in the [`cancelProgram()`](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/fluid/packages/contracts/src/FluidEPProgramManager.sol#L264-L268) function, where it recalculates the initial deposit amount using current governance parameters instead of the original values from when the program was funded:

```solidity
uint256 buffer =
    program.token.getBufferAmountByFlowRate(programDetails.fundingFlowRate + programDetails.subsidyFlowRate);
uint256 initialDeposit =
    buffer + uint96(programDetails.fundingFlowRate + programDetails.subsidyFlowRate) * EARLY_PROGRAM_END;
```

The buffer calculation depends on two governance-controlled parameters that can be changed at any time:

1. Minimum Deposit: retrieved via [`SUPERTOKEN_MINIMUM_DEPOSIT_KEY`](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/agreements/ConstantFlowAgreementV1.sol#L179-L181)
2. Liquidation Period: retrieved via [`CFAV1_PPP_CONFIG_KEY`](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/agreements/ConstantFlowAgreementV1.sol#L182-L183) and [decoded](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/agreements/ConstantFlowAgreementV1.sol#L184)

The buffer amount flows through this call chain:

```text
cancelProgram()
├── program.token.getBufferAmountByFlowRate(totalFlowRate)
    ├── SuperTokenV1Library.getBufferAmountByFlowRate()
        ├── cfa.getDepositRequiredForFlowRate(token, flowRate)
            ├── gov.getConfigAsUint256(host, token, SUPERTOKEN_MINIMUM_DEPOSIT_KEY)
            ├── gov.getConfigAsUint256(host, token, CFAV1_PPP_CONFIG_KEY)
            ├── SuperfluidGovernanceConfigs.decodePPPConfig(pppConfig)
            └── _getDepositRequiredForFlowRatePure(minimumDeposit, liquidationPeriod, flowRate)
                └── max(minimumDeposit, _clipDepositNumberRoundingUp(flowRate * liquidationPeriod))
```

### Internal Pre-conditions

1. The program needs to be cancelled

### External Pre-conditions

1. Superfluid governance changes either the minimum deposit or liquidation period parameters after program funding via:
   - [`setSuperTokenMinimumDeposit()`](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/gov/SuperfluidGovernanceBase.sol#L395-L404) for minimum deposit changes
   - [`setPPPConfig()`](https://github.com/sherlock-audit/2025-06-superfluid-locker-system/blob/main/protocol-monorepo/packages/ethereum-contracts/contracts/gov/SuperfluidGovernanceBase.sol#L342-L364) for liquidation period changes

### Attack Path

1. Program is created and funded with `initialDeposit` calculated using current governance parameters
2. Superfluid governance changes minimum deposit or liquidation period via:
   ```text
   npx truffle exec ops-scripts/gov-set-token-min-deposit.js : {TOKEN_ADDRESS} {MINIMUM_DEPOSIT}
   npx truffle exec ops-scripts/gov-set-3Ps-config.js : {TOKEN_ADDRESS} {LIQUIDATION_PERIOD} {PATRICIAN_PERIOD}
   ```
3. When `cancelProgram()` is called, it recalculates `initialDeposit` using new parameters
4. If parameters increased: transaction attempts to return more funds
5. If parameters decreased: excess funds remain in contract rather than sent back to treasury

### Impact

When governance parameters increase after program creation, `cancelProgram()` calculates higher buffer requirement than originally deposited, returning more funds to treasury than expected which may cause transaction revert due to insufficient fund balance. This cause inability to cancel the program.

When governance parameters decrease after program creation, `cancelProgram()` calculates lower buffer requirement than originally deposited, causing excess funds to remain in the contract while treasury receives less than the original deposit amount.

### PoC

No response

### Mitigation

Store the original `initialDeposit` amount in the program struct and use it during cancellation:

```solidity
struct EPProgram {
    // ... existing fields ...
    uint256 initialDeposit;
}

// In fundProgram():
programs[programId].initialDeposit = initialDeposit;

// In cancelProgram():
token.transfer(TREASURY, programs[programId].initialDeposit);
```

This ensures the exact deposited amount is returned regardless of governance parameter changes.

---
### Example 2

**Auto Label:** Inconsistent state validation leads to incorrect fund returns or unauthorized token gains via dynamic parameter misuse or unverified liquidity operations.  

**Original Text Preview:**

## Difficulty: Low

## Type: Data Validation

## Description

Users can avoid the _CONVERT_FACTOR applied to wrap and unwrap operations by providing unbalanced liquidity to the buffer and then immediately removing the liquidity. During wrap and unwrap operations, the _CONVERT_FACTOR is either added to the amount of tokens the user needs to deposit or removed from the amount of tokens the user will get out of this operation. This is done as a safety measure to prevent the buffer from being drained due to rounding issues.

### Function

```solidity
function _wrapWithBuffer(
    SwapKind kind,
    IERC20 underlyingToken,
    IERC4626 wrappedToken,
    uint256 amountGiven
) private returns (uint256 amountInUnderlying, uint256 amountOutWrapped) {
    // ...
    if (kind == SwapKind.EXACT_IN) {
        if (isQueryContext) {
            return (amountGiven, wrappedToken.previewDeposit(amountGiven));
        }
        // EXACT_IN wrap, so AmountGiven is underlying amount. If the buffer has
        // enough liquidity to handle the wrap, it'll send amountOutWrapped to the user
        // without calling the wrapper protocol, so it can't check
        // if the "convert" function has rounding errors or is oversimplified. To
        // make sure the buffer is not drained we remove a convert factor that decreases the
        // amount of wrapped tokens out, protecting the buffer balance.
        (amountInUnderlying, amountOutWrapped) = (
            amountGiven,
            wrappedToken.convertToShares(amountGiven) - _CONVERT_FACTOR
        );
    }
}
```

Figure 14.1: The _CONVERT_FACTOR is removed from the amountOutWrapped in _wrapWithBuffer (Vault.sol#L1148–L1171)

However, the `addLiquidityToBuffer` function of the `VaultAdmin` contract allows users to freely provide unbalanced liquidity. As a result, a user can provide liquidity with only the wrapped or underlying token and then remove the liquidity to obtain a proportional amount of the other token, essentially mimicking a wrap/unwrap operation. This allows them to avoid the _CONVERT_FACTOR.

## Exploit Scenario

Eve deposits unbalanced liquidity to a buffer and immediately withdraws it in order to emulate a wrap/unwrap action. She obtains more tokens than she would have if she had used the wrap or unwrap action.

## Recommendations

- **Short term:** Either ensure that liquidity can only be added proportionally to a buffer, or treat the unbalanced portion of the liquidity addition as a wrap/unwrap operation and "charge" the _CONVERT_FACTOR.
  
- **Long term:** Improve the testing suite by adding additional unit tests for providing and removing buffer liquidity. Test the wrap and unwrap operations against the `addLiquidityToBuffer` and `removeLiquidityFromBuffer` functions, ensuring that adding unbalanced liquidity and then removing does not give users an advantage.

---
### Example 3

**Auto Label:** Inaccurate state updates due to improper edge case handling, leading to fund locking, incorrect matching, or erroneous token transfers and enabling denial-of-service or financial loss.  

**Original Text Preview:**

**Details**

    function killBorrowerMatch(address lender) external returns (uint256 lenderAmount, uint256 borrowerReturn) {
        MatchOrder[] storage matches = _userMatchedOrders[lender];

        uint256 len = matches.length;
        for (uint256 i = 0; i != len; ++i) {
            if (matches[i].borrower == msg.sender) {
                lenderAmount = uint256(matches[i].lCollateral);
                borrowerReturn = uint256(matches[i].bCollateral);
                // Zero out the match order to preserve the array's order
                matches[i] = MatchOrder({lCollateral: 0, bCollateral: 0, borrower: address(0)});
                // Decrement the matched depth and open move the lenders collateral to an open order.
                _matchedDepth -= lenderAmount;
                _openOrder(lender, lenderAmount);
                break;
            }
        }

When a borrower kills a match, all values are zeroed out inside the MatchOrder struct and it is kept in the lender's matches array. This leads to a problem later when trying to match the lenders orders.

[BloomPool.sol#L382-L415](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L382-L415)

    uint256 length = matches.length;
    for (uint256 i = length; i != 0; --i) {
        uint256 index = i - 1;


        if (remainingAmount != 0) {
            (uint256 lenderFunds, uint256 borrowerFunds) =
                _calculateRemovalAmounts(remainingAmount, matches[index].lCollateral, matches[index].bCollateral);
            uint256 amountToRemove = lenderFunds + borrowerFunds;

            if (amountToRemove == 0) break; <- @audit matching loop breaks when collateral in match is 0
            remainingAmount -= amountToRemove;

            _borrowerAmounts[matches[index].borrower][id] += borrowerFunds;
            borrowerAmountConverted += borrowerFunds;

            emit MatchOrderConverted(id, account, matches[index].borrower, lenderFunds, borrowerFunds);

            if (lenderFunds == matches[index].lCollateral) {
                matches.pop();
            } else {
                matches[index].lCollateral -= uint128(lenderFunds);
            }
        } else {
            break;
        }
    }

We see above that when the collateral is 0 as it is with a canceled match, the matching loop breaks prematurely. Given this is FILO matching, all deposits made prior the cancelled order will be impossible to match. Those borrowers will need to all manually cancel their orders and resubmit. This presents a high potential for griefing as a malicious borrower can DOS lenders and borrowers alike.

**Lines of Code**

[BloomPool.sol#L377-L416](https://github.com/Blueberryfi/bloom-v2/blob/87a60380331cc914be41ad57691f08b532a4d6fb/src/BloomPool.sol#L377-L416)

**Recommendation**

BloomPool#\_convertMatchOrders should pop empty matches and continue rather than breaking.

**Remediation**

Fixed as recommended in bloom-v2 [PR#15](https://github.com/Blueberryfi/bloom-v2/pull/15)

---
