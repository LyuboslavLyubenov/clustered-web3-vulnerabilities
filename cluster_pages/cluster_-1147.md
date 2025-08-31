# Cluster -1147

**Rank:** #369  
**Count:** 15  

## Label
Unauthorized function execution via insufficient access control and signature validation, enabling attackers to exploit or drain funds through arbitrary or impersonated calls.

## Cluster Information
- **Total Findings:** 15

## Examples

### Example 1

**Auto Label:** Proxy-based function routing flaws enabling unauthorized or unintended function execution through selector collisions or missing calldata validation.  

**Original Text Preview:**

**Description:** The `POLIsolationModeWrapperUpgradeableProxy` constructor accepts initialization calldata without performing any validation on its content before executing it via `delegatecall` to the implementation contract. Specifically:

- The constructor does not verify that the `calldata` targets the expected initialize(address) function
- The constructor does not verify that the provided vaultFactory address parameter is non-zero

```solidity
// POLIsolationModeWrapperUpgradeableProxy.sol
constructor(
    address _berachainRewardsRegistry,
    bytes memory _initializationCalldata
) {
    BERACHAIN_REWARDS_REGISTRY = IBerachainRewardsRegistry(_berachainRewardsRegistry);
    Address.functionDelegateCall(
        implementation(),
        _initializationCalldata,
        "POLIsolationModeWrapperProxy: Initialization failed"
    );
}
```
This lack of validation means the constructor will blindly execute any calldata, potentially setting critical contract parameters incorrectly during deployment.

Note that a similar issue exists in `POLIsolationModeUnwrapperUpgradeableProxy`


**Impact:** The proxy could be initialized with a zero or invalid vaultFactory address, rendering it non-functional or insecure.  Additionally, if the implementation contract is upgraded and introduces new functions with weaker access controls, this pattern would allow those functions to be called during initialization of new proxies.

**Recommended Mitigation:** Consider adding explicit validation for both the function selector and parameters in the constructor:

```solidity
constructor(
    address _berachainRewardsRegistry,
    bytes memory _initializationCalldata
) {
    BERACHAIN_REWARDS_REGISTRY = IBerachainRewardsRegistry(_berachainRewardsRegistry);

    // Validate function selector is initialize(address)
    require(
        _initializationCalldata.length == 36 &&
        bytes4(_initializationCalldata[0:4]) == bytes4(keccak256("initialize(address)")),
        "Invalid initialization function"
    );

     // Decode and validate the vaultFactory address is non-zero
    address vaultFactory = abi.decode(_initializationCalldata[4:], (address));
    require(vaultFactory != address(0), "Zero vault factory address");

    Address.functionDelegateCall(
        implementation(),
        _initializationCalldata,
        "POLIsolationModeWrapperProxy: Initialization failed"
    );
}
```

**Dolomite:**
Acknowledged.

**Cyfrin:** Acknowledged.

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

**Auto Label:** Proxy-based function routing flaws enabling unauthorized or unintended function execution through selector collisions or missing calldata validation.  

**Original Text Preview:**

## Vulnerability Report

**Difficulty:** Low  
**Type:** Undefined Behavior  

## Description

The `_onFetchDataFromResolver` function does not return an error if the sender does not have reading rights; instead, it silently returns empty bytes.

```solidity
function _onFetchDataFromResolver(bytes32 key, bool hash)
    internal
    view
    virtual
    override
    returns (bytes memory data)
{
    if (_callerIsAuthorized()) {
        data = super._onFetchDataFromResolver(key, hash);
    }
}
```
_Figure 2.1: `_onFetchDataFromResolver` function in AutomataDAOBase.sol#L17-L27_

Because of this behavior, a user calling an external function that relies on `_onFetchDataFromResolver`, such as the `getEnclaveIdentity` function, cannot tell the difference between a lack of available data and lack of reading access to the data.

```solidity
function getEnclaveIdentity(uint256 id, uint256 version)
    external
    view
    returns (EnclaveIdentityJsonObj memory enclaveIdObj)
{
    bytes memory attestedIdentityData = _onFetchDataFromResolver(ENCLAVE_ID_KEY(id, version), false);
    if (attestedIdentityData.length > 0) {
        (, enclaveIdObj.identityStr, enclaveIdObj.signature) =
        abi.decode(attestedIdentityData, (IdentityObj, string, bytes));
    }
}
```
_Figure 2.2: `getEnclaveIdentity` function in EnclaveIdentityDAO.sol#L69-L79_

The example above shows that an empty struct will be returned in both cases, so the user cannot tell the difference between them.

## Recommendations

**Short term:** Revert when an unauthorized user attempts to access the system.  
**Long term:** Thoroughly document the intended behavior of the function and what should happen when an unauthorized user interacts with it or when no data is available.

---
