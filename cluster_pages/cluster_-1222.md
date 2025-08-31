# Cluster -1222

**Rank:** #268  
**Count:** 31  

## Label
Inadequate input validation and improper state synchronization lead to invalid state transitions, incorrect data returns, and unauthorized operations, enabling malicious actors to forge messages, manipulate validator sets, or trigger harmful side effects.

## Cluster Information
- **Total Findings:** 31

## Examples

### Example 1

**Auto Label:** Failure to validate or track pending state transitions during validator removal or redelegation, leading to inconsistent or corrupted validator state and compromised delegation integrity.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-04-kinetiq/blob/7f29c917c09341672e73be2f7917edf920ea2adb/src/StakingManager.sol# L365>

### Finding description and impact

The `processValidatorRedelegation` function in the StakingManager contract contains an incorrect balance check that could prevent legitimate rebalancing operations from being executed. The function checks the HyperEVM balance of the StakingManager contract, but the funds being redelegated exist on HyperCore, not on the HyperEVM.

According to the documentation, HYPE staking on Hyperliquid happens within HyperCore. The rebalancing process is designed to delegate/undelegate funds between validators and staking balance on HyperCore without those funds ever leaving the HyperCore environment. However, the current implementation incorrectly checks the StakingManager balance that is on HyperEVM.

When the ValidatorManager’s `closeRebalanceRequests` function is called, it calculates the total amount to be redelegated and then calls `processValidatorRedelegation` on the StakingManager:
```

function closeRebalanceRequests(
    address stakingManager,
    address[] calldata validators
) external whenNotPaused nonReentrant onlyRole(MANAGER_ROLE) {
    // ...
    uint256 totalAmount = 0;
    for (uint256 i = 0; i < validators.length; ) {
        // ...
        totalAmount += request.amount;
        // ...
    }
    // Trigger redelegation through StakingManager if there's an amount to delegate
    if (totalAmount > 0) {
        IStakingManager(stakingManager).processValidatorRedelegation(totalAmount);
    }
}
```

In the StakingManager’s `processValidatorRedelegation` function, there’s an incorrect balance check:
```

function processValidatorRedelegation(uint256 amount) external nonReentrant whenNotPaused {
    require(msg.sender == address(validatorManager), "Only ValidatorManager");
    require(amount > 0, "Invalid amount");
@>    require(address(this).balance >= amount, "Insufficient balance");

    _distributeStake(amount, OperationType.RebalanceDeposit);
}
```

This incorrect balance check could cause legitimate rebalancing operations to fail if the StakingManager doesn’t have sufficient HYPE balance, even though the HyperCore balance is adequate for the redelegation. This would prevent the protocol from properly rebalancing funds between validators, which could lead to operational disruptions and reduced protocol performance.

### Recommended Mitigation Steps

Remove the incorrect balance check from the `processValidatorRedelegation` function:
```

function processValidatorRedelegation(uint256 amount) external nonReentrant whenNotPaused {
    require(msg.sender == address(validatorManager), "Only ValidatorManager");
    require(amount > 0, "Invalid amount");
-   require(address(this).balance >= amount, "Insufficient balance");

    _distributeStake(amount, OperationType.RebalanceDeposit);
}
```

**Kinetiq confirmed**

---

---
### Example 2

**Auto Label:** Inadequate input validation and improper state synchronization lead to invalid state transitions, incorrect data returns, and unauthorized operations, enabling malicious actors to forge messages, manipulate validator sets, or trigger harmful side effects.  

**Original Text Preview:**

In the [`initialize` function](https://github.com/matter-labs/zksync-sso-clave-contracts/blob/c7714c0fe0a33a23acce5aa20355f088d330b4f7/src/SsoAccount.sol#L59) of the `SsoAccount` contract, no validation is performed to ensure that the array arguments are non-empty. This omission allows the contract to be initialized in a state where it cannot perform any operations, rendering the account unusable.

Consider adding a validation check to ensure that at least one of the `initialValidators` or `initialK1Owners` arrays has a non-zero length before initialization. This will prevent the creation of non-functional accounts.

***Update:** Resolved in [pull request #371](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/371) at commit [8454fce](https://github.com/matter-labs/zksync-sso-clave-contracts/pull/371/commits/8454fce35e9178ef394466d5b99096cde7a89d36).*

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
