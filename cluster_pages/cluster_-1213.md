# Cluster -1213

**Rank:** #326  
**Count:** 21  

## Label
Failure to validate critical invariants across control flow paths enables attackers to induce crashes, disrupt consensus, or manipulate data—leading to denial-of-service or loss of integrity through unverified or malformed inputs.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Lack of input validation enables attackers to manipulate critical parameters or submit forged data, leading to consensus errors, reward manipulation, or denial-of-service.  

**Original Text Preview:**

## Reports

**Severity:** Medium Risk  
**Context:** (No context files were provided by the reviewer)  
**Description:** This issue was found during the fixes review period in commit hash `e2bac5c`. As part of mitigating the issues found during the review, the Kinetiq team introduced a new contract named `ValidatorSanityChecker` which is used for validity checks on oracle reports. The contract has one main function named `checkValidatorSanity` which, among other checks, validates that given values (such as `avgUptimeScore`, `avgSpeedScore`, etc.) are inside bounds based on storage variables that can be set. The issue here is that these storage variables can be set by anyone, which can be exploited by attackers to cause the rejection of oracle reports. The full list of unprotected functions:

1. `setSlashingTolerance()`
2. `setRewardsTolerance()`
3. `setScoreTolerance()`
4. `setMaxScoreBound()`

**Recommendation:** Consider restricting access to these functions to privileged accounts only.  
**Kinetiq:** Fixed in commit `b739069`.  
**Cantina Managed:** Fix verified.

---
### Example 2

**Auto Label:** Lack of input validation enables attackers to manipulate critical parameters or submit forged data, leading to consensus errors, reward manipulation, or denial-of-service.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Data Validation

## Description
Due to missing validation on the proposer address included in the dispute data, the wrong user can be charged for a resolved dispute instead of the actual proposer of the response.

When a user submits a dispute for a particular response, the dispute will contain the disputer address, the address of the response proposer, the responseId, and the requestId:

```solidity
struct Dispute {
    address disputer;
    address proposer;
    bytes32 responseId;
    bytes32 requestId;
}
```

**Figure 12.1**: The Dispute struct in IOracle.sol

While the proposer address is intended to be the address of the user that proposed the response, this is never validated in the `disputeResponse` function of the Oracle contract. If the disputer were to win the dispute, the address that will be charged the bond amount will be the proposer defined in the dispute struct:

```solidity
} else if (_status == IOracle.DisputeStatus.Won) {
    // Disputer won, we pay the disputer and release their bond
    _params.accountingExtension.pay({
        _requestId: _dispute.requestId,
        _payer: _dispute.proposer,
        _receiver: _dispute.disputer,
        _token: _params.bondToken,
        _amount: _params.bondSize
    });
    [...]
}
```

**Figure 12.2**: A snippet from the `onDisputeStatusChange` function in BondedDisputeModule.sol

Due to this, a user is able to provide an arbitrary proposer address for a valid dispute in order to grief other users, causing them to lose their bond amount.

## Exploit Scenario
Alice creates a request through the Oracle contract. Eve submits an incorrect response and disputes her own response, setting the response proposer address to Alice’s address. Once the dispute is won, Alice will be charged the dispute bond instead of Eve; this will prevent her request from being finalized due to an insufficient balance.

## Recommendations
- **Short term**: Ensure that the proposer address of a dispute is equal to the proposer of the response that the dispute is targeting.
- **Long term**: Improve your testing suite by testing for common adversarial situations such as providing an incorrect combination of inputs to the contract functions.

---
### Example 3

**Auto Label:** Lack of input validation enables attackers to manipulate critical parameters or submit forged data, leading to consensus errors, reward manipulation, or denial-of-service.  

**Original Text Preview:**

##### Description
If `churnValidatorsPerDayLimit` is small and the frequency of the report is high, then there is a chance that `churnLimit` will be rounded down to zero: https://github.com/lidofinance/core/blob/efeff81c18f85451ebf98e8fd8bb78b8eb0095f6/contracts/0.8.9/sanity_checks/OracleReportSanityChecker.sol#L780.

##### Recommendation
We recommend adding a minimum value limit for `churnLimit`.

---
