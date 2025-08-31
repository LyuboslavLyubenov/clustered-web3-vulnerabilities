# Cluster -1167

**Rank:** #346  
**Count:** 18  

## Label
Gas-based denial-of-service and resource exhaustion via unbounded or malicious gas parameter manipulation, enabling attackers to flood systems, inflate costs, or bypass safeguards at negligible cost.

## Cluster Information
- **Total Findings:** 18

## Examples

### Example 1

**Auto Label:** Attackers exploit predictable account derivation and missing existence checks to prevent legitimate transactions, enabling denial-of-service via off-chain account creation or state reuse.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-01-pump-science/blob/768ef58478724bf6b464c9f0952e3e5a3b2a2613/programs/pump-science/src/instructions/migration/lock_pool.rs# L149>

The `lock_pool` operation requires the creation of a `lockEscrow` account. However, a malicious actor could preemptively create the `lockEscrow` account, causing the `create_lock_escrow` transaction to fail and resulting in a Denial of Service (DoS) for the `lock_pool` operation.

### Proof of Concept

During the `lock_pool` process, the `create_lock_escrow` function is called to create the `lock_escrow` account.
```

    // Create Lock Escrow
    let escrow_accounts = vec![
        AccountMeta::new(ctx.accounts.pool.key(), false),
        AccountMeta::new(ctx.accounts.lock_escrow.key(), false),
        AccountMeta::new_readonly(ctx.accounts.fee_receiver.key(), false),
        AccountMeta::new_readonly(ctx.accounts.lp_mint.key(), false),
        AccountMeta::new(ctx.accounts.bonding_curve_sol_escrow.key(), true), // Bonding Curve Sol Escrow is the payer/signer
        AccountMeta::new_readonly(ctx.accounts.system_program.key(), false),
    ];

    let escrow_instruction = Instruction {
        program_id: meteora_program_id,
        accounts: escrow_accounts,
        data: get_function_hash("global", "create_lock_escrow").into(),
    };

    invoke_signed(
        &escrow_instruction,
        &[
            ctx.accounts.pool.to_account_info(),
            ctx.accounts.lock_escrow.to_account_info(),
            ctx.accounts.fee_receiver.to_account_info(),
            ctx.accounts.lp_mint.to_account_info(),
            ctx.accounts.bonding_curve_sol_escrow.to_account_info(), // Bonding Curve Sol Escrow is the payer/signer
            ctx.accounts.system_program.to_account_info(),
        ],
        bonding_curve_sol_escrow_signer_seeds,
    )?;
```

However, the `lock_escrow` account is derived using the `pool` and `owner` as seeds, and its creation does not require the owner’s signature. This means that a malicious actor could preemptively create the `lock_escrow` account to perform a DoS attack on the `lock_pool` operation.
```

/// Accounts for create lock account instruction
#[derive(Accounts)]
pub struct CreateLockEscrow<'info> {
    /// CHECK:
    pub pool: UncheckedAccount<'info>,

    /// CHECK: Lock account
    #[account(
        init,
        seeds = [
            "lock_escrow".as_ref(),
            pool.key().as_ref(),
            owner.key().as_ref(),
        ],
        space = 8 + std::mem::size_of::<LockEscrow>(),
        bump,
        payer = payer,
    )]
    pub lock_escrow: UncheckedAccount<'info>,

    /// CHECK: Owner account
@>  pub owner: UncheckedAccount<'info>,

    /// CHECK: LP token mint of the pool
    pub lp_mint: UncheckedAccount<'info>,

    /// CHECK: Payer account
    #[account(mut)]
    pub payer: Signer<'info>,

    /// CHECK: System program.
    pub system_program: UncheckedAccount<'info>,
}
```

### Recommended mitigation steps

In the `lock_pool` process, check if the `lock_escrow` exists. If it exists, skip the creation process.

**Kulture (Pump Science) confirmed**

---

---
### Example 2

