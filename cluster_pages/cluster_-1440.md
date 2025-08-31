# Cluster -1440

**Rank:** #436  
**Count:** 9  

## Label
Misuse of data or state references leading to incorrect calculations, flawed validations, or erroneous state transitions, resulting in financial misrepresentation, over-distribution, or transaction failures.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Misuse of data or state references leading to incorrect calculations, flawed validations, or erroneous state transitions, resulting in financial misrepresentation, over-distribution, or transaction failures.  

**Original Text Preview:**

## Token Program Utilization in `kamino_lend_standarad::RedeemSy`

In `kamino_lend_standarad::RedeemSy`, `withdraw_obligation_collateral_ctx` utilizes the `token_program_base` account for both the `liquidity_token_program` and the `collateral_token_program`. This may result in problems when the two token programs (liquidity and collateral) are different. Specifically, suppose one token program is the standard SPL Token program, and the other is the token-2022 program. In that case, it may result in failures during execution as utilizing the same `token_program_base` for both operations will result in a mismatch.

## Example Code

> _ kamino_lend_standard/src/instructions/redeem_sy.rs rust

```rust
fn withdraw_obligation_collateral_ctx(
    &self,
) -> CpiContext<'_, '_, '_, 'info,
    WithdrawObligationCollateralAndRedeemReserveCollateral<'info>>
{
    let accs = WithdrawObligationCollateralAndRedeemReserveCollateral {
        lending_market: self.lending_market.to_account_info(),
        collateral_token_program: self.token_program_base.to_account_info(),
        obligation: self.kamino_obligation.to_account_info(),
        lending_market_authority: self.lending_market_authority.to_account_info(),
        owner: self.authority_klend_account.to_account_info(),
        liquidity_token_program: self.token_program_base.to_account_info(),
        [...],
    };
    CpiContext::new(self.kamino_lend_program.to_account_info(), accs)
}
```

## Remediation

Distinguish between the token programs utilized for liquidity and collateral. Instead of utilizing a single `token_program_base`, the function should explicitly specify the appropriate token programs for each operation.

## Patch

Resolved in PR#611.

---
### Example 2

**Auto Label:** Misuse of data or state references leading to incorrect calculations, flawed validations, or erroneous state transitions, resulting in financial misrepresentation, over-distribution, or transaction failures.  

**Original Text Preview:**

## Severity

**Impact:** Low

**Likelihood:** High

## Description

When users `unwrap` their wstUSR, the underlying `stUSRAmount` is returned from the operation :

```solidity
    function unwrap(uint256 _wstUSRAmount, address _receiver) public returns (uint256 stUSRAmount) {
        _assertNonZero(_wstUSRAmount);

        IERC20Rebasing stUSR = IERC20Rebasing(stUSRAddress);

>>>     stUSRAmount = stUSR.convertToUnderlyingToken(_wstUSRAmount);
        _burn(msg.sender, _wstUSRAmount);
        IERC20(stUSRAddress).safeTransfer(_receiver, stUSRAmount);

        emit Unwrap(msg.sender, _receiver, stUSRAmount, _wstUSRAmount);

>>>     return stUSRAmount;
    }
```

However, the returned `stUSRAmount` value is inaccurate. This issue occurs because of additional rounding down that happens when transferring `stUSR` from the `wstUSR` contract, and when calculating the user's underlying balances based on the transferred shares.

Now, when users call `stUSR.withdraw` using the returned value, the call will revert due to insufficient balance.

```solidity
    function withdraw(uint256 _usrAmount, address _receiver) public {
        // @audit - if providing the returned stUSRAmount, the call will revert due to insufficient balance
        uint256 shares = previewWithdraw(_usrAmount);
>>>     super._burn(msg.sender, shares);

        IERC20Metadata usr = super.underlyingToken();
        usr.safeTransfer(_receiver, _usrAmount);
        emit Withdraw(msg.sender, _receiver, _usrAmount, shares);
    }
```

