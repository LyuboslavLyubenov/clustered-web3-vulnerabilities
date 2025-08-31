# Cluster -1192

**Rank:** #287  
**Count:** 26  

## Label
Missing or misconfigured role-based access control leads to unauthorized administrative power or failed deployment, enabling unauthorized control or denial of critical functions.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

**Auto Label:** Inconsistent and incomplete admin access control leads to unauthorized operations, configuration gaps, and flawed authorization validation across critical functions and roles.  

**Original Text Preview:**

## Code Review Suggestions

1. **Separation of Admin Validation Principles**
   
   In the `InitSy::validate`, the current approach ties admin validation to `principles.jito_restaking`. It would be more appropriate to utilize separate principles for generic standard admin instructions.

   ```rust
   // src/instructions/admin/init_sy.rs
   pub fn validate(&self) -> Result<()> {
       self.admin_state
           .principles
           .jito_restaking
           .is_admin(&self.admin.key)?;
       Ok(())
   }
   ```

2. **Using `interface_type.to_bytes`**

   Utilize `interface_type.to_bytes` instead of the `INTERFACE_BYTES` array for converting the `interface_type` into a byte slice within `SyMeta::authority_seeds`.

## Remediation

Implement the above-mentioned suggestions.

---
### Example 2

**Auto Label:** Missing or misconfigured role-based access control leads to unauthorized administrative power or failed deployment, enabling unauthorized control or denial of critical functions.  

**Original Text Preview:**

**Description:** `QuantAMMBaseAdministration` is a contract that manages certain break-glass features and facilitates calls to other QuantAMM contracts.

However, this contract fails to deploy due to an issue in the [`QuantAMMBaseAdministration` constructor](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/QuantAMMBaseAdministration.sol#L68-L74), where roles are granted to proposers and executors:

```solidity
for (uint256 i = 0; i < proposers.length; i++) {
    timelock.grantRole(timelock.PROPOSER_ROLE(), proposers[i]);
}

for (uint256 i = 0; i < executors.length; i++) {
    timelock.grantRole(timelock.EXECUTOR_ROLE(), executors[i]);
}
```
The issue arises because the `QuantAMMBaseAdministration` contract needs to have the `DEFAULT_ADMIN_ROLE` in the `timelock` contract for these `grantRole` calls to succeed. Since it does not possess this role, the calls will [revert](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v5.0/contracts/access/AccessControl.sol#L122-L124) with an `AccessControlUnauthorizedAccount` error.

**Impact:** The deployment of `QuantAMMBaseAdministration` will fail due to unauthorized role assignment, preventing the contract from being deployed successfully.

**Proof of Concept:** Add the following test file to the `pkg/pool-quantamm/test/foundry/` folder:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "../../contracts/QuantAMMBaseAdministration.sol";

contract QuantAMMBaseAdministrationTest is Test {

    address daoRunner = makeAddr("daoRunner");

    address proposer = makeAddr("proposer");
    address executor = makeAddr("executor");

    QuantAMMBaseAdministration admin;

    function testSetupQuantAMMBaseAdministration() public {
        address[] memory proposers = new address[](1);
        proposers[0] = proposer;

        address[] memory executors = new address[](1);
        executors[0] = executor;

        vm.expectRevert(); // AccessControlUnauthorizedAccount(address(admin),DEFAULT_ADMIN_ROLE)
        admin = new QuantAMMBaseAdministration(
            daoRunner,
            1 hours,
            proposers,
            executors
        );
    }
}
```

**Recommended Mitigation:** Remove the role grants in the constructor, as these assignments are already handled in the `TimeLockController` [constructor]((https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v5.0/contracts/governance/TimelockController.sol#L125-L133)):

**QuantAMM:** Fixed in [`a299ce7`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a299ce72076b58503386bc146b81014560536d9e)

**Cyfrin:** Verified. `QuantAMMBaseAdministration` is removed.

\clearpage

---
### Example 3

**Auto Label:** Missing or misconfigured role-based access control leads to unauthorized administrative power or failed deployment, enabling unauthorized control or denial of critical functions.  

**Original Text Preview:**

**Description:** Several functions in `UpdateWeightRunner` require the `quantammAdmin` role, but these functions are missing from `QuantAMMBaseAdministration`. The affected functions are:

* [`UpdateWeightRunner::addOracle`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L182-L195)
* [`UpdateWeightRunner::removeOracle`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L197-L203)
* [`UpdateWeightRunner::setApprovedActionsForPool`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L205-L209)
* [`UpdateWeightRunner::setETHUSDOracle`](https://github.com/QuantAMMProtocol/QuantAMM-V1/blob/7213401491f6a8fd1fcc1cf4763b15b5da355f1c/pkg/pool-quantamm/contracts/UpdateWeightRunner.sol#L281-L287)

Since these functions rely on `quantammAdmin`, they cannot be executed unless the corresponding calls are included in `QuantAMMBaseAdministration`.

**Impact:** The `QuantAMMBaseAdministration` contract cannot execute the above functions. As a result, administrative actions such as adding or removing oracles, setting approved actions for pools, and updating the ETH/USD oracle cannot be performed, limiting the functionality and flexibility of the system.

**Recommended Mitigation:** Consider one of the following approaches:

1. Add the Missing Functions to `QuantAMMBaseAdministration`
2. Use OpenZeppelin’s `TimeLock` Contract Directly:
   If `QuantAMMBaseAdministration` is redundant, replace it with OpenZeppelin's `TimeLock` contract to manage administrative functions securely and effectively.

**QuantAMM:** Fixed in [`a299ce7`](https://github.com/QuantAMMProtocol/QuantAMM-V1/commit/a299ce72076b58503386bc146b81014560536d9e)

**Cyfrin:** Verified. `QuantAMMBaseAdministration` is removed.

---