**Auto Label:** Attackers exploit predictable account derivation and missing existence checks to prevent legitimate transactions, enabling denial-of-service via off-chain account creation or state reuse.  

**Original Text Preview:**

**Severity:** Critical

**Path:** programs/fusion-swap/src/lib.rs#L106-L258

**Description:**
The Fusion Swap program allows a user to create an order and cancel an order permissionlessly, containing some set of configurable parameters. A taker would have to accept the order by submitting a transaction calling `fill`, which requires the taker to be whitelisted.

Nevertheless, it is possible to steal from the taker directly by front-running the order fulfillment and swapping out the order’s data by reusing the order ID.

The order is stored at a PDA that is calculated from a constant seed, the maker’s address and the order ID. If the maker cancels their order, the escrow PDA is completely closed (as it should), but this allows for reinitialization and reuse of the same order ID and escrow address.

This means that a transaction containing `fill` and the escrow account from the initial order, could be forced to fill a second malicious order that uses the same order ID. 

For example, the attacker can swap out:

- The rate at which the token is sold, making the taker pay more.

- Increase the fees, making the taker pay more.

- Change the tokens to be native using the `bool` switch, which will cause the provided token to be ignored and the taker’s SOL balance to be used instead of for example a USD stable coin.

```
...    
    pub fn fill(ctx: Context<Fill>, order_id: u32, amount: u64) -> Result<()> {
        require!(
            Clock::get()?.unix_timestamp <= ctx.accounts.escrow.expiration_time as i64,
            EscrowError::OrderExpired
        );

        require!(
            amount <= ctx.accounts.escrow.src_remaining,
            EscrowError::NotEnoughTokensInEscrow
        );

        require!(amount != 0, EscrowError::InvalidAmount);

        // Update src_remaining
        ctx.accounts.escrow.src_remaining -= amount;

        // Escrow => Taker
        transfer_checked(
            CpiContext::new_with_signer(
                ctx.accounts.src_token_program.to_account_info(),
                TransferChecked {
                    from: ctx.accounts.escrow_src_ata.to_account_info(),
                    mint: ctx.accounts.src_mint.to_account_info(),
                    to: ctx.accounts.taker_src_ata.to_account_info(),
                    authority: ctx.accounts.escrow.to_account_info(),
                },
                &[&[
                    "escrow".as_bytes(),
                    ctx.accounts.maker.key().as_ref(),
                    order_id.to_be_bytes().as_ref(),
                    &[ctx.bumps.escrow],
                ]],
            ),
            amount,
            ctx.accounts.src_mint.decimals,
        )?;

        let dst_amount = get_dst_amount(
            ctx.accounts.escrow.src_amount,
            ctx.accounts.escrow.min_dst_amount,
            amount,
            Some(&ctx.accounts.escrow.dutch_auction_data),
        )?;

        let (protocol_fee_amount, integrator_fee_amount, maker_dst_amount) = get_fee_amounts(
            ctx.accounts.escrow.fee.integrator_fee as u64,
            ctx.accounts.escrow.fee.protocol_fee as u64,
            ctx.accounts.escrow.fee.surplus_percentage as u64,
            dst_amount,
            get_dst_amount(
                ctx.accounts.escrow.src_amount,
                ctx.accounts.escrow.estimated_dst_amount,
                amount,
                None,
            )?,
        )?;

        // Take protocol fee
        if protocol_fee_amount > 0 {
            let protocol_dst_ata = ctx
                .accounts
                .protocol_dst_ata
                .as_ref()
                .ok_or(EscrowError::InconsistentProtocolFeeConfig)?;

            transfer_checked(
                CpiContext::new(
                    ctx.accounts.dst_token_program.to_account_info(),
                    TransferChecked {
                        from: ctx.accounts.taker_dst_ata.to_account_info(),
                        mint: ctx.accounts.dst_mint.to_account_info(),
                        to: protocol_dst_ata.to_account_info(),
                        authority: ctx.accounts.taker.to_account_info(),
                    },
                ),
                protocol_fee_amount,
                ctx.accounts.dst_mint.decimals,
            )?;
        }

        // Take integrator fee
        if integrator_fee_amount > 0 {
            let integrator_dst_ata = ctx
                .accounts
                .integrator_dst_ata
                .as_ref()
                .ok_or(EscrowError::InconsistentIntegratorFeeConfig)?;

            transfer_checked(
                CpiContext::new(
                    ctx.accounts.dst_token_program.to_account_info(),
                    TransferChecked {
                        from: ctx.accounts.taker_dst_ata.to_account_info(),
                        mint: ctx.accounts.dst_mint.to_account_info(),
                        to: integrator_dst_ata.to_account_info(),
                        authority: ctx.accounts.taker.to_account_info(),
                    },
                ),
                integrator_fee_amount,
                ctx.accounts.dst_mint.decimals,
            )?;
        }

        // Taker => Maker
        if ctx.accounts.escrow.native_dst_asset {
            // Transfer native SOL
            anchor_lang::system_program::transfer(
                CpiContext::new(
                    ctx.accounts.system_program.to_account_info(),
                    anchor_lang::system_program::Transfer {
                        from: ctx.accounts.taker.to_account_info(),
                        to: ctx.accounts.maker_receiver.to_account_info(),
                    },
                ),
                maker_dst_amount,
            )?;
        } else {
            let maker_dst_ata = ctx
                .accounts
                .maker_dst_ata
                .as_ref()
                .ok_or(EscrowError::MissingMakerDstAta)?;

            // Transfer SPL tokens
            transfer_checked(
                CpiContext::new(
                    ctx.accounts.dst_token_program.to_account_info(),
                    TransferChecked {
                        from: ctx.accounts.taker_dst_ata.to_account_info(),
                        mint: ctx.accounts.dst_mint.to_account_info(),
                        to: maker_dst_ata.to_account_info(),
                        authority: ctx.accounts.taker.to_account_info(),
                    },
                ),
                maker_dst_amount,
                ctx.accounts.dst_mint.decimals,
            )?;
        }

        // Close escrow if all tokens are filled
        if ctx.accounts.escrow.src_remaining == 0 {
            close_escrow(
                ctx.accounts.src_token_program.to_account_info(),
                &ctx.accounts.escrow,
                ctx.accounts.escrow_src_ata.to_account_info(),
                ctx.accounts.maker.to_account_info(),
                order_id,
                ctx.bumps.escrow,
            )?;
        }

        Ok(())
    }
...
```

