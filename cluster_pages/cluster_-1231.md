# Cluster -1231

**Rank:** #185  
**Count:** 64  

## Label
Failure to validate caller identity enables unauthorized access and front-running attacks, allowing attackers to steal tokens or disrupt staking through improper access control and missing sender checks.

## Cluster Information
- **Total Findings:** 64

## Examples

### Example 1

**Auto Label:** Lack of caller validation in callback mechanisms enables unauthorized execution, leading to fund theft, state manipulation, or reentrancy via forged or unverified pool calls.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

`Burve::uniswapV3MintCallback` is an external function with no access control. This function takes three parameters: `amount0Owed`, `amount1Owed`, and `data`. It then decodes `data` to get an address and transfers tokens from that address to the liquidity pool (lp). An attacker could see which addresses have approved to `Burve.sol` and transfer tokens from those addresses to the pool.

```solidity
  function uniswapV3MintCallback(uint256 amount0Owed, uint256 amount1Owed, bytes calldata data) external {
        address source = abi.decode(data, (address));
        TransferHelper.safeTransferFrom(token0, source, address(pool), amount0Owed);
        TransferHelper.safeTransferFrom(token1, source, address(pool), amount1Owed);
    }
```

1. Provide `data` which decodes to a user address with a hanging approval to `Burve` and amounts equal to the approved amounts
2. As the function has no access control, the funds will be transferred into the pool
3. This causes a direct loss of funds for the users

## Recommendations

Restrict this function so that only the pool can call it.

---
### Example 2

**Auto Label:** Unauthorized function execution via insufficient access control and signature validation, enabling attackers to exploit or drain funds through arbitrary or impersonated calls.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

The `OmoVault.sol` contract implements some principles of the ERC7540 standard, Asynchronous redeem shares for assets.
However, users are still able to withdraw directly using `ERC4626.sol#withdraw()`

Also, `_validateSignature()` function in `DynamicAccount.sol` contract accepts the user wallet as a signer, through the Entry Point.
So, the user has the same privileges as the agent.

Malicious users can call `withdraw()` from the vault directly to receive their funds.
Next, he will transfer all the funds and the UniswapV3 positions out of their Dynamic Account

As a result, he will duple his first deposit in the vault.

## Recommendations

1- Override `withdraw()` function in `OmoVault.sol` and make it revert.
2- In `_validateSignature()` you need to check what function can be called if the signer is the owner

---
### Example 3

**Auto Label:** Missing validation of caller or transaction legitimacy leads to unauthorized access, spoofing, or state manipulation, enabling attacks like reentrancy, denial-of-service, or cascading failures.  

**Original Text Preview:**

**Impact**

The function `updateSystemSnapshots_excludeCollRemainder` doesn't have access control meaning that stakes can be manipulated

I found this by running invariant tests, with the simple check that no call should ever succeed


```solidity
 => [call] CryticTester.troveManager_updateSystemSnapshots_excludeCollRemainder((address,uint256)[])([]) (addr=0xA647ff3c36cFab592509E13860ab8c4F28781a66, value=0, sender=0x0000000000000000000000000000000000030000)
         => [call] TroveManager.updateSystemSnapshots_excludeCollRemainder((address,uint256)[])([]) (addr=0x48E4F3f3daE11341ff2eDF60Af6857Ae08C871C5, value=0, sender=0xA647ff3c36cFab592509E13860ab8c4F28781a66)
                 => [event] SystemSnapshotsUpdated([], [])
                 => [return ()]
         => [event] Log("should never be possible")
         => [panic: assertion failed]
```

**Mitigation**

Add a check to ensure that the caller is LiquidationOperations

---
