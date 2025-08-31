# Cluster -1089

**Rank:** #396  
**Count:** 12  

## Label
Redundant storage usage leading to increased gas costs, inconsistent state, and poor performance due to unnecessary variable declarations, improper memory-to-storage interactions, and inefficient storage reads.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Redundant storage usage leading to increased gas costs, inconsistent state, and poor performance due to unnecessary variable declarations, improper memory-to-storage interactions, and inefficient storage reads.  

**Original Text Preview:**

##### Description
The storage variables `extraImplicit` and `extraExplicit` are declared twice in the code:
- [ControllerV2.sol L39-L43](https://github.com/dforce-network/LendingContracts/blob/6f3a7b6946d8226b38e7f0d67a50e68a28fe5165/contracts/ControllerV2.sol#L39-L43)
- [ControllerStorage.sol L221-L225](https://github.com/dforce-network/LendingContracts/blob/6f3a7b6946d8226b38e7f0d67a50e68a28fe5165/contracts/ControllerStorage.sol#L221-L225)
This could potentially lead to unexpected behavior.
##### Recommendation
We recommend removing the redundant declaration in `ControllerV2.sol`.

---
### Example 2

**Auto Label:** Redundant storage usage leading to increased gas costs, inconsistent state, and poor performance due to unnecessary variable declarations, improper memory-to-storage interactions, and inefficient storage reads.  

**Original Text Preview:**

**Severity**: Informational	

**Status**: Acknowledged

**Description**

The `Fias.sol` contract contains a `CONTRACT_ADMIN` address which is not used in any part of the code.

**Recommendation**:

Consider removing this variable to save gas.

---
### Example 3

**Auto Label:** Inefficient memory access leading to unnecessary data copying and increased gas costs, primarily due to redundant struct operations and poor data targeting.  

**Original Text Preview:**

## Context
- `PoolConfigurator.sol#L469`
- `PoolConfigurator.sol#L479`

## Description
Both functions mentioned above invoke the `Pool::getReserveData` function in order to retrieve a copy of the reserve state data from the Pool's storage. However, the `getReserveData` function will now perform extra memory operations in order to copy the reserve state into a new `ReserveDataLegacy` struct. This is unnecessary in the context of the two functions shown above since the only reserve value utilized in these function calls is the `reserveInterestRateStrategyAddress`, which is present in both the `ReserveData` and `ReserveDataLegacy` structs. Therefore, the `Pool::getReserveDataExtended` function can instead be invoked in order to directly retrieve a single copy of the reserve data from the Pool's storage via a `ReserveData` struct.

## Recommendation
Consider using the `Pool::getReserveDataExtended` function instead of `Pool::getReserveData` to improve gas efficiency.

---
