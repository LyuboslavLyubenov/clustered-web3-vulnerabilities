# Cluster -1050

**Rank:** #223  
**Count:** 47  

## Label
Inconsistent token accounting and flawed liquidity validation lead to visibility gaps, unintended constraints, and yield erosion, enabling users to hide assets, exploit fee mechanics, or destabilize pool invariants.

## Cluster Information
- **Total Findings:** 47

## Examples

### Example 1

**Auto Label:** Uncontrolled LP token burning leads to pool state corruption, price manipulation, and liquidity erosion through unauthorized supply manipulation and flawed validation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The `LPToken` contract allows unrestricted burning of LP tokens via `burn()` and `burnFrom()`. This enables an attack where the first depositor manipulates the LP price by burning nearly all of their LP tokens. This skews the pool state such that any subsequent deposit may mint **less or 0 LP tokens**, either causing a denial of service (if slippage checks are used) or silent fund loss (if not).

```solidity
function _calcAddLiquidity() internal view returns (uint256 netAmount, uint256 pFee, uint256 lpAmount) {
    --- SNIPPED ---
    if (lpSupply == 0) {
        lpAmount = netAmount * tokenPrice;
        lpAmount = lpAmount.toDecimals(tokenDecimals + 8, plpDecimals);
    } else {
@1>     lpAmount = netAmount.mulDiv(tokenPrice * lpSupply, _getPoolValue(false, false) /*18, USD*/);
@2>     lpAmount = lpAmount.toDecimals(tokenDecimals + 8, plpDecimals);
    }
}
```

```solidity
//> src/utils/tokens/lp/LPToken.sol
function burn(uint256 value) public override(ILPToken, ERC20Burnable) {
    ERC20Burnable.burn(value);
}

function burnFrom(address _account, uint256 _amount) public override(ILPToken, ERC20Burnable) {
    ERC20Burnable.burnFrom(_account, _amount);
}
```

Consider the following scenario: 0. Initial states: - ETH Oracle price: 1500e8 (8 decimals, 1500 USD). - USDC Oracle price: 1e8 (8 decimals, 1 USD). - LP token decimals: 18. - 0% fee.

1. Bob attempts to add 1 ETH liquidity, which is equivalent to 1500 USD.
2. Alice front-runs Bob's deposit with 1501 USDC, which is equivalent to 1501 USD:
   - LP mint amount: `1501e6 * 1e8 = 1501e14` -- to 18 decimals --> `1501e14 * 1e(18-14) = 1501e18 LPs`.
   - PoolValue: 1501e18 ≈ 1501 USD.
3. Alice burns all but 1 wei of LP tokens, so now 1 LP is worth 1501 USD.
4. Bob's transaction gets executed:
   - PoolValue: 1501e18 ≈ 1501 USD.
   - LP mint amount: `(1e18 * 1500e8) / 1501e18 = 99999999` -- to 18 decimals --> `99999999 / 1e(26-18) = 99999999 / 1e8 = 0 (rounding down)`.
   - Due to rounding down in `lpAmount` calculation, it causes the `lpAmount` to be 0.
5. Bob also specifies the slippage `_minLpAmount`. This attack will create a DoS if `_minLpAmount` is specified. If not, this attack will steal Bob's deposits.

## Recommendation

Restrict burning to protocol roles: Pool contract for removing liquidity or disallowing public `burn()` and `burnFrom()` if `lpSupply` is less than the minimum LP supply.

---
### Example 2

**Auto Label:** Supply manipulation through unauthorized or flawed burn mechanisms, leading to incorrect allocations, financial losses, and violated invariant assumptions.  

**Original Text Preview:**

The `LPToken.burn()` function burns a specified amount of LP tokens and redeems the underlying assets:

```solidity
    function burn(address account, uint256 shares) external {
        _spendAllowance(account, _msgSender(), shares);
        burveMulti.removeLiq(_msgSender(), ClosureId.unwrap(cid), shares);
        _burn(account, shares);
    }
```

Currently, the function deducts the allowance from `account` to `msg.sender` even when `account == msg.sender`. This means users must approve themselves before redeeming their tokens, leading to unnecessary gas costs and an inefficient user experience.

Consider skipping the allowance check when `account == msg.sender`:

```solidity
if (account != _msgSender()) {
    _spendAllowance(account, _msgSender(), shares);
}
```

---
### Example 3

**Auto Label:** Supply manipulation through unauthorized or flawed burn mechanisms, leading to incorrect allocations, financial losses, and violated invariant assumptions.  

**Original Text Preview:**

## 01. Relevant GitHub Links

&#x20;

* [RAACNFT.sol#L75](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/tokens/RAACNFT.sol#L75)

## 02. Summary

The website mentions that users can burn NFTs to redeem assets. However, the contract code does not provide a burn function. Instead, `_update` prevents sending NFTs to the zero address, effectively blocking any potential burning operation.

## 03. Vulnerability Details

The site  explains that you can burn NFT tokens and actually receive assets for them.

> Real estate is held in a corporate structure that is designed to protect Regna Minima NFT holders. Eligible users can burn the Regna Minima NFT to redeem the real estate title.

However, the actual NFT implementation not only doesn't have a burn function, but it also overrides the `_update` function, making it impossible to burn.

```Solidity
function _update(address to, uint256 tokenId, address auth) internal override(ERC721, ERC721Enumerable) returns (address) {
    if (to == address(0)) revert RAACNFT__InvalidAddress();
    return super._update(to, tokenId, auth);
}

```

## 04. Impact

Users cannot burn their NFTs to redeem real estate or other assets, contrary to the documentation and advertised functionality.

## 05. Tools Used

Manual Code Review and Foundry

## 06. Recommended Mitigation

Implement a burn function andadd a mitigation to prevent the mint function from re-minting tokens that have already been burned.

---
