# Cluster -1073

**Rank:** #308  
**Count:** 23  

## Label
Lack of proper authorization and control over critical operations leads to unauthorized asset transfers or misallocation, enabling malicious actors to exploit system gaps and compromise fund security.

## Cluster Information
- **Total Findings:** 23

## Examples

### Example 1

**Auto Label:** Improper fund flow and authorization lead to permanent loss or unauthorized extraction of assets through flawed transfer mechanisms and missing state controls.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `AdminOperationalTreasury.withdrawFee()` function uses `safeTransferFrom()` instead of `safeTransfer()` when withdrawing protocol fees. This is incorrect because:

1. `safeTransferFrom()` requires the spender to have an allowance from the owner.
2. The contract is trying to transfer from itself (`address(this)`) to the recipient.
3. The contract has not approved itself to spend its own tokens.

```solidity
function withdrawFee(address token_, address recipient_) external {
    OperationalTreasuryStorage.Layout storage strg = OperationalTreasuryStorage.layout();

    if (msg.sender != strg.setUp.base.feesReceiver) revert("FeesReceiverOnly");
    uint256 amount = strg.ledger.collectedFees[token_];
    strg.ledger.collectedFees[token_] = 0;

@>  IERC20Metadata(token_).safeTransferFrom(address(this), recipient_, amount);
    emit ProtocolFeeWithdrawn(token_, recipient_, amount);
}
```

For normal `ERC20`, `transferFrom` always checks the allowance of the spender, even when `from` is the same as `msg.sender`. As a result, the fee transfer will fail due to insufficient allowance.

## Recommendation

Replace `safeTransferFrom()` with `safeTransfer()`, since the contract is transferring tokens it owns.

---
### Example 2

**Auto Label:** Improper fund flow and authorization lead to permanent loss or unauthorized extraction of assets through flawed transfer mechanisms and missing state controls.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The `transferFee` function in the `TheCounter` contract incorrectly uses `safeTransferFrom` instead of `safeTransfer` when transferring collected fees from the contract to the admin. This implementation error will cause the function to revert due to missing allowances.

As a result, the admin is unable to collect protocol fees.

```solidity
    function transferFee() external onlyRole(DEFAULT_ADMIN_ROLE) {
        ...
        token.safeTransferFrom(address(this), msg.sender, feeToTransfer); // @audit should use safeTransfer
        ...
    }
```

## Recommendations

```diff
    function transferFee() external onlyRole(DEFAULT_ADMIN_ROLE) {
        ...
-        token.safeTransferFrom(address(this), msg.sender, feeToTransfer);
+        token.safeTransfer(msg.sender, feeToTransfer);
        ...
    }

```

---
### Example 3

**Auto Label:** Unauthorized access or lack of proper authorization enables attackers to manipulate or withdraw funds, bypass validation, or exploit logic flaws for unauthorized claims or fund theft.  

**Original Text Preview:**

## Difficulty: Low

## Type: Access Controls

### File Location
`prophet-modules/solidity/contracts/extensions/BondEscalationAccounting.sol`

### Description
When settling a bond escalation, the BondEscalation contract does not call the AccountingExtension contract’s `onSettleBondEscalation` function unless at least one user has pledged against the dispute. If no users pledge against the dispute, any user who has pledged for the dispute will not be able to recover their pledge.

**Figure 10.1** shows the logic handling dispute settlement in the case that the dispute escalation process does not result in a tie.

```solidity
if (_pledgesAgainstDispute > 0) {
    uint256 _amountToPay = _won
        ? _params.bondSize + FixedPointMathLib.mulDivDown(_pledgesAgainstDispute, _params.bondSize, _pledgesForDispute)
        : _params.bondSize + FixedPointMathLib.mulDivDown(_pledgesForDispute, _params.bondSize, _pledgesAgainstDispute);
    
    _params.accountingExtension.onSettleBondEscalation({
        _requestId: _dispute.requestId,
        _disputeId: _escalation.disputeId,
        _token: _params.bondToken,
        _amountPerPledger: _amountToPay,
        _winningPledgersLength: _won ? _pledgesForDispute : _pledgesAgainstDispute
    });
}
```

**Figure 10.1**: An excerpt of the `onDisputeStatusChange` function of the BondEscalationModule contract

### Exploit Scenario
Alice submits an incorrect response to a request. Bob disputes the response by posting a bond, then further escalates the dispute by submitting a pledge. Alice realizes that her response was wrong and does not pledge against the dispute. After the bond escalation period expires, the BondEscalationModule contract releases Bob’s bond but does not call the accounting extension’s `onSettleBondEscalation`. Bob is then unable to successfully call `claimEscalationReward` to recover his pledge.

### Recommendations
- **Short term**: Change the conditional to execute the `onSettleBondEscalation` logic when either of `_pledgesAgainstDispute` or `_pledgesForDispute` is nonzero.
- **Long term**: Add unit tests exercising all edge cases and conditionals in the contract logic.

---
