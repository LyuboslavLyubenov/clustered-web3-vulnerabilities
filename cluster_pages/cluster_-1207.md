# Cluster -1207

**Rank:** #146  
**Count:** 96  

## Label
Insufficient input validation and missing fee enforcement lead to unauthorized parameter manipulation, underpayment, or spam, enabling denial-of-service, incorrect state changes, or economic exploitation.

## Cluster Information
- **Total Findings:** 96

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

**Auto Label:** Failure to validate zero fee amounts leads to reverts during token transfers, causing denial-of-service or incorrect state updates in critical functions.  

**Original Text Preview:**

##### Description
The `P2pSuperformProxy` contract enforces:
```solidity
require(
    req.superformData.receiverAddress == address(this),
    P2pSuperformProxy__ReceiverAddressShouldBeP2pSuperformProxy(
        req.superformData.receiverAddress
            )
        );
```

  1.  Superform may enter an emergency state during which the normal withdrawal flow is interrupted.
  2.  In such an emergency, calling withdraw may only enqueue an emergency withdrawal request without actually transferring assets, so the proxy’s balance may remain unchanged. If withdraw reverts when the balance delta is zero, it will break the emergency flow and block further recovery.
  3.  During this emergency flow, tokens can be sent by the Superform administrator to **receiverAddress** (the proxy itself) — a path that never occurs in normal operation — and without a rescue mechanism these assets will become permanently locked.

##### Recommendation
We recommend:
  1.  Ensure that withdraw never reverts when the proxy’s balance change is zero. Treat a zero-delta result as a successful no-op to preserve the emergency withdrawal queue flow.
  2.  Implement a client-accessible rescue mechanism to recover any ERC-20 or native tokens held by the proxy as a result of the emergency flow.

> **Client's Commentary:**
> Fixed in https://github.com/p2p-org/p2p-lending-proxy/commit/b7b2a4ff5b321afa7d9edaddf62953411eab8ff0

---

---
