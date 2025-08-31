# Cluster -1092

**Rank:** #112  
**Count:** 152  

## Label
Missing input validation and unchecked fee parameter manipulation enable malicious actors to bypass protocol fees, redirect revenue, or render token sales unviable—leading to revenue loss, rug pulls, and governance failures.

## Cluster Information
- **Total Findings:** 152

## Examples

### Example 1

**Auto Label:** Missing input or state validation leads to exploitable fee or parameter manipulation, enabling incorrect logic, overflow, or unfair economic outcomes.  

**Original Text Preview:**

In the `updateFeeToken`, `updateFeeAmount`, and `updateFeeTo` functions, the fee config is not checked to ensure it has been properly set beforehand. As a result, an editor could mistakenly update the fee config for a chain ID where the fee config has not been set.

Example of impact: If the editor updates the fee token, the fee config could not be set for that chain ID.

---
### Example 2

**Auto Label:** Missing or flawed fee validation leads to unauthorized fee setting, incorrect fee calculation, or denial-of-service, enabling economic manipulation or transaction failure.  

**Original Text Preview:**

**Description:** Currently the protocol allows users to over-pay:
```solidity
function _revertIfInsufficientFee() internal view {
    if (msg.value < _getFee()) revert SoulBoundToken__InsufficientFee();
}
```

Consider changing this to require the exact fee to prevent users from accidentally over-paying:
```solidity
function _revertIfIncorrectFee() internal view {
    if (msg.value != _getFee()) revert SoulBoundToken__IncorrectFee();
}
```

[Fat Finger](https://en.wikipedia.org/wiki/Fat-finger_error) errors have previously resulted in notorious unintended errors in financial markets; the protocol could choose to be defensive and help protect users from themselves.

**Evo:**
Fixed in commit [e3b2f74](https://github.com/contractlevel/sbt/commit/e3b2f74239601b2721118e11aaa92b42dbb502e9).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Insufficient input validation and bounds checking enable attackers to manipulate or nullify fees, leading to loss of value, unauthorized fee redirection, or system failure through improper parameter handling.  

**Original Text Preview:**

**Description:** `PolygonStrategy::addFee` does not check if `_feeBasisPoints > 0` when adding a new fee element to the `fees` array. This is inconsistent with the `PolygonStrategy::updateFee` logic where `_feeBasisPoints = 0` triggers a removal of the fee element from the `fees` array.

**Recommended Mitigation:** Consider introducing a non-zero check to `_feeBasisPoints`.

**Stake.Link:** Acknowledged. Owner will ensure the correct value is set.

**Cyfrin:** Acknowledged.

---
