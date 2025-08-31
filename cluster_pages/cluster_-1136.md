# Cluster -1136

**Rank:** #353  
**Count:** 17  

## Label
Failure to validate module state or lifecycle leads to unauthorized execution, state inconsistency, or unintended module operations, compromising security and transaction integrity.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Failure to validate module state or lifecycle leads to unauthorized execution, state inconsistency, or unintended module operations, compromising security and transaction integrity.  

**Original Text Preview:**

**Impact**

`_isModuleEnabled` is written as follows:

https://github.com/onchainification/aura_locker_v2/blob/07294ae3638909ecd768a6a0f831fa513abe91a0/src/AuraLockerModule.sol#L123

```solidity
    /// @dev The Gnosis Safe v1.1.1 does not yet have the `isModuleEnabled` method, so we need a workaround
    function _isModuleEnabled() internal view returns (bool) {
        address[] memory modules = SAFE.getModules();
        for (uint256 i = 0; i < modules.length; i++) {
            if (modules[i] == address(this)) return true;
        }
        return false;
    }
```

Which uses `getModules`, which is paginated and limited to the first 10 enabled modules

https://etherscan.io/address/0x34cfac646f301356faa8b21e94227e3583fe3f5f#code
```solidity
    function getModules()
        public
        view
        returns (address[] memory)
    {
        (address[] memory array,) = getModulesPaginated(SENTINEL_MODULES, 10);
        return array;
    }
```

It's worth noting that if this module where to be used with a lot of other modules setup, the check could fail

However, current there are no other modules set, meaning that the code is safe as is

Additionally, even if the check were to fail, no particular damage would be caused to the Safe, at worst the `checkUpkeep` would always return false, making no upkeep run, but causing no DOS to the Safe

**Mitigation**

Add a comment to the module and make sure to have less than 10 modules

---

---
### Example 2

**Auto Label:** Failure to validate modules against a registry at runtime due to out-of-order execution, enabling unauthorized or revoked modules to operate with unchecked access.  

**Original Text Preview:**

## Security Report

## Severity
**Low Risk**

## Context
`RegistryBootstrap.sol#L46`

## Description
The `Bootstrap.init*` functions install modules before setting the registry. This will skip the registry check on these modules. Modules that are not validated could be installed during initialization.

## Recommendation
Set the registry before installing modules.

## Biconomy
Fixed in PR 115.

## Spearbit
Fixed.

---
### Example 3

**Auto Label:** Failure to validate module state or lifecycle leads to unauthorized execution, state inconsistency, or unintended module operations, compromising security and transaction integrity.  

**Original Text Preview:**

## Denial of Service Vulnerability in Oracle Contract

**Difficulty:** Low  
**Type:** Denial of Service  

## Description  
Users are required to include a nonce in each request, which must correspond to a global request counter for the Oracle contract. When many requests are included in a single block, users interacting with the Oracle contract from an externally owned account may be unable to predict the correct nonce ahead of time and find that all of their request transactions are reverted. 

Figure 14.1 shows the check that would fail if a user is unable to predict the correct nonce at the time of transaction signing.

```solidity
function _createRequest(Request calldata _request, bytes32 _ipfsHash) internal
returns (bytes32 _requestId) {
    uint256 _requestNonce = totalRequestCount++;
    if (_requestNonce != _request.nonce || msg.sender != _request.requester) revert Oracle_InvalidRequestBody();
}
```

**Figure 14.1:** Users must predict `totalRequestCount` before sending the transaction.

If multiple users submit requests using the `totalRequestCount` value in the current state of the contract, then at most one of those users will have their transactions succeed. Users interacting with the Oracle contract via a smart contract call will not experience this issue, since they can fetch the current count and then submit a request without preemption.

## Exploit Scenario  
Alice wants to make a request and uses a local DApp interface to interact with the Oracle smart contract. The DApp front end fetches the latest `totalRequestCount` value and includes it in a `Request` struct. Alice sends the transaction, which is then included in a block. However, earlier in the same block, Bob also makes an unrelated request. Alice’s transaction then fails, consuming her transaction fee, without successfully submitting her request. Alice may need to try many times before the contract accepts her request.

## Recommendations  
- **Short term:** Allow users to submit requests with `Request.nonce` set to zero and overwrite the value with the current `totalRequestCount` value before further processing.
- **Long term:** Add tests exercising the case where multiple EOAs send interleaved transactions to the contracts.

---
