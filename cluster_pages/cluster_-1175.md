# Cluster -1175

**Rank:** #450  
**Count:** 7  

## Label
Improper data layout and type usage lead to state inconsistencies, incorrect comparisons, and ambiguous state transitions, enabling unintended behavior and security risks through flawed storage packing and type selection.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Poor data handling and flawed type design lead to incorrect state, inefficient execution, and subtle behavioral inconsistencies.  

**Original Text Preview:**

##### Description

The `lockLiquidity` function relies on string comparisons to determine the `animalType` tier using `keccak256` hashing. This approach incurs unnecessary gas costs and increases computational complexity. Strings are less gas-efficient than alternatives like enums, which are compact and avoid the need for hashing or string processing.

##### BVSS

[AO:S/AC:L/AX:L/C:N/I:H/A:N/D:N/Y:N/R:N/S:U (1.5)](/bvss?q=AO:S/AC:L/AX:L/C:N/I:H/A:N/D:N/Y:N/R:N/S:U)

##### Recommendation

Replace the string-based `animalType` mechanism with an enum. Enums are gas-efficient, more readable, and prevent invalid inputs. Example:

1. Define an `enum` for animal tiers:

```
   enum AnimalType { Crab, Octopus, Fish, Dolphin, Shark, Whale }

```

1. Update `lockLiquidity` to accept `AnimalType` as an input:

```
   function lockLiquidity(uint256 months, AnimalType animalType) external payable {
       ...
       require(
           (animalType == AnimalType.Crab && ethValueInUSD >= 100e6) ||
           (animalType == AnimalType.Octopus && ethValueInUSD >= 500e6) ||
           ...
       );
       ...
   }

```

1. Emit the `animalType` enum as an integer in the event for easier off-chain processing.

Switching to enums eliminates the gas costs associated with string hashing and improves overall contract efficiency and clarity.

##### Remediation

**SOLVED**: The code is now using enum instead of strings.

##### Remediation Hash

1f89558c1394d2c6a59238172e3e17ed50e32265

---
### Example 2

**Auto Label:** Improper data layout and type usage lead to state inconsistencies, incorrect comparisons, and ambiguous state transitions, enabling unintended behavior and security risks through flawed storage packing and type selection.  

**Original Text Preview:**

## CyclicBoundedVec::eq

CyclicBoundedVec::eq compares the underlying data of two vectors without accounting for the cyclic nature of the vector, which may result in incorrect outcomes. This is because the data in CyclicBoundedVec may be stored in a non-contiguous manner due to the cyclic indexing, implying that the order of elements in the data array may not match the logical order of elements in the vector.

> _merk le-tree/bound ed-vec/src/lib.rs rust_

```rust
impl<'a, T> PartialEq for CyclicBoundedVec<'a, T>
where
    T: Clone + PartialEq,
{
    fn eq(&self, other: &Self) -> bool {
        self.data[..self.length].iter().eq(other.data.iter())
    }
}
```

The function creates a slice from the start of `self.data` up to `self.length` and iterates over the elements of the slice. It compares the elements of the iterator from `self.data` with those from `other.data`. This approach assumes that `self.data` and `other.data` are in the same order, but in a `CyclicBoundedVec`, the logical order may be different due to the cyclic indexing, specifically the `first_index` and `last_index`.

## Remediation

Ensure the comparison accounts for the cyclic ordering by iterating through the elements in their logical order to properly compare two `CyclicBoundedVec` instances.

## Patch

Resolved in 534d90f.

© 2024 Otter Audits LLC. All Rights Reserved. 25/53

---
### Example 3

**Auto Label:** Improper data layout and type usage lead to state inconsistencies, incorrect comparisons, and ambiguous state transitions, enabling unintended behavior and security risks through flawed storage packing and type selection.  

**Original Text Preview:**

The [`_computePublicInput` function](https://github.com/Consensys/zkevm-monorepo/blob/7f3614e59901402c804e72396afedb15263dcbe4/contracts/contracts/LineaRollup.sol#L666) extracts several fields from a [`FinalizationDataV2` struct](https://github.com/Consensys/zkevm-monorepo/blob/7f3614e59901402c804e72396afedb15263dcbe4/contracts/contracts/interfaces/l1/ILineaRollup.sol#L111) using hard-coded numeric offsets.


For code clarity, consider defining named offset constants next to the struct so they can be validated directly.


***Update:** Resolved. This is not an issue. The Linea team stated:*



> *As discussed, the interface does not support constants and we can disregard this issue.*

---
