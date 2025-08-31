# Cluster -1033

**Rank:** #61  
**Count:** 281  

## Label
Inaccurate asset or share calculations due to flawed mathematical modeling or improper handling of fees and supply states, leading to misaligned user incentives, value leakage, and compromised economic behavior.

## Cluster Information
- **Total Findings:** 281

## Examples

### Example 1

**Auto Label:** Inaccurate asset valuation due to flawed state tracking, missing updates, or improper aggregation—leading to overvaluation, underestimation, or misrepresentation of true asset holdings.  

**Original Text Preview:**

The `FoliotoAssets()` function includes balances from the `activeTrustedFill` contract. If an auction swap is in progress, this may include both the buy token and/or the sell token, depending on whether the swap is partially or fully completed. As a result, the returned asset amounts for a given share amount may become inaccurate and fluctuate based on the current state of the active filler. This could mislead consuming protocols or users relying on this **view function** for precise information.

**Recommendation:**

Consider implementing a mechanism to make `toAssets()` callable only when there is no active filler (e.g. `require(address(activeTrustedFill) == address(0))`), or clearly document in the function comment that asset amounts may be inaccurate when a filler is active due to inclusion of unsettled filler balances.

---
### Example 2

**Auto Label:** Inaccurate asset or share calculations due to flawed mathematical modeling or improper handling of fees and supply states, leading to misaligned user incentives, value leakage, and compromised economic behavior.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

The function `rebase()` is supposed to apply daily APR to the share price by decreasing the total share amount. The issue is that the code uses `totalSharesAmount * apr` to calculate `burnAmount` so the burned amount would be bigger than what it should be. This is the POC:

1. Suppose there are 100 shares and 100 tokens.
2. Admin wants to apply a 20% increase for one day.
3. Code would calculate the burn amount as `100 * 20% = 20` and the new total share would be 80.
4. Now the token to share price would be `100/80 = 1.25` and as you can see the ratio increases by 25%.
5. This would cause a wrong reward rate and those who withdraw sooner would steal others tokens.

## Recommendations

Code should use `totalSharesAmount * apr / (1 +apr)` to calculate the burn amount.

---
### Example 3

**Auto Label:** Inaccurate asset or share calculations due to flawed mathematical modeling or improper handling of fees and supply states, leading to misaligned user incentives, value leakage, and compromised economic behavior.  

**Original Text Preview:**

## Severity

**Impact:** High, because the accounting will be incorrect, and the sharePrice will be affected

**Likelihood:** Low, because fee on transfer token is not commonly used

## Description

`mint()/deposit()` is using `amount` for transfering and accounting. But fee on transfer token could break the accounting, since the actual token received will be less than amount. As a result, `sharePrice` will have some small error each time.

```solidity
File: src\abstract\As4626.sol
69:     function mint(
70:         uint256 _shares,
71:         address _receiver
72:     ) public returns (uint256 assets) {
73:         return _deposit(previewMint(_shares), _shares, _receiver);
74:     }

117:     function deposit(
118:         uint256 _amount,
119:         address _receiver
120:     ) public whenNotPaused returns (uint256 shares) {
121:         return _deposit(_amount, previewDeposit(_amount), _receiver);
122:     }

84:     function _deposit(
85:         uint256 _amount,
86:         uint256 _shares,
87:         address _receiver
88:     ) internal nonReentrant returns (uint256) {

98:         asset.safeTransferFrom(msg.sender, address(this), _amount);

105:         _mint(_receiver, _shares);

```

USDT potentially could turn on fee on transfer feature, but not yet.

## Recommendations

Use before and after balance to accurately reflect the true amount received, and update share price accordingly.

---
