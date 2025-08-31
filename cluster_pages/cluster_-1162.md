# Cluster -1162

**Rank:** #406  
**Count:** 11  

## Label
Lack of dependency version control leads to inconsistent, insecure, and unreproducible smart contract deployments, enabling supply chain risks and exploitation of unpatched vulnerabilities.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Lack of dependency version control leads to inconsistent, insecure, and unreproducible smart contract deployments, enabling supply chain risks and exploitation of unpatched vulnerabilities.  

**Original Text Preview:**

## Consistency of Seed Across Chains

The seed must remain consistent across all chains to ensure uniform signature usage. However, chains operate independently, and there is currently no mechanism to simultaneously update the seed across all chains when invalidating prior signatures, creating potential synchronization issues. Similarly, differences in `block.timestamp` across chains may require signature regeneration for certain chains due to varying expiry times.

## Remediation

Implement a mechanism to synchronize state across multiple blockchains to ensure consistency in signature generation.

## Patch

Fixed in 2d69c32.

---
### Example 2

**Auto Label:** Lack of dependency version control leads to inconsistent, insecure, and unreproducible smart contract deployments, enabling supply chain risks and exploitation of unpatched vulnerabilities.  

**Original Text Preview:**

**Severity**: Informational

**Status**: Resolved

**description**

A recent review of the fixed contracts found that multiple copies of the same contract were
left in the repository. This may confuse determining the current, active version of any given
contract.

**Recommendation:**

Remove outdated or redundant contract copies from the repository.

---
### Example 3

**Auto Label:** Use of unverified third-party dependencies introduces unknown vulnerabilities, compromising contract security through unvalidated external code.  

**Original Text Preview:**

**Description**

In the Factory contract, all except init and update of the contract are using third-party lib, which is out of scope.
programs/nft_factory

**Recommendation**

We need verification from the Onemind team that out-of-scope libs can be trusted.

**Re-audit comment**

Verified.

From client:
The only lib contracts are using is the common library for token and metaplex helpers - included in the git repository under 'libraries/common and attached to both contracts in Cargo.toml.

Post-audit.
The 3rd-party code has been verified by the client.

---
