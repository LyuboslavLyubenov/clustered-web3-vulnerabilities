# Cluster -1043

**Rank:** #257  
**Count:** 34  

## Label
Failure to revoke or properly authorize privileged role assignments creates unauthorized access and disrupts protocol initialization, enabling attacks through persistent or misaligned permissions.

## Cluster Information
- **Total Findings:** 34

## Examples

### Example 1

**Auto Label:** Failure to revoke or properly authorize privileged role assignments creates unauthorized access and disrupts protocol initialization, enabling attacks through persistent or misaligned permissions.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `connect()` function in the `AdminStrategy` contract is externally callable and lacks access control, which allows any address to call it before the official initialization via `AdminOperationalTreasury`. Once called, it sets the `setUp.treasury` field, and any subsequent legitimate call (including the one from `init()`) will revert due to the `TreasuryAlreadySet()` check.

```solidity
    function connect() external override {
        StrategyStorage.SetUp storage setUp = StrategyStorage.layout().setUp;
        IOperationalTreasury treasury_ = IOperationalTreasury(msg.sender);
        if (address(setUp.treasury) != address(0)) revert TreasuryAlreadySet();
        setUp.treasury = treasury_;
    }
```

This creates a denial-of-service (DoS) vector where an attacker can call `connect()`, lock in a malicious treasury address, and cause `AdminOperationalTreasury` initialization to fail irreversibly. This breaks the protocol setup flow and can brick strategy contracts before they are properly initialized.

Note:
A similar issue exists in `addStrategy`, where invoking the strategy's `connect` function directly prevents admins from successfully adding a new strategy.

## Recommendations

Restrict `connect()` access to only a trusted contract (e.g., via `onlyRole` or checking `msg.sender`).

---
### Example 2

**Auto Label:** Failure to revoke or properly authorize privileged role assignments creates unauthorized access and disrupts protocol initialization, enabling attacks through persistent or misaligned permissions.  

**Original Text Preview:**

## Severity

Informational

## Description

In the updateTreasuryWallet() function, the contract sets a new treasuryWallet address
andgrants it the TREASURY_ROLE
However, the role from the previous treasury address is not revoked. As a result, any previously as
signed treasury address retains the TREASURY_ROLE

## Recommendation

Revoke the old treasuryWallet

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Lack of revocation or value handling enables unauthorized access or functional failures in access control mechanisms.  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
An IRM contract cannot be removed from the allowlist. Disallowing an allowed IRM ensures that future deployed markets cannot use that IRM anymore.

## Recommendation
Add a method to disallow IRMs so that new markets cannot be created using those IRMs. Markets with IRMs that are no longer allowed due to a bug can be deprecated. If a market is not deprecated and the IRM is not allowed, users will understand that the IRM is discouraged but does not have a bug.

## Fixes
- **Dahlia:** Fixed in commit `bb8ce8fa`.
- **Cantina Managed:** Fixed.

---
