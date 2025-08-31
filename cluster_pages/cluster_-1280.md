# Cluster -1280

**Rank:** #100  
**Count:** 168  

## Label
Inadequate input validation and improper state synchronization lead to invalid state transitions, incorrect data returns, and unauthorized operations, enabling malicious actors to forge messages, manipulate validator sets, or trigger harmful side effects.

## Cluster Information
- **Total Findings:** 168

## Examples

### Example 1

**Auto Label:** Inadequate input validation and improper state synchronization lead to invalid state transitions, incorrect data returns, and unauthorized operations, enabling malicious actors to forge messages, manipulate validator sets, or trigger harmful side effects.  

**Original Text Preview:**

In the [`initialize` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/SsoAccount.sol#L59) of the `SsoAccount` contract, no validation is performed to ensure that the array arguments are non-empty. This omission allows the contract to be initialized in a state where it cannot perform any operations, rendering the account unusable.

Consider adding a validation check to ensure that at least one of the `initialValidators` or `initialK1Owners` arrays has a non-zero length before initialization. This will prevent the creation of non-functional accounts.

***Update:** Resolved in [pull request #371](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/371) at commit [8454fce](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/371/commits/8454fce35e9178ef394466d5b99096cde7a89d36).*

---
### Example 2

**Auto Label:** Failure to validate state before execution enables attackers to trigger reverts, duplicate operations, or infinite loops, leading to denial of service and loss of control.  

**Original Text Preview:**

The [`initRecovery` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/validators/GuardianRecoveryValidator.sol#L202-L215) of the `GuardianRecoveryValidator` contract lacks a crucial validation step, allowing for the overwriting of existing recovery data without any checks for ongoing recovery processes. This oversight permits a guardian to initiate a new recovery using incorrect data or to refresh the timestamp, thereby obstructing or postponing the intended recovery process.

This vulnerability is particularly problematic in scenarios where an account is protected by multiple guardians. In such cases, a single guardian acting maliciously can indefinitely disrupt the recovery process by:

1. Overwriting existing recovery data with malicious or incorrect information.
2. Forcing well-intentioned guardians to overwrite this malicious data with the correct information, only for the cycle to repeat ad infinitum.

This cycle not only stalls the recovery process but also leaves no recourse for removing the malicious guardian and regaining control of the account, as the necessary `Transaction` cannot be executed by any party.

To address these vulnerabilities, consider implementing a consensus mechanism among the guardians for recovery processes involving multiple guardians. Such a mechanism would only allow a recovery to proceed once a majority of guardians have concurred on the submitted parameters. Additionally, for accounts with a single guardian, consider enhancing the `initRecovery` function by incorporating a preliminary check to ascertain when a recovery process is still active. These measures will significantly mitigate the risk of overwrites and delays, thereby bolstering the security and efficacy of the recovery process.

***Update:** Partially resolved in [pull request #379](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/379) at commit [f2b6bbf](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/379/commits/f2b6bbfe95f791f157c4174b019b59481f03643e). The Matter Labs team stated:*

> *We decided not to include an N-out-of-M design in this iteration; instead, we added a recommendation to the user-facing documentation to use a Multisig as a guardian to replicate this behavior.*

---
### Example 3

**Auto Label:** Inadequate input validation and improper state synchronization lead to invalid state transitions, incorrect data returns, and unauthorized operations, enabling malicious actors to forge messages, manipulate validator sets, or trigger harmful side effects.  

**Original Text Preview:**

## Diﬃculty: High

## Type: Data Validation

## Description

The `update_folio` instruction allows `program_version` and `program_deployment_slot` arguments to be passed independently (figure 9.1). However, if the owner passes one without the other, they risk locking themselves out of the program. The relevant code appears in figure 9.1. The first thing `update_folio` does is call `validate` on line 96, which calls `validate_folio_program_post_init` on line 70 of figure 9.2. The call to `validate_folio_program_post_init` results in a call to `validate_program_registrar`, part of which appears in figure 9.3.

```rust
85    pub fn handler(
86        ctx: Context<UpdateFolio>,
87        program_version: Option<Pubkey>,
88        program_deployment_slot: Option<u64>,
89        folio_fee: Option<u128>,
90        minting_fee: Option<u128>,
91        trade_delay: Option<u64>,
92        auction_length: Option<u64>,
93        fee_recipients_to_add: Vec<FeeRecipient>,
94        fee_recipients_to_remove: Vec<Pubkey>,
95    ) -> Result<()> {
96        ctx.accounts.validate()?;
```
*Figure 9.1: The `update_folio` function signature*  
*(dtfs-solana/programs/folio/src/instructions/owner/update_folio.rs#85–95)*

```rust
68    pub fn validate(&self) -> Result<()> {
69        let folio = self.folio.load()?;
70        folio.validate_folio_program_post_init(
71            &self.folio.key(),
72            Some(&self.program_registrar),
73            Some(&self.dtf_program),
74            Some(&self.dtf_program_data),
75            Some(&self.actor),
76            Some(Role::Owner),
77            None, // Can update no matter the status
78        )?;
79
80        Ok(())
81    }
```
*Figure 9.2: Definition of `UpdateFolio::validate`*  
*(dtfs-solana/programs/folio/src/instructions/owner/update_folio.rs#68–81)*

```rust
87    let deployment_slot = DtfProgram::get_program_deployment_slot(
88        &dtf_program.key(),
89        &dtf_program.to_account_info(),
90        &dtf_program_data.to_account_info(),
91    )?;
92
93    check_condition!(
94        self.program_deployment_slot == deployment_slot,
95        InvalidProgram
96    );
```
*Figure 9.3: Excerpt of `Folio::validate_program_registrar`*  
*(dtfs-solana/programs/folio/src/utils/accounts/folio.rs#87–96)*

The code in figure 9.3 determines the deployment slot using values stored in the Folio. The determined deployment slot is compared to the one stored in the Folio. If there is a mismatch, the call fails.

If the owner calls `update_folio` with only one of the two values and then calls `update_folio` again to correct the error, the second call will fail in `UpdateFolio::validate`.

## Exploit Scenario

Alice calls `update_folio` to update her deployment’s program ID and deployment slot; however, she forgets to pass the deployment slot. Alice notices her error and calls `update_folio` a second time to correct it. The second call to `update_folio` fails when it calls `UpdateFolio::validate`.

## Recommendations

Short term, have `update_folio` call `UpdateFolio::validate` before returning to ensure that subsequent calls to `UpdateFolio::validate` will succeed. This will reduce the likelihood that Folio owners will lock themselves out of the program.

Long term, expand the folio program’s tests. It is possible that this bug could have been found with more extensive negative (i.e., failing) tests.

---
