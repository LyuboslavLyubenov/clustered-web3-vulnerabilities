# Cluster -1274

**Rank:** #459  
**Count:** 6  

## Label
Failure to validate function return values leads to unhandled execution errors and silent state mismatches, compromising correctness and reliability of critical operations.

## Cluster Information
- **Total Findings:** 6

## Examples

### Example 1

**Auto Label:** Failure to validate library function returns leads to invalid state and misleading events, enabling false signals and potential exploitation through silent duplicates.  

**Original Text Preview:**

## Vulnerability Report

**Difficulty:** N/A

**Type:** Undefined Behavior

## Description

The return values of the `EnumerableSet.remove` and `EnumerableSet.add` functions are not validated in the Agreement contract, as shown in figures 2.1 and 2.2. This means that adding a duplicate collateral token address or removing a collateral token address that does not exist will still emit a `CollateralAdded` or `CollateralRemoved` event, respectively.

```solidity
function removeCollaterals(address[] calldata tokens) external override onlyFactory {
    Storage storage $ = _storage();
    for (uint256 i = 0; i < tokens.length; i++) {
        $.collaterals.remove(tokens[i]);
        emit CollateralRemoved(tokens[i]);
    }
}
```
*Figure 2.1: The return value of `EnumerableSet.remove` is ignored, leading to an erroneous event emission. (contracts/agreement/Agreement.sol#158–164)*

```solidity
function addCollaterals(address[] calldata tokens) public override onlyFactory {
    Storage storage $ = _storage();
    for (uint256 i = 0; i < tokens.length; i++) {
        $.collaterals.add(tokens[i]);
        emit CollateralAdded(tokens[i]);
    }
}
```
*Figure 2.2: The return value of `EnumerableSet.add` is ignored, leading to an erroneous event emission. (contracts/agreement/Agreement.sol#135–141)*

## Exploit Scenario

The Arkis admin intends to remove USDC as a valid collateral token for agreement A. Due to a typo, the transaction sent tries to remove USDC as a valid collateral for agreement B, which already does not have USDC as a valid collateral.

When the transaction is executed, the Agreement still emits the `CollateralRemoved` event even though no action took place, reducing the probability that the error will be caught immediately.

## Recommendations

- **Short term:** Add code to handle the Boolean returned by `add` and `remove`, and consider having the functions revert if a collateral token is added twice or there is an attempt to remove a collateral token that never existed.
  
- **Long term:** Run Slither as a regular part of the development process. This finding was detected using the unused-return Slither detector, and regular use of Slither in the development process may have prevented the issue from being introduced.

---
### Example 2

**Auto Label:** Failure to validate library function returns leads to invalid state and misleading events, enabling false signals and potential exploitation through silent duplicates.  

**Original Text Preview:**

## Description

The return value of the `EnumerableSet.add` function is not validated. As a result, having duplicates within `_allowedActions` or `_allowedAssets` does not result in a revert. Instead, the call will succeed and emit the `StrategyCreated` event with duplicate values within the `_allowedAssets` and/or `_allowedActions` event fields. This issue has no other effects besides an incorrect event emission.

## Code Example

```solidity
function storeStrategy(
    address _strategy,
    bytes4[] calldata _allowedActions,
    address[] calldata _allowedAssets
) external onlyOwner returns (uint _strategyIndex) {
    if (strategies.add(_strategy) == false) revert AlreadyExist();
    _strategyIndex = strategies.length() - 1;
    for (uint i; i < _allowedActions.length; ) {
        parameters[_strategy].whitelistedActions.add(_allowedActions[i]);
        unchecked {
            ++i;
        }
    }
    for (uint i; i < _allowedAssets.length; ) {
        parameters[_strategy].whitelistedAssets.add(_allowedAssets[i]);
        unchecked {
            ++i;
        }
    }
    parameters[_strategy].isActive = true;
    emit StrategyCreated(_strategyIndex, _allowedAssets, _allowedActions);
}
```

**Figure 6.1:** The `storeStrategy` function in `StrategyStorage.sol`

**Figure 6.2** shows that the `add` function (and the `_add` function) in the `EnumerableSet` library returns a Boolean indicating if the addition was successful. If, for example, the addition failed because that value was already in the set, the `add` function will return `false`.

```solidity
function add(AddressSet storage set, address value) internal returns (bool) {
    return _add(set._inner, bytes32(uint256(uint160(value))));
}

function _add(Set storage set, bytes32 value) private returns (bool) {
    if (!_contains(set, value)) {
        set._values.push(value);
        // The value is stored at length-1, but we add 1 to all indexes
        // and use 0 as a sentinel value
        set._indexes[value] = set._values.length;
        return true;
    } else {
        return false;
    }
}
```

**Figure 6.2:** The `add` and `_add` functions in OpenZeppelin’s `EnumerableSet.sol`

## Recommendations

- **Short term:** Validate the return value of the call to `EnumerableSet.add`. Have it revert if the value is `false`.
- **Long term:** Read the source code of all external libraries/contracts to know how to write correct integrations. This helps to ensure error-free usage of external libraries and contracts.

---
### Example 3

**Auto Label:** Failure to validate function return values leads to unhandled execution errors and silent state mismatches, compromising correctness and reliability of critical operations.  

**Original Text Preview:**

**Severity**: Informational

**Status**: Acknowledged

**Location**: TeaVaultAmbient.sol

**Description**: 

In the functions setFeeConfig(...), deposit(...), and withdraw(...) within TeaVaultAmbient.sol, the return values of the functions _collectManagementFee() and _collectAllSwapFee() are not being utilized or checked. This may result in overlooked fee collection statuses.

**Recommendation**: 

Evaluate the return values of _collectManagementFee() and _collectAllSwapFee() within the setFeeConfig(...), deposit(...), and withdraw(...) functions. Ensure that the results are properly checked and handled to confirm proper execution and to appropriately address potential unexpected values.	

**Client comment**: Returned values are included in events.

---
