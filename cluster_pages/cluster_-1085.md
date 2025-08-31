# Cluster -1085

**Rank:** #314  
**Count:** 22  

## Label
Missing state constraints allow malicious actors to bypass critical validation checks, enabling invalid state transitions and compromised proof integrity through insufficient logical enforcement of row-level execution boundaries.

## Cluster Information
- **Total Findings:** 22

## Examples

### Example 1

**Auto Label:** Missing state constraints allow malicious actors to bypass critical validation checks, enabling invalid state transitions and compromised proof integrity through insufficient logical enforcement of row-level execution boundaries.  

**Original Text Preview:**

## Context

**File:** `air.rs#L476-L480`

## Description

The variable `start_top_level` is supposed to be true only once during top-level processing, specifically during the first row incorporation. However, the current implementation in the code does not enforce these constraints. Setting `start_top_level` in the middle of the top-level process can influence the value expected as a result of the `row_hash`:

```rust
let row_hash = from_fn(|i| {
    (start_top_level * left_output[i])
    + ((AB::Expr::ONE - start_top_level) * right_input[i])
});
```

This means that a malicious prover could set the value of the incorporated row to the preimage of `left_output[i]` (which should be `left_input[i] || right_input[i]`), provided that the length (`opened_len`) to ingest matches as well.

It is worth noting that depending on the previous row's `root_is_on_right` value, either `right_input` or `left_input` is completely unconstrained and can be supplied arbitrarily by the prover.

## Recommendation

Apply the following additional checks:

- The first row of the top level is `incorporate_row`.
- If `local.start_top_level = 1`, then `next.start_top_level = 0`.
- If `local.start_top_level = 0`, then `next.start_top_level = 0`.

## OpenVM

This issue has been fixed in commit `d9f525b1`. The fix changes the existing constraint to properly enforce that `start_top_level = 1` if and only if it's the beginning of a top-level process (meaning the previous row is `end` and the current row is `incorporate_row`). Previously, the only constraint was that the top-level had `start_top_level = 1`, but there was no enforcement that it must equal `0` on non-top levels.

## Cantina Managed

Fix verified.

---
### Example 2

**Auto Label:** Missing state constraints allow malicious actors to bypass critical validation checks, enabling invalid state transitions and compromised proof integrity through insufficient logical enforcement of row-level execution boundaries.  

**Original Text Preview:**

## Context

Refer to `air.rs#L155-L172` for details.

## Description

When using an `incorporate_row` type during a verify batch instruction, execution of the actual ingestion of the row is "deferred" using the internal bus:

```rust
self.internal_bus.interact(
    builder,
    true,
    incorporate_row,
    timestamp_after_initial_reads.clone(),
    end_timestamp - AB::F::TWO,
    opened_base_pointer,
    opened_element_size_inv,
    initial_opened_index,
    final_opened_index,
    row_hash,
);
```

This means that the actual row hashing is constrained by a series of `inside_row` rows, which will emit the associated receive event once finished:

```rust
// end
self.internal_bus.interact(
    builder,
    false,
    end_inside_row,
    very_first_timestamp,
    start_timestamp + AB::F::from_canonical_usize(2 * CHUNK),
    opened_base_pointer,
    opened_element_size_inv,
    initial_opened_index,
    cells[CHUNK - 1].opened_index,
    left_output,
);
```

To ensure the row is hashed correctly, a series of state transition rules is enforced between consecutive `inside_row` rows:

```rust
// things that stay the same (roughly)
builder.when(inside_row - end_inside_row).assert_eq(
    next.start_timestamp,
    start_timestamp + AB::F::from_canonical_usize(2 * CHUNK),
);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.opened_base_pointer, opened_base_pointer);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.opened_element_size_inv, opened_element_size_inv);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.initial_opened_index, initial_opened_index);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.very_first_timestamp, very_first_timestamp);
```

