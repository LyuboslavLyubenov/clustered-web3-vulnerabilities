# Cluster -1232

**Rank:** #205  
**Count:** 54  

## Label
Failure to update external library dependencies leads to exposure to known vulnerabilities, including reentrancy, access control flaws, and arithmetic errors, due to outdated or unpatched code in trusted components.

## Cluster Information
- **Total Findings:** 54

## Examples

### Example 1

**Auto Label:** Failure to update external library dependencies leads to exposure to known vulnerabilities, including reentrancy, access control flaws, and arithmetic errors, due to outdated or unpatched code in trusted components.  

**Original Text Preview:**

##### Description
The local versions of the `Ownable` and `Context` contracts in the `StringKeyValueStore` contract is unnecessary as can be seamlessly imported from the OpenZeppelin library.

##### Recommendation
It is advised to remove the local implementations of the `Ownable` and `Context` contracts and instead import them directly from the OpenZeppelin library.

---
### Example 2

**Auto Label:** Failure to update external library dependencies leads to exposure to known vulnerabilities, including reentrancy, access control flaws, and arithmetic errors, due to outdated or unpatched code in trusted components.  

**Original Text Preview:**

##### Description

Throughout the codebase, several OpenZeppelin contracts implementations are inherited and used. However, the version of these contracts are outdated (`4.9.2`) and may contain vulnerabilities that may have been fixed in newer versions (latest stable version is `5.1.0`).

  

For more reference about OpenZeppelin contracts versions and their vulnerabilities, see <https://security.snyk.io/package/npm/@openzeppelin%2Fcontracts>

##### BVSS

[AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N (0.0)](/bvss?q=AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N)

##### Recommendation

Consider updating all OpenZeppelin contracts to the latest versions to benefit from the latest security patches and improvements.

##### Remediation

**ACKNOWLEDGED:** The **SideKick team** made a business decision to acknowledge this finding and not alter the contracts.

##### References

[livegame-labs/sidekick-contracts/contracts/Escrow.sol#L4-L7](https://github.com/livegame-labs/sidekick-contracts/blob/c116e404a03cb9240a4e8f48773bdfa81e2697e7/contracts/Escrow.sol#L4-L7)

[livegame-labs/sidekick-contracts/contracts/usdt.sol#L4-L5](https://github.com/livegame-labs/sidekick-contracts/blob/c116e404a03cb9240a4e8f48773bdfa81e2697e7/contracts/usdt.sol#L4-L5)

[livegame-labs/sidekick-contracts/package.json#L16](https://github.com/livegame-labs/sidekick-contracts/blob/c116e404a03cb9240a4e8f48773bdfa81e2697e7/package.json#L16)

---
### Example 3

**Auto Label:** Failure to update external library dependencies leads to exposure to known vulnerabilities, including reentrancy, access control flaws, and arithmetic errors, due to outdated or unpatched code in trusted components.  

**Original Text Preview:**

##### Description

The review identified that the project directly includes copies of OpenZeppelin modules within its repository (e.g., some files under `contracts/erc20/`, `contracts/utils/` and `contracts/library/`) and applies modifications to these files. Notably, the project contains:

* Two separate copies of the `Address.sol` file (one under `contracts/library/` and another under `contracts/utils/`), each marked with "OpenZeppelin Contracts (last updated v4.8.0) (utils/Address.sol)".
* A custom interface `IDualCore.sol` that is derived from the OpenZeppelin `IERC20.sol` interface with modifications.

  

Additionally, contracts like `CoreVault` and `MergeMarketplaceStrategy` use a custom `OnwnableWithPausable` module, which is a combination of `Ownable2Step` and `Pausable`. However, `Marketplace` is also using `Ownable2Step`, but implementing the pausable logic inside. Furthermore, notice the `OnwnableWithPausable` module has a typo and should be `OwnableWithPausable` instead.

  

Finally, a further issue was observed with the `Math` library. In version `v4.8.0` of the included OpenZeppelin contracts, the function `log256` is documented incorrectly. The outdated documentation comment reads:

```
/**
 * @dev Return the log in base 10, following the selected rounding direction, of a positive value.
 * Returns 0 if given 0.
 */
```

Whereas the correct documentation (as updated in later commits, see commit [3a3c87b1a676f277c17a4601de56ddfc432d427d](https://github.com/OpenZeppelin/openzeppelin-contracts/commit/3a3c87b1a676f277c17a4601de56ddfc432d427d)) should specify that the function returns the log in base 256. Retaining an outdated and incorrect version not only causes confusion for auditors and developers but also raises the risk of misinterpretation or misimplementation of mathematical operations.

  

This approach poses several risks:

* **Security and Maintenance Drift:** OpenZeppelin contracts are thoroughly audited and maintained. By copying and modifying these files, the project bypasses the continuous improvements and security patches provided by the official library. This may inadvertently introduce vulnerabilities or inconsistencies with the audited versions.
* **Upgrade and Compatibility Issues:** Custom modifications mean that future updates or bug fixes released by OpenZeppelin will not be automatically integrated. This increases the risk of the project using outdated or insecure implementations.
* **Code Duplication and Divergence:** Maintaining multiple copies of similar code (as seen with `Address.sol`) can lead to inconsistencies and maintenance challenges. This duplication increases the attack surface and complicates the auditing and review process.

##### BVSS

[AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N (1.7)](/bvss?q=AO:A/AC:L/AX:M/R:N/S:U/C:N/A:N/I:L/D:N/Y:N)

##### Recommendation

It is recommended the following:

* **Utilize Dependency Management:** Instead of copying OpenZeppelin contracts directly into the repository, consider using a package manager (such as npm or yarn) to include the OpenZeppelin libraries as external dependencies. This practice ensures access to the latest security updates and improvements.
* **Minimize Unnecessary Modifications:** When modifications are necessary, it is recommended to fork the OpenZeppelin repository and document the changes along with their justifications.
* **Eliminate Redundant Code:** Remove duplicate copies of modules (for instance, consolidate the `Address.sol` file into a single location) to prevent inconsistencies and simplify future updates.

  

By adhering to these recommendations, the project can benefit from the robust security and maintenance practices provided by the official OpenZeppelin libraries while minimizing the risks associated with direct modifications and code duplication.

##### Remediation

**PENDING:** The **B14G team** indicated that they will be addressing this finding in the future.

---
