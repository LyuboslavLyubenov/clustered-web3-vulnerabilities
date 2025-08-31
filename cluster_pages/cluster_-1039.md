# Cluster -1039

**Rank:** #35  
**Count:** 580  

## Label
Redundant validation and control flow checks that fail to enforce critical invariants, leading to unnecessary gas costs, potential state corruption, or inconsistent execution—often due to poor logic design or lack of abstraction.

## Cluster Information
- **Total Findings:** 580

## Examples

### Example 1

**Auto Label:** Redundant condition checks and unnecessary state mutations lead to inefficiency, increased gas costs, and potential logic errors or race conditions in critical control flows.  

**Original Text Preview:**

The `_reset` function in the `Voter` contract contains a redundant check for the variable `_votes`, which is of type `uint256`.

```solidity
uint256 _votes = votes[_tokenId][_pool];
```

The first `if` statement checks whether `_votes` is not equal to zero. However, since `_votes` is a `uint256`, it cannot be negative, and if it is not zero, it must be greater than zero. Therefore, the second check for `_votes` > 0 is redundant and leads to unreachable code in the `else` block.

```solidity
if (_votes != 0) {
    _updateFor(gauges[_pool]);
    weights[_pool] -= _votes;
    votes[_tokenId][_pool] -= _votes;
    if (_votes > 0) {
        IBribe(internal_bribes[gauges[_pool]])._withdraw(
            uint256(_votes),
            _tokenId
        );
        IBribe(external_bribes[gauges[_pool]])._withdraw(
            uint256(_votes),
            _tokenId
        );
        _totalWeight += _votes;
    } else {
        _totalWeight -= _votes;
    }
}
```

It is recommended to remove the redundant check for `_votes > 0` and simplify the code.

---
### Example 2

**Auto Label:** Redundant condition checks and unnecessary state mutations lead to inefficiency, increased gas costs, and potential logic errors or race conditions in critical control flows.  

**Original Text Preview:**

The `_reset` function in the `Voter` contract contains a redundant check for the variable `_votes`, which is of type `uint256`. 

```solidity
uint256 _votes = votes[_tokenId][_pool];
```

The first `if` statement checks whether `_votes` is not equal to zero. However, since `_votes` is a `uint256`, it cannot be negative, and if it is not zero, it must be greater than zero. Therefore, the second check for `_votes` > 0 is redundant and leads to unreachable code in the `else` block.

```solidity
if (_votes != 0) {
    _updateFor(gauges[_pool]);
    weights[_pool] -= _votes;
    votes[_tokenId][_pool] -= _votes;
    if (_votes > 0) {
        IBribe(internal_bribes[gauges[_pool]])._withdraw(
            uint256(_votes),
            _tokenId
        );
        IBribe(external_bribes[gauges[_pool]])._withdraw(
            uint256(_votes),
            _tokenId
        );
        _totalWeight += _votes;
    } else {
        _totalWeight -= _votes;
    }
}
```

It is recommended to remove the redundant check for `_votes > 0` and simplify the code.

---
### Example 3

**Auto Label:** Redundant state mutations and unnecessary storage operations that waste gas, introduce inefficiency, and risk unintended state inconsistencies without affecting functionality.  

**Original Text Preview:**

**Description:** Both `pUSDeVault` and `yUSDeVault` inherit the `PreDepositVault` which in turn inherits the `PreDepositPhaser`; however, there is an inconsistency between the state of `pUSDe::currentPhase`, which is updated when the phase changes, and `yUSDe::currentPhase`, which is never updated and is thus always the default `PointsPhase` variant. This is assumedly not an issue given that this state is never needed for the yUSDe vault, though a view function is exposed by virtue of the state variable being public which could cause confusion.

**Recommended Mitigation:** The simplest solution would be modifying this state to be internal by default and only expose the corresponding view function within `pUSDeVault`.

**Strata:** Fixed in commit [aac3b61](https://github.com/Strata-Money/contracts/commit/aac3b617084fb5a06b29728a9f52e5884b062b6a).

**Cyfrin:** Verified. The `yUSDeVault` now returns the `pUSDeVault` phase state.

\clearpage

---
