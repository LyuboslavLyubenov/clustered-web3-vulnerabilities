# Cluster -1337

**Rank:** #62  
**Count:** 281  

## Label
Race conditions enabling front-running and state-dependent exploits, allowing attackers to manipulate share allocations, trigger reverts, or withdraw funds prematurely through transaction ordering vulnerabilities.

## Cluster Information
- **Total Findings:** 281

## Examples

### Example 1

**Auto Label:** Front-running and improper initialization expose contracts to unauthorized state manipulation, enabling attacks via premature or manipulated execution of initialization logic.  

**Original Text Preview:**

**Description:** The `SpinGame` contract is upgradeable but does **not** call `_disableInitializers()` in its constructor. In upgradeable contract patterns, this call is a best practice to prevent the implementation (logic) contract from being initialized directly.

While this doesn’t affect the proxy’s behavior, it helps protect against accidental or malicious use of the implementation contract in isolation, especially in environments where both proxy and implementation contracts are visible, like block explorers.

Consider adding the following line to the constructor of the `SpinGame` contract:

```solidity
constructor(address _trustedForwarderAddress) ERC2771ContextUpgradeable(_trustedForwarderAddress) {
    _disableInitializers();
}
```

This ensures that the implementation contract cannot be initialized independently.

**Linea:** Fixed in commit [`9f9d9fd`](https://github.com/Consensys/linea-hub/pull/554/commits/9f9d9fd76d2672f572e31079b5811bf6f0f48eed)

**Cyfrin:** Verified. Constructor now calls `_disableInitializers`.

---
### Example 2

**Auto Label:** Front-running and improper initialization expose contracts to unauthorized state manipulation, enabling attacks via premature or manipulated execution of initialization logic.  

**Original Text Preview:**

It's [best practice](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing_the_implementation_contract) to disable the ability to initialize the implementation contracts. The following contracts are missing `_disableInitializers()` call in their constructor:

- Voter.
- VotingEscrow.
- CLFactory.
- FactoryRegistry.
- CLGauge.

Consider adding a constructor with the OpenZeppelin `_disableInitializers` in all the upgradeable contracts:

```solidity
constructor() {
    _disableInitializers();
}
```

---
### Example 3

**Auto Label:** Front-running and improper initialization expose contracts to unauthorized state manipulation, enabling attacks via premature or manipulated execution of initialization logic.  

**Original Text Preview:**

The `CLFactory` contract uses an `initialize` function with the `initializer` modifier but does not implement any upgrade pattern (such as UUPSUpgradeable).

Recommendation to eliminate the risk of front-run initialization:

- If the contract is not intended to be upgradeable, replace the initialize function with a constructor.
- If upgradeability is desired, implement a proper upgrade pattern (e.g., UUPSUpgradeable) like the other contracts in the codebase.

---
