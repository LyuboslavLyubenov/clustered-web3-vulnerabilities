# Cluster -1131

**Rank:** #56  
**Count:** 304  

## Label
Insecure token approval mechanisms leading to unauthorized transfers, silent failures, or denial-of-service due to improper allowance handling and lack of validation.

## Cluster Information
- **Total Findings:** 304

## Examples

### Example 1

**Auto Label:** Insecure token approval mechanisms leading to unauthorized transfers, silent failures, or denial-of-service due to improper allowance handling and lack of validation.  

**Original Text Preview:**

## Severity

**Impact**: Medium, because functionality won't work

**Likelihood**: Medium, because USDT is a common token

## Description

Code uses the `approve` method to set allowance for ERC20 tokens in `setSwapperAllowance`. This will cause revert if the target ERC20 was a non-standard token that has different function signature for `approve()` function. Tokens like USDT will cause revert for this function, so they can't be used as reward token, input token and underlying asset.

```solidity
    function setSwapperAllowance(uint256 _amount) public onlyAdmin {
        address swapperAddress = address(swapper);

        for (uint256 i = 0; i < rewardLength; i++) {
            if (rewardTokens[i] == address(0)) break;
            IERC20Metadata(rewardTokens[i]).approve(swapperAddress, _amount);
        }
        for (uint256 i = 0; i < inputLength; i++) {
            if (address(inputs[i]) == address(0)) break;
            inputs[i].approve(swapperAddress, _amount);
        }
        asset.approve(swapperAddress, _amount);
    }
```

## Recommendations

Use `SafeERC20`'s `forceApprove` method instead to support all the ERC20 tokens.

---
### Example 2

**Auto Label:** Insecure token approval mechanisms leading to unauthorized transfers, silent failures, or denial-of-service due to improper allowance handling and lack of validation.  

**Original Text Preview:**

**Description:** Use [`SafeERC20::forceApprove`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/utils/SafeERC20.sol#L101-L108) when dealing with a range of potential tokens instead of standard `IERC20::approve`:
```solidity
predeposit/yUSDeDepositor.sol
58:        pUSDe.approve(address(yUSDe), amount);

predeposit/pUSDeVault.sol
178:        USDe.approve(address(sUSDe), USDeAssets);

predeposit/pUSDeDepositor.sol
86:            asset.approve(address(vault), amount);
98:        sUSDe.approve(address(pUSDe), amount);
110:        USDe.approve(address(pUSDe), amount);
122:        token.approve(swapInfo.router, amount);
```

**Strata:** Fixed in commit [f258bdc](https://github.com/Strata-Money/contracts/commit/f258bdcc49b87a2f8658b150bc3e3597a5187816).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Insecure token approval mechanisms leading to unauthorized transfers, silent failures, or denial-of-service due to improper allowance handling and lack of validation.  

**Original Text Preview:**

## Vulnerability Report

**Severity:** Medium Risk  
**Context:** BaseVault.sol#L316-L322  

**Description:**  
The approvals list is appended whenever `IERC20.approve.selector` is called. However, for some tokens, it is also possible to change the allowance using `increaseAllowance` (and potentially also `decreaseAllowance`).

**Area:** Fixed in PR 219.  

**Spearbit:** Verified. The code now detects calls to `approve()` and `increaseAllowance()` to track approvals.

---
