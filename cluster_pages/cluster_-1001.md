# Cluster -1001

**Rank:** #447  
**Count:** 7  

## Label
Lack of SPDX license identifiers compromises legal clarity and trust by creating ambiguity in licensing terms, exposing projects to intellectual property risks and undermining auditability and compliance in smart contracts.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Inconsistent or invalid SPDX license expressions across smart contracts lead to legal ambiguities, tooling errors, and license incompatibilities, increasing compliance risks and undermining interoperability.  

**Original Text Preview:**

##### Description

The project contains contracts with varying SPDX license identifiers:

* Some contracts specify `// SPDX-License-Identifier: Apache-2.0`.
* Others specify `// SPDX-License-Identifier: MIT`.
* Some use `// SPDX-License-Identifier: UNLICENSED`.

  

This inconsistency can result in potential license incompatibilities or legal ambiguities regarding the use, distribution, and modification of the codebase. Specifically:

* The **MIT License** is permissive, allowing free use, modification, and redistribution with attribution.
* The **Apache 2.0 License** is also permissive but includes explicit patent grants and requires a more detailed notice, including a copy of the license and a notice of any modifications.
* The **UNLICENSED** identifier indicates that the code is not licensed under any open-source terms, which may default to "all rights reserved" in some jurisdictions, limiting its usability.

  

While the MIT and Apache 2.0 licenses are generally compatible with each other, mixing them without clear documentation can create confusion. The presence of `UNLICENSED` files further complicates the legal standing of the codebase.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

To mitigate potential legal risks:

1. **Standardize Licensing:**

   * Choose a single open-source license that aligns with the project's goals and ensure all contracts uniformly declare this license.
   * If multiple licenses are necessary, clearly document which parts of the codebase are under which licenses and ensure they are compatible.
2. **Avoid** `UNLICENSED` Identifier:

   * Refrain from using the `UNLICENSED` identifier unless the intention is to restrict usage. If the code is meant to be open-source, apply an appropriate open-source license.
3. **Review License Compatibility:**

   * If retaining multiple licenses, consult with legal counsel to ensure compatibility and compliance with all applicable licensing terms.

  

By standardizing the licensing across the codebase and ensuring clarity in licensing terms, the project can prevent legal ambiguities and facilitate easier use, distribution, and contribution.

##### Remediation

**PENDING:** The **B14G team** indicated that they will be addressing this finding in the future.

---
### Example 2

**Auto Label:** Inconsistent or invalid SPDX license expressions across smart contracts lead to legal ambiguities, tooling errors, and license incompatibilities, increasing compliance risks and undermining interoperability.  

**Original Text Preview:**

##### Description

The project contains contracts with varying SPDX license identifiers:

* Some contracts specify `// SPDX-License-Identifier: Apache-2.0`.
* Others specify `// SPDX-License-Identifier: MIT`.
* Some use `// SPDX-License-Identifier: UNLICENSED`.

  

This inconsistency can result in potential license incompatibilities or legal ambiguities regarding the use, distribution, and modification of the codebase. Specifically:

* The **MIT License** is permissive, allowing free use, modification, and redistribution with attribution.
* The **Apache 2.0 License** is also permissive but includes explicit patent grants and requires a more detailed notice, including a copy of the license and a notice of any modifications.
* The **UNLICENSED** identifier indicates that the code is not licensed under any open-source terms, which may default to "all rights reserved" in some jurisdictions, limiting its usability.

  

While the MIT and Apache 2.0 licenses are generally compatible with each other, mixing them without clear documentation can create confusion. The presence of `UNLICENSED` files further complicates the legal standing of the codebase.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

To mitigate potential legal risks:

1. **Standardize Licensing:**

   * Choose a single open-source license that aligns with the project's goals and ensure all contracts uniformly declare this license.
   * If multiple licenses are necessary, clearly document which parts of the codebase are under which licenses and ensure they are compatible.
2. **Avoid** `UNLICENSED` Identifier:

   * Refrain from using the `UNLICENSED` identifier unless the intention is to restrict usage. If the code is meant to be open-source, apply an appropriate open-source license.
3. **Review License Compatibility:**

   * If retaining multiple licenses, consult with legal counsel to ensure compatibility and compliance with all applicable licensing terms.

  

By standardizing the licensing across the codebase and ensuring clarity in licensing terms, the project can prevent legal ambiguities and facilitate easier use, distribution, and contribution.

##### Remediation

**PENDING:** The **B14G team** indicated that they will be addressing this finding in the future.

---
### Example 3

**Auto Label:** Inconsistent or invalid SPDX license expressions across smart contracts lead to legal ambiguities, tooling errors, and license incompatibilities, increasing compliance risks and undermining interoperability.  

**Original Text Preview:**

##### Description

The inconsistent use of open-source licenses across the files in the protocol was identified. The **VaultController** and **IpRoyaltyVault** contracts specifying the **MIT** license, the **IPAccountStorageOps** is unlicensed and others using the more restrictive **BUSL-1.1** license. Specifically:

  

* Certain files are marked with `// SPDX-License-Identifier: MIT`, which is a permissive open-source license that allows anyone to use, copy, modify, and distribute the code, provided the original license is included.
* Other files use `// SPDX-License-Identifier: BUSL-1.1`, a more restrictive license that limits commercial use for a period of time, making it incompatible with **MIT** in terms of redistributing or combining the code.

  

This discrepancy could lead to **license incompatibility issues** when attempting to combine or redistribute the project's code as a whole, potentially violating the terms of one or both licenses. This is particularly problematic in open-source projects, where license compliance is crucial for legal and community standards.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

Ensureconsistent licensing across the project by reviewing and aligning all file headers to use the same license, depending on the intended usage and distribution model.

##### Remediation

**SOLVED:** The **Story team** solved the issue in the specified PRs. All contracts use the BUSL-1.1 license.

##### Remediation Hash

<https://github.com/storyprotocol/protocol-core-v1/pull/333 https://github.com/storyprotocol/protocol-core-v1/pull/338>

---
