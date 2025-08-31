# Cluster -1307

**Rank:** #290  
**Count:** 26  

## Label
Inconsistent access control logic leads to ambiguous or misapplied permissions, enabling unauthorized actions, undermining decentralization, and creating confusion in role management and policy enforcement.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

**Auto Label:** Unauthorized governance control through flawed access logic, improper address assignment, or inconsistent role definitions.  

**Original Text Preview:**

Grant Guardian is setting to address(0) as well, meaning it's also a revoke, `setGuardian` seems to be most appropriate

---
### Example 2

**Auto Label:** Inconsistent access control logic leads to ambiguous or misapplied permissions, enabling unauthorized actions, undermining decentralization, and creating confusion in role management and policy enforcement.  

**Original Text Preview:**

## Difficulty: Low

## Type: Undefined Behavior

### Description

The current implementation of the authorization mechanism is confusing, as it allows revoking individual writing permissions but only allows granting those permissions in bulk. As we can see in the figures below, the owner has the ability to revoke and grant writing permissions.

```solidity
function updateDao(address _pcsDao, address _pckDao, address _fmspcTcbDao, address _enclaveIdDao)
external
onlyOwner
{
    _updateDao(_pcsDao, _pckDao, _fmspcTcbDao, _enclaveIdDao);
}

function revokeDao(address revoked) external onlyOwner {
    _authorized_writers[revoked] = false;
}
```

Figure 3.1: `updateDAO`, `revokeDAO` functions in `AutomataDAOStorage.sol#L48-L57`

However, when granting permissions the owner can only do so in bulk (i.e. to all DAOs at the same time). This flow is confusing as if individual DAOs may have their permissions revoked then it should also be possible to grant them individually.

```solidity
function _updateDao(address _pcsDao, address _pckDao, address _fmspcTcbDao, address _enclaveIdDao) private {
    _authorized_writers[_pcsDao] = true;
    _authorized_writers[_pckDao] = true;
    _authorized_writers[_fmspcTcbDao] = true;
    _authorized_writers[_enclaveIdDao] = true;
}
```

Figure 3.2: `_updateDAO` function in `AutomataDAOStorage.sol#L96-L101`

What makes this confusing is that, in the case an individual DAO has its permissions revoked and a new DAO has to be incorporated, then both the current DAOs and the new one will be granted permissions (even if it is a no-op for the existing ones).

### Recommendations

- Short term: Make it possible to grant roles individually.
- Long term: Thoroughly document the intended behavior of the authorization flow.

---
### Example 3

**Auto Label:** Lack of role-based access control enables unauthorized actors to manipulate or drain funds, bypassing intended authorization and leading to financial loss or protocol failure.  

**Original Text Preview:**

## Security Report

## Severity: Low Risk

### Context
`ParaswapAdapter.sol#L55`

### Description
The `ParaswapAdapter` contract has several functions that are not protected by the `onlyBundler` modifier. Because of that, it's possible to trigger them from any caller. This creates a possibility of stealing users' funds from the adapter.

The purpose of the adapter is to receive funds, swap them, and then either transfer them or leave them in the contract. There are a few cases when funds are still in the adapter after the Paraswap swap:

- **Source token leftover.**
- **When the receiver of the swap is `ParaswapAdapter` itself** (see `ParaswapAdapter.sol#L171`).

In case any malicious contract gets a callback in the next step of bundle multicall after the swap, while funds are still in the `ParaswapAdapter`, it can swap them through the Paraswap and transfer them out of the `ParaswapAdapter`. As a result of this, when the victim expects to transfer tokens back from `ParaswapAdapter` (likely using `type(uint256).max` as amount to the `CoreAdapter.erc20Transfer()` call), the call won't revert as the attacker just leaves 1 wei to bypass the zero amount check (see `CoreAdapter.sol#L70`).

### Recommendation
It's recommended to apply the `onlyBundler` modifier to functions of `ParaswapAdapter`.

### Morpho
Fixed in PR 192. The scenario assumes the user calls a hostile contract after using Paraswap, but before skimming. It is the user’s responsibility not to use an adapter to call a hostile contract that can exploit that adapter’s funds or approvals. However, we have still added the `onlyBundler` modifier to help reasoning across all non-callback external adapter functions.

### Spearbit
Verified, the `onlyBundler` modifier has been added to `swap()`, which ensures all functions in the adapter can only be called by the bundler.

---
