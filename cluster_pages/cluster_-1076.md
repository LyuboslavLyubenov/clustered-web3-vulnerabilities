# Cluster -1076

**Rank:** #286  
**Count:** 26  

## Label
Failure to validate or derive Program-Defined Addresses (PDAs) securely, enabling attackers to forge or manipulate account identities, leading to unauthorized access, state tampering, or fund theft.

## Cluster Information
- **Total Findings:** 26

## Examples

### Example 1

**Auto Label:** Failure to validate or derive Program-Defined Addresses (PDAs) securely, enabling attackers to forge or manipulate account identities, leading to unauthorized access, state tampering, or fund theft.  

**Original Text Preview:**

A deposit action to the SpokePool program is meant to [pull tokens from the caller](https://github.com/across-protocol/contracts/blob/71e990b3f908ecf994be8723e041b584c7d49318/programs/svm-spoke/src/lib.rs#L213) (i.e., the [signer](https://github.com/across-protocol/contracts/blob/71e990b3f908ecf994be8723e041b584c7d49318/programs/svm-spoke/src/instructions/deposit.rs#L30)) which may be deposited on behalf of any `depositor` account. However, when the signer is not the depositor, a `transfer_from` operation is performed with the [depositor token account as the source address](https://github.com/across-protocol/contracts/blob/71e990b3f908ecf994be8723e041b584c7d49318/programs/svm-spoke/src/instructions/deposit.rs#L105). Such an operation would fail unless the depositor had delegated at least the input amount to the state PDA. This presents two consequences:

1. The signer will not be able to deposit for another account, disabling the intended feature of an [account being able to deposit on behalf of someone else](https://github.com/across-protocol/contracts/blob/71e990b3f908ecf994be8723e041b584c7d49318/programs/svm-spoke/src/lib.rs#L210).
2. If any token account delegates to the state PDA, then it is possible for anyone to call the deposit function, passing the victim's account as the [`depositor_token_account`](https://github.com/across-protocol/contracts/blob/71e990b3f908ecf994be8723e041b584c7d49318/programs/svm-spoke/src/instructions/deposit.rs#L45) and performing a successful deposit with arbitrary arguments other than the input token and amount. A malicious user could then submit a deposit with their own recipient address to steal depositor funds. This is possible due to the state PDA being passed as the [authority and signer seed](https://github.com/across-protocol/contracts/blob/71e990b3f908ecf994be8723e041b584c7d49318/programs/svm-spoke/src/utils/transfer_utils.rs#L19-L28) to the `transfer_checked` call, which, when delegated to, will pass [the transfer validation logic](https://docs.rs/spl-token/latest/src/spl_token/processor.rs.html#274). Note that this scenario is mitigated by the fact that native instruction batching in Solana transactions would typically involve delegation and transferring in one operation, making such a front-running scenario less likely but still possible.

Consider validating the `signer` token account in the same way as `depositor_token_account`, and performing the transfer from the signer token account.

***Update:** Resolved in [pull request #971](https://github.com/across-protocol/contracts/pull/971), at commit [58f2665](https://github.com/across-protocol/contracts/commit/58f2665031836d5c9b0686870418f22f856a8e5a). The team stated:*

> *We replaced the one “state” PDA with two distinct PDAs—one for deposit and one for fill\_relay. Now users must explicitly delegate to the correct PDA before anyone can pull their tokens, restoring safe third‑party deposits and eliminating the risk of a single authority being misused to steal funds.*

---
### Example 2

**Auto Label:** Failure to validate or derive Program-Defined Addresses (PDAs) securely, enabling attackers to forge or manipulate account identities, leading to unauthorized access, state tampering, or fund theft.  

**Original Text Preview:**

## Marketplace Concerns and Remediation

## Overview

1. In the marketplace, `transfer_lamports_from_pda_min_balance` checks whether transferring the requested lamports will keep the recipient account rent-exempt after the transfer. If the transfer will leave the account below the rent-exempt threshold, the transfer is skipped entirely. However, this may result in some funds being stranded in the program-derived address if they are not enough to meet the threshold.

2. In the marketplace, there is a potential concern with mixing/matching instructions, where different types of NFTs may possibly be processed incorrectly.

3. `amm::close_expired_pool` should utilize `transfer_lamports_from_pda_min_balance` to safely transfer the remaining lamports from the PDA (Program Derived Address) of the pool account back to the original rent payer while ensuring the account balance never drops below the minimum required balance for account closure.

   > _program/src/state/pool.rs rust  
   /// Closes a pool and returns the rent to the rent payer and any remaining SOL to the owner.  
   pub fn close_pool<'info>(  
   pool: &Account<'info, Pool>,  
   rent_payer: AccountInfo<'info>,  
   owner: AccountInfo<'info>,  
   ) -> Result<()> {  
   [...]  
   let pool_state_bond = Rent::get()?.minimum_balance(POOL_SIZE);  
   let pool_lamports = pool.get_lamports();  
   // Any SOL above the minimum rent/state bond goes to the owner.  
   if pool_lamports > pool_state_bond {  
   let owner_amount = unwrap_int!(pool_lamports.checked_sub(pool_state_bond));  
   transfer_lamports_from_pda(&pool.to_account_info(), &owner, owner_amount)?;  
   }  
   // Rent goes back to the rent payer.  
   pool.close(rent_payer)  
   }

4. Define `is_none` generically in `toolbox::nullable` for improved clarity.

## Remediation

1. Truncate the amount transferred, calculating the maximum possible amount to be transferred while keeping the recipient below the rent-exempt threshold.
2. Introduce a type discriminator for NFTs to ensure that each instruction is matched to the correct NFT type.
3. Utilize `transfer_lamports_from_pda_min_balance` in `close_expired_pool`.
4. Update `nullable` with a generic implementation of `is_none`.

## Patch

1. Issue #1 resolved in b95be94.
2. Issue #3 resolved in 518f90b.

---
### Example 3

**Auto Label:** Failure to validate or derive Program-Defined Addresses (PDAs) securely, enabling attackers to forge or manipulate account identities, leading to unauthorized access, state tampering, or fund theft.  

**Original Text Preview:**

## Implementation Summary

In the current implementation, it is assumed that accounts with names ending in `*_ata`, especially `maker_output_ata`, are indeed associated token accounts (ATAs) for specific tokens and owners, without performing any explicit validation. This is especially important for the `maker_output_ata`, as if this account is not verified as an actual ATA for the specified maker and token, tokens intended for the maker may instead be sent to any arbitrary account owned by them.

## Code Reference

> _programs/limo/src/handlers/flash_take_order.rs_ (rust)

```rust
#[account(mut,
token::mint = output_mint,
token::authority = maker
)]
pub maker_output_ata: Box<InterfaceAccount<'info, TokenAccount>>,
```

## Remediation

Verify that each `*_ata` account is a valid associated token account and is correctly linked to the expected token mint and owner, preventing the diversion of assets to a non-designated account.

## Patch

Fixed in PR#30.

---
