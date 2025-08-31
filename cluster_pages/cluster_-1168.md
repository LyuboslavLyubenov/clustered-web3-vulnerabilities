# Cluster -1168

**Rank:** #430  
**Count:** 9  

## Label
Inconsistent account validation and improper bounds checking allow malicious actors to bypass initialization, corrupt state, or trigger panics through incorrect account type parsing or memory overwrites.

## Cluster Information
- **Total Findings:** 9

## Examples

### Example 1

**Auto Label:** Inconsistent account validation and improper bounds checking allow malicious actors to bypass initialization, corrupt state, or trigger panics through incorrect account type parsing or memory overwrites.  

**Original Text Preview:**

## Account Initialization Issues with Constraints in Anchor

If an account marked with both `init` and `zero` constraints in Anchor is utilized in multiple locations within a context, and the same account is referenced, it may result in incorrect initialization. Specifically, the `init` constraint creates an account with a specific space and initializes it with the expected discriminator, but when a `zero` constraint is also applied to the same account, the constraint will pass successfully and overwrite the same account. Consequently, one account is initialized instead of two.

## Example Code

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Foo::SIZE,
    )]
    foo: Account<'info, Foo>,
    
    #[account(
        zero,
    )]
    bar: Account<'info, Bar>,
}
```

## Remediation

Deduplicate account initializer constraints to ensure that no duplicate accounts are passed in.

## Patch

Fixed in PR#3422

---
### Example 2

**Auto Label:** Inconsistent account validation and improper bounds checking allow malicious actors to bypass initialization, corrupt state, or trigger panics through incorrect account type parsing or memory overwrites.  

**Original Text Preview:**

## Anchor Zero Constraint

In Anchor, the zero constraint is used to check that the initial account data is empty by verifying that all discriminator bytes are zero. This is usually used when the account initialization occurs in a prior instruction.

> **Example:** `example.rs`
> ```rust
> #[derive(Accounts)]
> pub struct Initialize<'info> {
>     #[account(zero)]
>     bar: AccountLoader<'info, Bar>,
>     #[account(zero)]
>     foo: AccountLoader<'info, Foo>,
> }
> ```

Suppose two accounts, `bar` and `foo` (shown above), are both set with the zero constraint in the instruction context. If the same account is passed into both `bar` and `foo` in the transaction, this will result in unintended overwriting, with one account being initialized instead of two.

```rust
let mut __data: &[u8] = &bar.try_borrow_data()?;
let __disc = &__data[..Bar::DISCRIMINATOR.len()];
let __has_disc = __disc.iter().any(|b| *b != 0);
if __has_disc {
    return Err(
        anchor_lang::error::Error::from(
            anchor_lang::error::ErrorCode::ConstraintZero,
        )
        .with_account_name("bar"),
    );
}
```

After confirming that both account constraints have zeroed discriminators, Anchor will write the `bar` discriminator in the first part of the account’s memory, initializing it as a `Bar` type. Then, without additional checks, it proceeds to `foo` and overwrites the memory by writing the `foo` discriminator in the same account region.

## Remediation

Deduplicate the zero constraint such that no duplicate accounts may be passed in.

## Patch

Fixed in PR#3409

---
### Example 3

**Auto Label:** Inconsistent account validation and improper bounds checking allow malicious actors to bypass initialization, corrupt state, or trigger panics through incorrect account type parsing or memory overwrites.  

**Original Text Preview:**

## Vulnerability Overview

This vulnerability arises when utilizing single-byte custom discriminators in the Anchor framework. Normally, an account type’s discriminator is utilized to ensure account data is associated with the correct program. However, if an account with only a single-byte discriminator (such as `[account(discriminator = [89])]`) is initialized with the zero constraint, it checks only that the first byte of the account’s data is zero. This may result in an accidental match with other account types that generate default 8-byte discriminators if their discriminator’s first byte is also zero.

## Proof of Concept

```rust
#[account(zero)]
Foo: Account<'info, Foo>,
[...]
#[account(discriminator = [89])]
pub struct Foo {
    [...]
}
```

In the example provided above, suppose the Ab account type has a default 8-byte discriminator, where its first byte is zero. This creates a 1-in-256 chance that the Ab account type will have the same starting byte as Foo (the single-byte `[89]` discriminator) and thus be erroneously interpreted as Foo. Consequently, the account data of Ab will be overwritten if it is passed in.

## Remediation

Ensure that the zero constraint is modified to check at least 8 bytes, regardless of the discriminator size, to reduce the likelihood of accidental matches.

## Patch

Fixed in PR#3365

---
