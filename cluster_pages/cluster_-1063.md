# Cluster -1063

**Rank:** #66  
**Count:** 265  

## Label
Insufficient input validation and state transition flaws enable attackers to exploit financial invariants, cause unintended portfolio exposure, or manipulate auction outcomes through bid re-execution, zero-value bids, or unbounded challenge cycles.

## Cluster Information
- **Total Findings:** 265

## Examples

### Example 1

**Auto Label:** Insufficient input validation leads to unauthorized operations, incorrect routing, or unintended token transfers, enabling denial-of-service, fund loss, or malicious redirection.  

**Original Text Preview:**

The [`SwapProxy` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L26) contains the [`performSwap` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L67), which allows the caller to execute a swap in two ways: [by approving or sending tokens to the specified exchange](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L81-L84), or by [approving tokens through the `Permit2` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L86-L101). However, since it is possible to supply any address as the `exchange` parameter and any call data through the `routerCalldata` parameter of the `performSwap` function, the `SwapProxy` contract may be forced to perform an [arbitrary call to an arbitrary address](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L108).

This could be exploited by an attacker, who could force the `SwapProxy` contract to call the [`invalidateNonces` function](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L113) of the `Permit2` contract, specifying an arbitrary spender and a nonce higher than the current one. As a result, the nonce for the given (token, spender) pair will be [updated](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L124). If the `performSwap` function is called again later, it [will attempt to use a subsequent nonce](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L95), which has been invalidated by the attacker and the code inside `Permit2` will [revert due to nonces mismatch](https://github.com/Uniswap/permit2/blob/cc56ad0f3439c502c246fc5cfcc3db92bb8b7219/src/AllowanceTransfer.sol#L138).

As the `performSwap` function is the only place where the nonce passed to the `Permit2` contract is updated, the possibility of swapping a given token on a certain exchange will be blocked forever, which impacts all the functions of the [`SpokePoolPeriphery` contract](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L140) related to swapping tokens. The attack may be performed for many different (tokens, exchange) pairs.

Consider not allowing the `exchange` parameter to be equal to the `Permit2` contract address.

***Update:** Resolved in [pull request #1016](https://github.com/across-protocol/contracts/pull/1016) at commit [`713e76b`](https://github.com/across-protocol/contracts/pull/1016/commits/713e76b8388d90b4c3fbbe3d16b531d3ef81c722).*

---
### Example 2

**Auto Label:** Insufficient input validation leads to unauthorized operations, incorrect routing, or unintended token transfers, enabling denial-of-service, fund loss, or malicious redirection.  

**Original Text Preview:**

`swapActive()` is called by critical flows:

1. `stateChangeActive()` --> `swapActive()`.
2. `sync()` --> `_poke()` --> `_closeTrustedFill()` --> `closeFiller()` --> `swapActive()`.

**Case1:**
External integrating protocol calls `stateChangeActive()` and wants to see `(false, false)` before trusting the Folio state. Suppose:

- `sellAmount` is `100`.
- `sellTokenBalance` is `98` i.e. swap is still active and only partial sell has occurred.
- Ideally external protocol should see `(false, true)`.
- Attack scenario: Attacker front runs the call to `stateChangeActive()` and donates `2` sellTokens.
- Now, inside the `swapActive()` function, `if (sellTokenBalance >= sellAmount)` evaluates to `true`, and swap is reported as completed i.e. `false` is returned.
- External protocol trusts an inconsistent state & balance.

Note that a similar attack can be mounted by donating buyTokens which would result in `minimumExpectedIn > buyToken.balanceOf(address(this))` to return `false` instead of `true`.

**Case2:**
Ideally all `sync()` modifier protected functions should revert if swaps are active because `closeFiller()` checks that:

```js
require(!swapActive(), BaseTrustedFiller__SwapActive());
```

Almost all the critical Folio functions like `mint()`, `redeem()`, `distributeFees()` etc are protected by `sync()`. But due to the same attack path as above, protocol can be fooled into believing that the swap has concluded. This allows the function calls to proceed and potentially end up in an inconsistent state.

**Recommendations:**

Track balance internally through accounting variables instead of using `balanceOf()` inside `swapActive()`.

---
### Example 3

**Auto Label:** **Insufficient input validation and improper execution ordering lead to balance manipulation, pricing distortions, and economic exploitation of transaction sequences.**  

**Original Text Preview:**

## Severity

**Impact:** High, as the victim loses their funds

**Likelihood:** Low, as it comes at a cost for the attacker

## Description

The famous initial deposit attack is largely mitigated by the `+1` done in the asset/shares conversion. However, doing this attack can cause some strange behavior that could grief users (at high cost of the attacker) and leave the vault in a weird state:

Here's a PoC showing the impacts, can be added to `Deposit.t.sol`:

```solidity
    Vault vault;
    MockERC20 asset;

    address bob = makeAddr('bob');

    function setUp() public {
        asset = new MockERC20();
        vault = new Vault(100e18,100e18,asset);

        asset.mint(address(this), 10e18 + 9);
        asset.mint(bob,1e18);
    }

    function test_initialSupplyManipulation() public {
        // mint a small number of shares
        // (9 + 1 = 10) makes math simpler
        asset.approve(address(vault),9);
        vault.deposit(9, address(this));

        // do a large donation
        asset.transfer(address(vault), 10e18);

        // shares per assets is now manipulated
        assertEq(1e18+1,vault.convertToAssets(1));

        // victim stakes in vault
        vm.startPrank(bob);
        asset.approve(address(vault), 1e18);
        vault.deposit(1e18, bob);
        vm.stopPrank();

        // due to manipulation they receive 0 shares
        assertEq(0,vault.balanceOf(bob));

        // attacker redeems their shares
        vault.redeem(vault.balanceOf(address(this)), address(this), address(this));

        // even though the attacker loses 0.1 tokens
        assertEq(9.9e18 + 9,asset.balanceOf(address(this)));
        // the vicims tokens are lost and locked in the contract
        assertEq(1.1e18,asset.balanceOf(address(vault)));
    }
```

As you can see the attacker needs to pay `0.1e18` of assets for the attack. But they have effectively locked the victims `1e18` tokens in the contract.

Even though this is not profitable for the attacker it will leave the vault in a weird state and the victim will still have lost their tokens.

## Recommendations

Consider mitigating this with an initial deposit of a small amount. This is the most common and easy way to make sure this is not possible, as long as it is an substantial amount it will make this attack too costly.

---
