# Cluster -1412

**Rank:** #221  
**Count:** 49  

## Label
Failure to accurately track or adjust token balances during deposits/withdrawals leads to incorrect pool allocations, reward misdistribution, and transaction deadlocks, undermining liquidity and incentive mechanisms.

## Cluster Information
- **Total Findings:** 49

## Examples

### Example 1

**Auto Label:** Failure to account for token transfer fees or dynamic token behaviors leads to incorrect balance tracking, causing deposits to fail, withdrawals to revert, or user funds to be lost due to unvalidated asset amounts.  

**Original Text Preview:**

The protocol will support stable tokens e.g.USDT, USDC. some of them are fee-on-transfer tokens.

However, The `deposit()` function in the `OmoRouter.sol` contract, Transfers tokens from the user to the router and then will be deposited into the vault.

```solidity
File: OmoRouter.sol#deposit()

         token.safeTransferFrom(msg.sender, address(this), assets);
         token.safeApprove(vault, assets);
         shares = IOmoVault(vault).deposit(assets, receiver);
```

The issue here is the `assets` value will be not available here.

This issue can be fixed as follows: checking the contract balance before and after, and passing the difference to the `deposit()` function.

---
### Example 2

**Auto Label:** Failure to account for token transfer fees or dynamic token behaviors leads to incorrect balance tracking, causing deposits to fail, withdrawals to revert, or user funds to be lost due to unvalidated asset amounts.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

- The `OmoRouter.deposit()` function doesn't correctly handle the `stETH` token, which is one of the LST tokens supported as an underlying asset in `OmoVaults`, the issue arises because `stETH` transfers **1-2 wei less than the amount specified during a `safeTransferFrom` operation**.

- This leads to the `OmoRouter` contract approving and attempting to deposit more tokens than it has received, causing the subsequent `OmoVault.deposit()` call to revert due to insufficient contract balance (received is less than deposited).

```javascript
 function deposit(
        uint256 vaultId,
        uint256 assets,
        address receiver
    ) external onlyWhitelisted returns (uint256 shares) {
        address vault = vaultFactory.getVault(vaultId);
        if (!isVaultEnabled[msg.sender][vault]) revert VaultNotEnabled();

        // Get the underlying token using IERC4626
        ERC20 token = ERC20(IERC4626(vault).asset());

        // Transfer tokens from user to router
        token.safeTransferFrom(msg.sender, address(this), assets);

        // Approve vault to spend tokens
        token.safeApprove(vault, assets);

        // Deposit into vault
        shares = IOmoVault(vault).deposit(assets, receiver);

        return shares;
    }
```

- The same issue exists in:
  - `OmoVault.deposit()` function when whitelisted users interact directly with the function.
  - `OmoVault.topOff()` and `OmoAgent.repayAsset()` functions.

## Recommendations

Update `OmoRouter.deposit()` to deposit the actual amount received:

```diff
 function deposit(
        uint256 vaultId,
        uint256 assets,
        address receiver
    ) external onlyWhitelisted returns (uint256 shares) {
        address vault = vaultFactory.getVault(vaultId);
        if (!isVaultEnabled[msg.sender][vault]) revert VaultNotEnabled();

        // Get the underlying token using IERC4626
        ERC20 token = ERC20(IERC4626(vault).asset());

        // Transfer tokens from user to router
+       uint256 balanceBefore = token.balanceOf(address(this));
        token.safeTransferFrom(msg.sender, address(this), assets);
+       uint256 amountToDeposit = token.balanceOf(address(this)) - balanceBefore;
        // Approve vault to spend tokens
-       token.safeApprove(vault, assets);
+       token.safeApprove(vault, amountToDeposit);

        // Deposit into vault
-       shares = IOmoVault(vault).deposit(assets, receiver);
+       shares = IOmoVault(vault).deposit(amountToDeposit, receiver);

        return shares;
    }
```

---
### Example 3

**Auto Label:** Failure to properly validate, transfer, or update token balances leads to incorrect reward calculations, reward loss, or reward monopolization, undermining yield accuracy and fair distribution.  

**Original Text Preview:**

##### Description

The `MasterChef` contract allows users to stake rewards, and based on their stake, the time they have been staking and the `XCN` rewards per second they are supposed to receive some amount of rewards. However there is no mechanism that guarantees that the full amount of `XCN` rewards will be deposited into the `MasterChef` contract, and stakers will be able to withdraw them.

##### BVSS

[AO:A/AC:L/AX:H/R:N/S:U/C:N/A:N/I:N/D:N/Y:C (3.3)](/bvss?q=AO:A/AC:L/AX:H/R:N/S:U/C:N/A:N/I:N/D:N/Y:C)

##### Recommendation

Consider implementing a mechanism that guarantees the `XCN` rewards are deposited into the `MasterChef` contract before they are accrued to stakers.

##### Remediation Comment

**RISK ACCEPTED:** The **Onyx team** has accepted the risk.

##### References

<https://etherscan.io/address/0x3fa642c0bbad64569eb8424af35f518347249216#code>

---