Unfortunately, it is not enforced that until the end of inside rows (`end_inside_row`) all rows are of type `inside_row`, and rows of type `incorporate_sibling` can be appended after a series of `inside_row` to bypass the initialization checks of the "top-level" phase (indeed, initialization checks are carried only if the previous row was an end one).

## Recommendation

Ensure that until `end_inside_row` only `inside_row` type rows are used:

```rust
// things that stay the same (roughly)
builder.when(inside_row - end_inside_row).assert_eq(
    next.start_timestamp,
    start_timestamp + AB::F::from_canonical_usize(2 * CHUNK),
);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.opened_base_pointer, opened_base_pointer);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.opened_element_size_inv, opened_element_size_inv);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.initial_opened_index, initial_opened_index);
builder
    .when(inside_row - end_inside_row)
    .assert_eq(next.very_first_timestamp, very_first_timestamp);
+ builder
+     .when(inside_row - end_inside_row)
+     .assert_eq(next.inside_row, inside_row);
```

## OpenVM

Fixed in commit `2fe9ad79`, the fix adds the constraint `builder.when(inside_row).when(AB::Expr::ONE - end_inside_row).assert_one(next.inside_row)`. This ensures that an Inside Row row that is not the last one in its block is followed by another Inside Row row, fixing the issue that Top Level rows could be interspersed inside Inside Row rows. While this differs slightly from the fix suggested in the finding, this constraint more clearly expresses the intent.

## Cantina Managed

Fix verified.

---
### Example 3

**Auto Label:** Missing state constraints allow malicious actors to bypass critical validation checks, enabling invalid state transitions and compromised proof integrity through insufficient logical enforcement of row-level execution boundaries.  

**Original Text Preview:**

## Context: air.rs#L109-L112

## Description
The verify batch instruction should always start with a row that is of type `incorporate_row` and `start_top_level`, as hinted by the constraint:

- **poseidon2/air.rs#L109-L112:**
  
  ```rust
  let end = end_inside_row + end_top_level + simple + (AB::Expr::ONE - enabled.clone());
  builder
      .when(end.clone())
      .when(next.incorporate_row)
      .assert_one(next.start_top_level);
  ```

However, only the following implication is constrained by these conditions:
If the first row is an `incorporate_row`, then `start_top_level` must be true. This means that all of the constraints on the actual opened values, enforced by the internal bus interactions, will be skipped if the first row is `incorporate_sibling` instead (please note that it does not need to be `start_top_level`):

- **poseidon2/air.rs#L465-L476:**
  
  ```rust
  self.internal_bus.interact(
      builder,
      true,
      incorporate_row,
      timestamp_after_initial_reads.clone(),
      end_timestamp - AB::F::TWO,
      opened_base_pointer,
      opened_element_size_inv,
      initial_opened_index,
      final_opened_index,
      row_hash,
  );
  ```

And a malicious prover would be able to provide top-level hashes directly without proving that the row hashed correctly into the leaf provided to the proof.

## Recommendation
Enforce that:
1. After `end`, if `incorporate_row` is true, `start_top_level` is true (already enforced).
2. After `end`, it is not possible to provide `incorporate_sibling` directly:

- **poseidon2/air.rs#L109-L112:**

  ```rust
  let end = end_inside_row + end_top_level + simple + (AB::Expr::ONE - enabled.clone());
  builder
      .when(end.clone())
      .when(next.incorporate_row)
      .assert_one(next.start_top_level);
  + builder.when(end.clone()).assert_zero(next.incorporate_sibling);
  ```

## OpenVM
Fixed in commit `7468e84a`, the fix adds the constraint `builder.when(end.clone()).assert_zero(next.incorporate_sibling)`. This ensures that a row with `end = 1` can only be followed by either an `Inside Row` row or an `Incorporate Sibling` row. The existing constraint `builder.when(end.clone()).when(next.incorporate_row).assert_one(next.start_top_level)` then ensures that when it is an `Incorporate Row` row, we enforce `start_top_level = 1.

## Cantina Managed
Fix verified.

---
