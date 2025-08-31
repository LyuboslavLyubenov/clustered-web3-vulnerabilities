# Cluster -1012

**Rank:** #214  
**Count:** 50  

## Label
Insufficient state validation and improper token flow ordering enable attackers to mint unauthorized tokens or create imbalances, leading to insolvency, unauthorized exposure, and loss of value.

## Cluster Information
- **Total Findings:** 50

## Examples

### Example 1

**Auto Label:** **Misaligned token valuation and improper principal validation lead to inflated supply, unauthorized minting, and fund loss through flawed conversion and redemption logic.**  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-lend-audit-contest-judging/issues/224 

## Found by 
Emine, Kirkeelee, PNS, Waydou, Z3R0, h2134, jokr, mahdifa, oxelmiguel, patitonar, zxriptor

### Summary

A user can supply nearly all their collateral on Chain A and borrow against it on Chain B, then supply a dust amount on Chain B and borrow a dust amount on Chain A. This sequence results in both cross-chain borrow and collateral mappings being populated for the same asset and user on both chains, violating protocol invariant. As a result, the protocol cannot calculate or liquidate the user’s position, because the function `borrowWithInterest ` reverts, allowing the user to bridge out nearly all their collateral and evade liquidation.

### Root Cause

The protocol enforces an invariant in the `borrowWithInterest(address borrower, address _lToken)` function that, for any user and underlying asset on a given chain, only one of `crossChainBorrows `or `crossChainCollaterals `should be populated. However, if a user first supplies collateral on Chain A and borrows on Chain B (populating `crossChainBorrows `on A and `crossChainCollaterals `on B), and then supplies a dust amount on Chain B and borrows a dust amount on Chain A (populating `crossChainBorrows `on B and `crossChainCollaterals `on A), both mappings become populated for the same user and asset on both chains. This breaks the protocol’s invariant, causing the require statement in `borrowWithInterest `to revert. As a result, any function that checks or calculates borrow balances (including liquidation logic) will fail for that user and asset. The user loses out on LEND rewards for the position but is able to bridge out almost all their collateral, leaving the protocol unable to liquidate or close their position.

https://github.com/sherlock-audit/2025-05-lend-audit-contest/blob/main/Lend-V2/src/LayerZero/LendStorage.sol#L478-L486

### Internal Pre-conditions

The protocol allows users to open cross-chain borrows in both directions for the same asset.

### External Pre-conditions

The user can supply a dust (very small) amount on one chain and borrow a dust amount on the other, in addition to their main position.

### Attack Path

1. User supplies almost all collateral on Chain A.
2. User borrows nearly the full amount on Chain B (cross-chain borrow).
3. User supplies a dust amount on Chain B.
4. User borrows a dust amount on Chain A (reverse cross-chain borrow).
5. Now, both `crossChainBorrows `and `crossChainCollaterals `are populated for the same asset and user on both chains.
6. Any liquidation or borrow calculation reverts due to the require in `borrowWithInterest`, making the position uncloseable and non-liquidatable.
7. User loses out on LEND rewards for the position but successfully bridges out almost all their collateral.

### Impact

The protocol cannot calculate or liquidate the user’s position, as `borrowWithInterest `reverts. The user can bridge out nearly all their collateral, leaving the protocol with an uncloseable, non-liquidatable position. This results in bad debt accumulation: the protocol is left with outstanding borrows that cannot be repaid or liquidated, directly threatening solvency and leading to potential loss of funds for other users and the protocol itself.

### PoC

_No response_

### Mitigation

Redesign the cross-chain borrow/collateral tracking logic to prevent both mappings from being populated for the same user and asset on the same chain.

---
### Example 2

**Auto Label:** Insufficient state validation and improper token flow ordering enable attackers to mint unauthorized tokens or create imbalances, leading to insolvency, unauthorized exposure, and loss of value.  

**Original Text Preview:**

## Token Swap Vulnerability

`claim_native` and `return_native` in bridge allow users to swap wrapped tokens (WT) for native tokens and vice versa. However, an attacker can repeatedly cycle a small token amount between minting (`claim_native`) and burning (`return_native`). Since each minting operation contributes to the epoch’s minting limit, this exploitation will exhaust the treasury’s capacity within an epoch, preventing legitimate users from minting new tokens.

## Remediation

Remove the ability for the user to wrap and unwrap purely inside Sui and force them to go cross-chain, which should significantly slow down such an attack.

## Patch

Resolved in `a58183d`.

---
### Example 3

**Auto Label:** Race conditions and inadequate state validation enable unauthorized token withdrawal or manipulation, leading to permanent fund loss or loss of user control.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

HoneyLocker migrations are authorized based on the codehashes of the contracts:

```solidity
function migrate(address[] calldata _LPTokens, uint256[] calldata _amountsOrIds ,address payable _newHoneyLocker) external onlyOwner {
    // check migration is authorized based on codehashes
    if (!HONEY_QUEEN.isMigrationEnabled(address(this).codehash, _newHoneyLocker.codehash)) {
        revert MigrationNotEnabled();
    }

    ...
}
```

The problem is that the function assumes that it would be migrated to another HoneyLocker with the same initialized values.

But it is possible to create a new `HoneyLocker` with different values for `unlocked`, `HONEY_QUEEN`, and `referral`.

This can lead to many impacts. The most notable one is the possibility to withdraw all the LP tokens that were supposed to remain locked until the expiration time.

Other attacks can be performed given a malicious HoneyQueen contract provided by the user.

### Proof of Concept

This test shows how some locked tokens can be withdrawn by migrating to an unlocked HoneyLocker.

- Copy the test to `test/HoneyLocker.t.sol`
- Run `forge test --mt test_migrationExploit`

```solidity
function test_migrationExploit() external prankAsTHJ {
    // LEGIT SETUP

    // deposit first some into contract
    uint256 balance = HONEYBERA_LP.balanceOf(THJ);
    HONEYBERA_LP.approve(address(honeyLocker), balance);
    honeyLocker.depositAndLock(address(HONEYBERA_LP), balance, expiration);

    // clone it
    HoneyLockerV2 honeyLockerV2 = new HoneyLockerV2();
    honeyLockerV2.initialize(THJ, address(honeyQueen), referral, false);

    // set hashcode in honeyqueen then attempt migration
    honeyQueen.setMigrationFlag(true, address(honeyLocker).codehash, address(honeyLockerV2).codehash);

    // CREATE A NEW LOCKER, INITIALIZE IT WITH MALICIOUS DATA AND MIGRATE TO IT
    HoneyLockerV2 maliciousLocker = new HoneyLockerV2();
    address maliciousHoneyQueen = makeAddr("MALICIOUS_HONEY_QUEEN");
    address anotherReferral = makeAddr("ANOTHER_REFERRAL");
    bool unlocked = true;
    maliciousLocker.initialize(THJ, address(maliciousHoneyQueen), anotherReferral, unlocked);

    honeyLocker.migrate(SLA.addresses(address(HONEYBERA_LP)), SLA.uint256s(balance), payable(address(maliciousLocker)));
    assertEq(HONEYBERA_LP.balanceOf(address(maliciousLocker)), balance);

    // The malicious owner got back their tokens
    maliciousLocker.withdrawLPToken(address(HONEYBERA_LP), balance);
    assertEq(HONEYBERA_LP.balanceOf(THJ), balance);
}
```

## Recommendations

One possible way to mitigate this is to add additional checks to verify that the `unlocked`, `HONEY_QUEEN`, and `referral` values are the same in both locks.

---
