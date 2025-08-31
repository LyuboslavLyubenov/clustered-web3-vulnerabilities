# Cluster -1354

**Rank:** #182  
**Count:** 66  

## Label
Failure to synchronize or validate state during critical operations leads to irreversible fund loss, incorrect balances, or denial-of-service, enabling irreversible or erroneous token transfers.

## Cluster Information
- **Total Findings:** 66

## Examples

### Example 1

**Auto Label:** Failure to synchronize critical state across contracts leads to unauthorized access or permanent loss of funds due to improper asset handling and lack of recovery mechanisms.  

**Original Text Preview:**

## Severity

High Risk

## Description

In the `DaosLive::finalize()` function, there's a parameter `_mintToDao()` that allows minting additional tokens to the contract address.

These tokens are minted to the contract address (`address(this)`), but there is no functionality implemented to allow the DAO manager to access or utilize these tokens. The tokens become permanently locked in the contract.

## Location of Affected Code

File: [contracts/DaosLive.sol](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosLive.sol)

```solidity
 function finalize(
    string calldata name,
    string calldata symbol,
    IDaosVesting.Schedule[] calldata schedules,
@-> uint256 _mintToDao,
    uint256 _snipeAmount // eth
) external onlyOwner {
    // code

    uint256 mintToDao = _mintToDao;
    if (mintToDao > 0) {
@->    IDaosToken(token).mint(address(this), mintToDao);
    }

    // code
}
```

## Impact

- Tokens minted through `_mintToDao()` are permanently locked in the contract, leading to a permanent loss of tokens, as there's no way to recover them
- Affects all DAOs that use the `_mintToDao()` parameter during `finalize()`

## Recommendation

- Add a dedicated function to transfer tokens to the DAO manager.
- Modify the `finalize()` function to mint directly to the DAO manager or any recipient address.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Failure to synchronize critical state across contracts leads to unauthorized access or permanent loss of funds due to improper asset handling and lack of recovery mechanisms.  

**Original Text Preview:**

## Severity

High Risk

## Description

When performing a token swap for `snipeAmount` in the `finalize()` function, using the `IUniRouter.exactInputSingle()` function in the following code snippet:

```solidity
IUniRouter uniRouter = IUniRouter(_factory.uniRouter());
uniRouter.exactInputSingle{value: snipeAmount}(
    IUniRouter.ExactInputSingleParams({
        tokenIn: wethAddr,
        tokenOut: token,
        fee: UNISWAP_V3_FEE,
        recipient: address(this), // swapped tokens sent here
        amountIn: snipeAmount,
        amountOutMinimum: 0,
        sqrtPriceLimitX96: 0
    })
);
```

The `recipient` of the swapped tokens is set to `address(this)`, i.e., the `DaosLive.sol` contract. However, the contract lacks any function or mechanism to retrieve or forward these tokens, resulting in the swapped tokens being permanently stuck in the contract.

## Location of Affected Code

File: [contracts/DaosLive.sol#L314](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosLive.sol#L314)

## Impact

Swapped tokens are locked and inaccessible forever.

## Recommendation

- Implement a **withdrawal** or **sweep** function in `DaosLive.sol` that allows an authorized address (e.g., owner or governance) to recover any ERC20 tokens held by the contract.
- Alternatively, consider making the `recipient` parameter configurable or passed in explicitly, allowing swapped tokens to be sent to a retrievable address.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** **State inconsistency and unsynchronized contract configurations lead to functional failures, fund loss, or unauthorized control via improper data migration or address reuse.**  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Summary
The `VaultFactory::createVault` function is missing access controls, allowing anyone to call it. This vulnerability can be abused to DDoS and/or hijack legitimate vault creation attempts.

## Finding Description
In the `VaultFactory::createVault` function, the vault address salt is generated using the following parameters:
- **admin**
- **asset**
- **name**
- **symbol**
- **salt**

Changing any of these parameters results in a different vault address, which is not likely to cause problems. However, the caller must also provide these parameters as part of `initialParams`:
- **curator**
- **timelock**
- **maxCapacity**
- **performanceFeeRate**

Consider the following scenario:
1. Alice submits a transaction to create a vault.
2. Bob front-runs that transaction with his own, modifying the curator, timelock, maxCapacity, and performanceFeeRate values.
3. Bob's transaction succeeds.
4. Alice's transaction reverts because the vault address is already taken.

Two possibilities arise:
- Most likely, Alice's transaction failure results in the vault created by Bob never being used because Alice is aware Bob's vault is not identical to the one she wanted to create. In this case, an attacker can DoS the protocol by front-running every legitimate transaction for creating a vault.
- However, if Alice ignores the transaction failure and still goes ahead with using the vault at the predetermined address, Bob has successfully hijacked the curator and allocator roles of the vault and can perform all the privileged actions associated with this role.

## Impact Explanation
This attack is either a continuous DoS for `VaultFactory` or a trusted role hijack, depending on how the legitimate vault creator handles their transaction reverting.

## Likelihood Explanation
This attack can be performed by anyone whenever a new vault is deployed.

## Proof of Concept
Create file `./termmax-contracts/test/VaultDos.t.sol` and add the following test to it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;
import {console, Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {TermMaxVault, ITermMaxVault} from "contracts/vault/TermMaxVault.sol";
import {OrderManager} from "contracts/vault/OrderManager.sol";
import {VaultInitialParams} from "contracts/storage/TermMaxStorage.sol";
import {VaultFactory} from "contracts/factory/VaultFactory.sol";

contract VaultDosTest is Test {
    address deployer = vm.randomAddress();
    address curator = vm.randomAddress();
    address alice = vm.randomAddress();
    address bob = vm.randomAddress();
    ITermMaxVault vault;
    VaultInitialParams initialParams;
    uint256 timelock = 86400;
    uint256 maxCapacity = 1000000e18;
    uint64 performanceFeeRate = 0.5e8;

    function testPoc_vaultDoS() public {
        OrderManager orderManager = new OrderManager();
        TermMaxVault implementation = new TermMaxVault(address(orderManager));
        VaultFactory vaultFactory = new VaultFactory(address(implementation));
        
        // set initial parameters
        initialParams = VaultInitialParams(
            deployer,
            curator,
            timelock,
            IERC20(address(1)),
            maxCapacity,
            "Vault-DAI",
            "Vault-DAI",
            performanceFeeRate
        );

        // vault with the first version of initial parameters is deployed successfully
        vm.prank(bob);
        vaultFactory.createVault(initialParams, 0);

        // update initial parameters
        initialParams.curator = address(12345);
        initialParams.timelock = timelock * 2;
        initialParams.maxCapacity = maxCapacity * 2;
        initialParams.performanceFeeRate = performanceFeeRate * 2;

        // vault with the second version of initial parameters can't be deployed
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSignature("FailedDeployment()"));
        vaultFactory.createVault(initialParams, 0);
    }
}
```

## Recommendation
Include all vault parameters in the salt of the vault address such that changing any parameter will change the address. Alternatively, only allow a trusted address to create the vaults.

## Term Structure
Addressed in PR 4 and PR 5.

## Cantina Managed
Fixes verified.

---
