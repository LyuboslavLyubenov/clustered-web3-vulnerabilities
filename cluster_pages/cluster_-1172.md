# Cluster -1172

**Rank:** #379  
**Count:** 14  

## Label
Inefficient account handling and misaligned instruction logic lead to unnecessary transaction costs, redundant state operations, and invalid instruction execution due to incorrect mutability or account requirements.

## Cluster Information
- **Total Findings:** 14

## Examples

### Example 1

**Auto Label:** **Improper account mutability and incorrect instruction ordering lead to state corruption, unauthorized access, and transaction failures.**  

**Original Text Preview:**

## GodMode Admin Account and Lamport Transfers

In GodMode, the admin account handles lamport transfers when resizing the staking account. However, the transfer CPI call in `add_ephemeral_multiplier` attempts to deduct lamports from the admin account, while `reclaim_rent` (shown below) transfers excess lamports back to it. Since the admin account is currently read-only, these operations will fail—preventing lamport deductions in `add_ephemeral_multiplier` and blocking modifications in `reclaim_rent`.

> _ claynosaurz-staking/src/instructions/admin/godmode.rs rust

```rust
/// Reclaims rent from unused space in the staking account.
pub fn reclaim_rent(ctx: Context<GodMode>) -> Result<()> {
    [...]
    // Verify if the data_len is less than the current data_len, if so, resize the account
    if account_info.lamports() > minimum_balance {
        **ctx.accounts.admin.to_account_info().try_borrow_mut_lamports()? +=
        ,→ account_info.lamports() - minimum_balance;
        **account_info.try_borrow_mut_lamports()? = minimum_balance;
    }
    Ok(())
}
```

## Remediation

Mark the admin account as `mut`.

### Patch

Fixed in `0b87d6e`.

---
### Example 2

**Auto Label:** Inefficient account handling and misaligned instruction logic lead to unnecessary transaction costs, redundant state operations, and invalid instruction execution due to incorrect mutability or account requirements.  

**Original Text Preview:**

##### Description

Some inconsistencies have been found in Huma Protocol Spec for Solana document:

* In the `Introduction` section: "*Institutional investor participation. The most critical thing for institutional investors is principal safety. This makes tranche support essential so that* ***intentional investors can choose to participate in senior tranches only***." However, investor can choose to participate in a senior tranche only, but if the pool has the senior tranche (in addition to the juniot tranche, since there is no pool with only a senior tranche)
* In the `Pool-level User Roles` , section (3.1.2 ) *"Pool Owners:* ***Pool owners are a list of addresses that are approved by the Protocol Owner*** *to create and manage pools.".* However, a pool can be created by anyone.
* In the `Credit Approval` , section (5.1.2) : " *For example, assuming the approved credit is 1000, the borrower has borrowed 500, and the borrower pays back 400.* ***If it is not revolving, the remaining credit is only 500*."** However, this is Inaccurate/confusing revolving description. The non-revolving credit borrowers can drawdown only once and not multiple times which is not explicitly stated in the docs.
* In the `Credit Approval` , section (5.1.2) : "***The credit limit cannot exceed 4 billion of the coin (stablecoin) supported by the pool.***" This requirement is not relevant and should be removed.
* In the `Redemption Request and Cancellation`, section (4.2.1)"only requests that meet lockup period requirements will be accepted. Redemption requests can be canceled **before the epoch starts to process the requests** at no cost." This can be not accurate and can be confusing, as a non processed redemption request that has been rolled up to the next epoch can also be cancelled at no cost.

Other inconsistencies have been found in [lib.rs](http://lib.rs) file in the program:

In `add_pool_operator` the comments mention the huma owner can call this instruction. However, only the pool owner can add operators.

[***lib.rs***](http://deposit.rs)

```
 /// # Access Control
    /// Only the pool owner and the Huma owner can call this instruction.
    pub fn add_pool_operator(ctx: Context<AddPoolOperator>, operator: Pubkey) -> Result<()> {
        pool::add_pool_operator(ctx, operator)
    }
```

In `remove_pool_operator` the comments mention the huma owner can call this instruction. However, only the pool owner can add operators.

```
 /// # Access Control
    /// Only the pool owner and the Huma owner can call this instruction.
    pub fn remove_pool_operator(ctx: Context<RemovePoolOperator>, operator: Pubkey) -> Result<()> {
        pool::remove_pool_operator(ctx, operator)
```

In `start_committed_credit` the comments the mention the pool owner and sentinel can call this instruction. However, it is the Evaluation Agent and the sentinel who can call this instruction.

```
/// # Access Control
    /// Only the pool owner and the Sentinel Service account can call this instruction.
    pub fn start_committed_credit(ctx: Context<StartCommittedCredit>) -> Result<()> {
        credit::start_committed_credit(ctx)
    }
```

##### Score

Impact:   
Likelihood:

##### Recommendation

It is recommended to update the documentation to remove all the inconsistencies.

##### Remediation

**PARTIALLY SOLVED:** The **Huma team** partially solved this issue. They updated the [lib.rs](http://lib.rs) file with the appropriate modifications correcting the mentioned inconsistent comments.

##### Remediation Hash

<https://github.com/00labs/huma-solana-programs/pull/98>

---
### Example 3

**Auto Label:** Inefficient account handling and misaligned instruction logic lead to unnecessary transaction costs, redundant state operations, and invalid instruction execution due to incorrect mutability or account requirements.  

**Original Text Preview:**

##### Description

The instructions `ManageCredit`, `ManageCreditConfig`, `TriggerDefault`, `RefreshCredit`, `StartCommittedCredit` required passing also the `SystemProgram` account. However this account is not used in the program and is therefore superfluous.

Passing the `SystemProgram` account only increases transaction costs.

***manage\_credit.rs***

```
pub system_program: Program<'info, System>,
```

```
pub system_program: Program<'info, System>,
```

***manage\_credit\_config.rs***

```
pub system_program: Program<'info, System>,
```

***refresh\_credit.rs***

```
pub system_program: Program<'info, System>,
```

***start\_committed\_credit.rs***

```
pub system_program: Program<'info, System>,
```

##### Score

Impact:   
Likelihood:

##### Recommendation

To address this issue, it is recommended to remove all superfluous accounts.

##### Remediation

**SOLVED:** The **Huma team** solved this issue by removing all superfluous accounts.

##### Remediation Hash

<https://github.com/00labs/huma-solana-programs/pull/105>

---