PoC :

```javascript
it("PoC - unwrap and withdraw revert", async () => {
  // given
  const { stUSR, wstUSR, USR, otherAccount2 } = await loadFixture(
    deployFixture
  );
  // Initial deposit
  await USR["mint(address,uint256)"](otherAccount2.getAddress(), parseUSR(100));
  const wstUSRFromAnotherSigner2 = wstUSR.connect(otherAccount2);
  const usrFromAnotherSigner2 = USR.connect(otherAccount2);
  const stUSRFromAnotherSigner2 = stUSR.connect(otherAccount2);
  await usrFromAnotherSigner2.approve(wstUSR.getAddress(), parseUSR(100));
  await wstUSRFromAnotherSigner2["deposit(uint256)"](parseUSR(100));
  // Mint USR to StUSR as a reward
  await USR["mint(address,uint256)"](stUSR.getAddress(), parseUSR(100));

  const withdrawAmount = parseUSR(100);
  const expectedWstUSR = await wstUSR.previewWithdraw(withdrawAmount);
  const expectedUsrInShares = await stUSR.convertToUnderlyingToken(
    expectedWstUSR
  );
  const balancewstBefore = await wstUSR.balanceOf(otherAccount2);
  console.log("balance wstUSR before withdraw : ");
  console.log(balancewstBefore);
  const sharestUSRBefore = await stUSR.sharesOf(otherAccount2);
  console.log("share stUSR before unwrap : ");
  console.log(sharestUSRBefore);
  // when
  const tx = await wstUSRFromAnotherSigner2["unwrap(uint256,address)"](
    expectedWstUSR,
    otherAccount2
  );
  const sharestUSRAfter = await stUSR.sharesOf(otherAccount2);
  console.log("share stUSR after unwrap : ");
  console.log(sharestUSRAfter);
  const tx2 = stUSRFromAnotherSigner2["withdraw(uint256)"](expectedUsrInShares);
  // then
  await expect(tx2)
    .to.revertedWithCustomError(stUSR, "ERC20InsufficientBalance")
    .withArgs(otherAccount2, expectedWstUSR - BigInt(1), expectedWstUSR);
});
```

## Recommendations

Update `stUSRAmount` by querying the user's balance from `stUSR`.

```diff
    function unwrap(uint256 _wstUSRAmount, address _receiver) public returns (uint256 stUSRAmount) {
        _assertNonZero(_wstUSRAmount);

        IERC20Rebasing stUSR = IERC20Rebasing(stUSRAddress);

        stUSRAmount = stUSR.convertToUnderlyingToken(_wstUSRAmount);
+      uint256 balanceBefore = stUSR.balanceOf(_receiver);
        _burn(msg.sender, _wstUSRAmount);
        IERC20(stUSRAddress).safeTransfer(_receiver, stUSRAmount);

+       stUSRAmount = stUSR.balanceOf(_receiver) - balanceBefore;

        emit Unwrap(msg.sender, _receiver, stUSRAmount, _wstUSRAmount);

        return stUSRAmount;
    }
```

---
### Example 3

**Auto Label:** Misuse of data or state references leading to incorrect calculations, flawed validations, or erroneous state transitions, resulting in financial misrepresentation, over-distribution, or transaction failures.  

**Original Text Preview:**

**Severity**: High

**Status**: Resolved

**Description**

In the `withdrawAirdropTokens()` and `getCurrentAirdropBalance()` functions, `tokensToWithdraw` is assumed to be calculated by multiplying `airdropTokenPerDay` by the number of days passed since the last withdrawal. But it's calculated by multiplying `airdropTokenPerDay` by the number of minutes passed.
It can result in larger airdrops than expected.

**Recommendation**: 

Calculate the correct number of elapsed days and multiply it by `airdropTokensPerDay` to calculate the `tokensToWithdraw`.

---
