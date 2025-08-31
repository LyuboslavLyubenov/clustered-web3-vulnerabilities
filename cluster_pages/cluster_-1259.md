# Cluster -1259

**Rank:** #318  
**Count:** 22  

## Label
Hardcoded or unverified addresses and missing interface validation enable attackers to manipulate or misconfigure critical contract interactions, leading to failed operations, data corruption, or unauthorized execution.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Lack of address and contract validation enables unauthorized or malicious contract injection, leading to privilege escalation, fund loss, or denial of service through improper address or code verification.  

**Original Text Preview:**

##### Description
The protocol integrates with a yield platform that may introduce new user functions in the future. To allow clients to call these yet-unknown functions without bypassing system restrictions, a generic function `callAnyFunction` is implemented. This function validates calldata using a fixed set of configurable rules (`None`, `StartsWith`, `EndsWith`, `AnyCalldata`). However, since the set of validation rules is limited, it might not be flexible enough to enforce more complex restrictions (e.g., ensuring the second argument matches the proxy’s address), potentially forcing a trade-off between excessive permissions and insufficient rights.

##### Recommendation
We recommend offloading the calldata validation to an external validator contract whose address is stored in the factory and can be updated by an operator. This approach would allow for more flexible and future-proof validation rules without needing to modify the core protocol logic.

---
### Example 2

**Auto Label:** Hardcoded or incorrect addresses in contracts lead to failed interactions, unintended transfers, or loss of funds due to misalignment with actual on-chain token or protocol addresses.  

**Original Text Preview:**

## \[L-01] Incorrect event emitted in \`setUpdateWeightRunnerAddress()\` function

## Summary

Incorrect event emitted in \`setUpdateWeightRunnerAddress()\` function. When changing the `UpdateWeightRunner.sol` address, the event should emit 2 params - old and new addresses:

```solidity
event UpdateWeightRunnerAddressUpdated(address indexed oldAddress, address indexed newAddress);
```

The issue is that this function does not store the old address in memory, but returns the updated `updateWeightRunner` variable as first param instead of old `updateWeightRunner`:

```solidity
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
        updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
        emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner); 
    }
```

## Impact

Frontend or other off-chain services may display incorrect values, potentially misleading users, disrupt update tracking and make it difficult to find updates.

## Recommendations

```diff
function setUpdateWeightRunnerAddress(address _updateWeightRunner) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
+       address oldUpdateWeightRunner = updateWeightRunner;
        updateWeightRunner = UpdateWeightRunner(_updateWeightRunner);
-       emit UpdateWeightRunnerAddressUpdated(address(updateWeightRunner), _updateWeightRunner);
+       emit UpdateWeightRunnerAddressUpdated(oldUpdateWeightRunner, _updateWeightRunner);  
    }
```

---
### Example 3

**Auto Label:** Hardcoded or unverified addresses and missing interface validation enable attackers to manipulate or misconfigure critical contract interactions, leading to failed operations, data corruption, or unauthorized execution.  

**Original Text Preview:**

##### Description

The `BasePaymaster` contract lacks interface validation for the \_entryPoint parameter in its constructor. The current implementation directly sets the `EntryPoint` address without verifying if it implements a matching `IEntryPoint` interface.

```
constructor(IEntryPoint _entryPoint, address _owner) Ownable(_owner) {

	//E @tocheck missing sanity check for _entryPoint

	setEntryPoint(_entryPoint);
}
```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

It is recommended to implement the same mechanism as it is done in eth-infinitism [development branch](https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/core/EntryPoint.sol#L49) repository which includes proper validation with an interface to be implemented:

```
constructor(IEntryPoint _entryPoint) Ownable(msg.sender) {

	_validateEntryPointInterface(_entryPoint);
	
	entryPoint = _entryPoint;

}

function _validateEntryPointInterface(IEntryPoint _entryPoint) internal virtual {
   require(IERC165(address(_entryPoint)).supportsInterface(type(IEntryPoint).interfaceId), "IEntryPoint interface mismatch");
}
```

##### Remediation

**ACKNOWLEDGED:** An on chain version of this file will be used instead of the one in the repository, but the project does not use the affected code.

##### References

[RunOnFlux/account-abstraction/contracts/erc4337/core/BasePaymaster.sol#L19](https://github.com/RunOnFlux/account-abstraction/blob/master/contracts/erc4337/core/BasePaymaster.sol#L19)

---