**Remediation:**  The order ID should not be reusable for the maker, such that the same PDA can never be derived twice. This can be implemented using a new state account that keeps a bitmap of maker to order IDs.

Another layer of protection could be a slippage control for the taker to specify the maximum amount of tokens to be spent.

**Status:**  Fixed


- - -

---
### Example 3

**Auto Label:** Attackers exploit predictable account derivation and missing existence checks to prevent legitimate transactions, enabling denial-of-service via off-chain account creation or state reuse.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `CreateBondingCurve` instruction requires passing `bonding_curve_token_account`:

```rust
    #[account(
        init,
        payer = creator,
        associated_token::mint = mint,
        associated_token::authority = bonding_curve,
    )]
    bonding_curve_token_account: Box<Account<'info, TokenAccount>>,
```

`bonding_curve_token_account` uses `associated_token::mint` and `associated_token::authority`, so the `TokenAccount` can be created in advance.
If this already exists, creating the function will fail because init is used.

An attacker can extract the calculated `bonding_curve` address and create a `TokenAccount` to prevent users from creating a Bonding Curve.

## Recommendations

````diff
    #[account(
-        init,
+        init_if_needed,
        payer = creator,
        associated_token::mint = mint,
        associated_token::authority = bonding_curve,
    )]
    bonding_curve_token_account: Box<Account<'info, TokenAccount>>,```

---
