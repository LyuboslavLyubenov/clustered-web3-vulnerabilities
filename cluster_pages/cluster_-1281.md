# Cluster -1281

**Rank:** #328  
**Count:** 21  

## Label
Redundant or misaligned inheritance leading to unnecessary access controls, code bloat, and potential security misconfigurations due to unused or inconsistent code paths.

## Cluster Information
- **Total Findings:** 21

## Examples

### Example 1

**Auto Label:** Redundant and conflicting access control due to improper inheritance, leading to ambiguous privileges, increased complexity, and potential security risks.  

**Original Text Preview:**

In the `createDaoErc20` function within the `Emitter` contract, there is a line that grants the `EMITTER` role to `msg.sender`:

```solidity
File: emitter.sol
161:     function createDaoErc20(
162:         address _deployerAddress,
163:         address _proxy,
164:         string memory _name,
165:         string memory _symbol,
166:         uint256 _distributionAmount,
167:         uint256 _pricePerToken,
168:         uint256 _minDeposit,
169:         uint256 _maxDeposit,
170:         uint256 _ownerFee,
171:         uint256 _totalDays,
172:         uint256 _quorum,
173:         uint256 _threshold,
174:         address _depositTokenAddress,
175:         address _emitter,
176:         address _gnosisAddress,
177:         address lzImpl,
178:         bool _isGovernanceActive,
179:         bool isTransferable,
180:         bool assetsStoredOnGnosis
181:     ) external payable onlyRole(FACTORY) {
182:         _grantRole(EMITTER, _proxy);
183:>>>      _grantRole(EMITTER, msg.sender);
```

Given that the function is restricted by the `onlyRole(FACTORY)` modifier, `msg.sender` will always be the `FACTORY` contract. Since the `FACTORY` contract already has the `EMITTER` role, this operation is redundant. Continually executing this line on every call to `createDaoErc20` results in unnecessary gas consumption.

Eliminate the line that grants the `EMITTER` role to `msg.sender` within the `createDaoErc20` function. This will reduce unnecessary gas usage and simplify the role management logic.

---
### Example 2

**Auto Label:** Redundant access control checks due to lack of abstraction, leading to inconsistent logic, increased attack surface, and higher risk of bugs and vulnerabilities.  

**Original Text Preview:**

##### Description
`checkTransaction()`: simplify logical conditions https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/borgCore.sol#L155 to `if (!policy[to].allowed)`, https://github.com/MetaLex-Tech/borg-core/blob/9074503d37cfa1d777ef16f6c88b84c98b4f54eb/src/borgCore.sol#L158 to `if (policy[to].fullAccess)`.

##### Recommendation
We recommend making improvements from the description.

---
### Example 3

**Auto Label:** Redundant and conflicting access control due to improper inheritance, leading to ambiguous privileges, increased complexity, and potential security risks.  

**Original Text Preview:**

##### Description

The SolidlySynthChef and other contracts currently inherits from both `OwnableUpgradeable` and `AccessControlUpgradeable`. This creates unnecessary complexity and potential confusion in the access control system. We can simplify the contract by removing `OwnableUpgradeable` and solely relying on `AccessControlUpgradeable` with the `DEFAULT_ADMIN_ROLE`.

  

```
abstract contract SolidlySynthChef is
    IProtocolSynthChef,
    Initializable,
    UUPSUpgradeable,
    AccessControlUpgradeable,
    OwnableUpgradeable,
    ReentrancyGuardUpgradeable
{
    // ...
}
```

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

1. Remove `OwnableUpgradeable` from the inheritance list.

2. Replace `onlyOwner` modifier with `onlyRole(DEFAULT_ADMIN_ROLE).`

3. Remove `transferOwnership` call in the `initialize` function.

4. Update the `_authorizeUpgrade` function to use `DEFAULT_ADMIN_ROLE.`

  

### Remediation Plan

**ACKNOWLEDGED :** The **Entangle team** acknowledged the issue**.**

---
