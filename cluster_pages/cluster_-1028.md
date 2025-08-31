# Cluster -1028

**Rank:** #64  
**Count:** 279  

## Label
Common vulnerability type: Exploitable price or state manipulation through timing, front-running, and insufficient validation of dynamic market conditions, enabling profit extraction via sandwich attacks and arbitrage.

## Cluster Information
- **Total Findings:** 279

## Examples

### Example 1

**Auto Label:** Common vulnerability type: **State inconsistency and missing validation leading to value manipulation, supply overinflation, or denial of service through dynamic reserve or price exploits.**  

**Original Text Preview:**

##### Description
This issue has been identified within the `DIAExternalStaking` contract.
The contract is vulnerable to a classic inflation attack due to the permissionless nature of the `addRewardToPool` function and the way pool shares are calculated. An attacker can exploit this as follows in newly deployed pool:

1. The attacker stakes in an empty pool the minimum allowed amount (`minimumStake`), receiving `minimumStake` share of the pool.
2. The attacker then unstakes almost all of their tokens, leaving only a single token (i.e., unstakes `minimumStake - 1`).
3. At this point, the total pool shares and the total supply are both equal to 1.
4. The attacker sees in the mempool a transaction of victim and frontruns it, calling `addRewardToPool()` with amount == victim_deposit.
5. After this step there is 1 share in the pool and victim_deposit + 1 stake. So victim receives:
`poolSharesGiven = (amount * totalShareAmount) / totalPoolSize = victim_deposit * 1 / (victim_deposit + 1) = 0` shares
6. The attacker can unstake funds and repeat the attack on an empty pool.

The issue is classified as **Critical** severity because it allows a malicious actor to steal user's funds.
<br/>
##### Recommendation
We recommend adding a non-zero shares minted requirement to the `_stake` function or add a special variable which will be passed to the function by the user to control for the minimum shares minted. Also, we recommend restricting who can call `addRewardToPool` function.

> **Client's Commentary:**
> Client: The issue has been fixed in commit https://github.com/diadata-org/lumina-staking/commit/95aca806165c15ab1826ff19cdac8104019f6fd9

---
### Example 2

**Auto Label:** Common vulnerability type: **Input validation failures enabling unauthorized access, gas exhaustion, or reward manipulation through forged or manipulated state transitions.**  

**Original Text Preview:**

## Severity

Critical Risk

## Description

The `DaosVesting::claim()` function has a critical vulnerability that allows an attacker to drain tokens from legitimate DAOs. The issue stems from insufficient validation of the DAO contract address and its state.

Vulnerable code in `DaosVesting::claim()`:

```solidity
function claim(address dao, address user, uint256 index) external {
    IDaosLive daosLive = IDaosLive(dao);
@-> address token = daosLive.token();

@-> uint256 maxPercent = getClaimablePercent(dao, index);
    if (maxPercent > DENOMINATOR) revert InvalidCalculation();

    unchecked {
@->     uint256 totalAmount = daosLive.getContributionTokenAmount(user);
        uint256 maxAmount = (totalAmount * maxPercent) / DENOMINATOR;
        if (maxAmount < claimedAmounts[dao][user]) {
            revert InvalidCalculation();
        }
        uint256 amount = maxAmount - claimedAmounts[dao][user];

        if (amount > 0) {
            claimedAmounts[dao][user] += amount;
            totalClaimeds[dao] += amount;
@->         TransferHelper.safeTransfer(token, user, amount);
            observer.emitClaimed(dao, token, user, amount);
        }
    }
}
```

**Attack Vector:**

1. Attacker deploys fake malicious contract implementing `IDaosLive`
2. Sets `token` address to a legitimate DAO's token
3. Implements `getContributionTokenAmount()` to manually return the amount of legit `DaosLive` tokens locked in `DaoVesting.sol` contract.
4. Sets vesting schedules of that Fake `IDaosLive` implementation with 100% unlock for an `index`.
5. Calls `claim()` on the vesting contract with their Fake malicious `DoasLive` contract address
6. Receives tokens from the legitimate DAO's vesting contract

## Location of Affected Code

File: [contracts/DaosVesting.sol#L91](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosVesting.sol#L91)

```solidity
function claim(address dao, address user, uint256 index) external {
    IDaosLive daosLive = IDaosLive(dao);
@-> address token = daosLive.token();

@-> uint256 maxPercent = getClaimablePercent(dao, index);
    if (maxPercent > DENOMINATOR) revert InvalidCalculation();

    unchecked {
@->     uint256 totalAmount = daosLive.getContributionTokenAmount(user);
        uint256 maxAmount = (totalAmount * maxPercent) / DENOMINATOR;
        if (maxAmount < claimedAmounts[dao][user]) {
            revert InvalidCalculation();
        }
        uint256 amount = maxAmount - claimedAmounts[dao][user];

        if (amount > 0) {
            claimedAmounts[dao][user] += amount;
            totalClaimeds[dao] += amount;
@->         TransferHelper.safeTransfer(token, user, amount);
            observer.emitClaimed(dao, token, user, amount);
        }
    }
}
```

## Impact

- Complete drain of tokens from legitimate DAOs
- Loss of all funds of vested tokens that should have been claimed by the real contributors of `DaosLive`
- Permanent loss of tokens
- Affects all DAOs using the vesting contract.

## Recommendation

- Restrict the caller to claim the tokens only through `DaosLive` contract function, claim only. Do not allow users to clam directly through the vesting contract.
- Validate that the DAO contract is legitimate or not.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Common vulnerability type: **State inconsistency and missing validation leading to value manipulation, supply overinflation, or denial of service through dynamic reserve or price exploits.**  

**Original Text Preview:**

## Vulnerability Assessment

**Difficulty:** High  

**Type:** Data Validation  

## Description  
An attacker can complete a user’s pool liquidity-providing action by transferring an insignificant amount to the pool with the `add_liquidity` message, mentioning the same receiver as the user. This causes the user to receive significantly fewer LP tokens than expected.

The FIVA protocol allows users to provide liquidity to the SY/PT pool. The liquidity provision process requires users to transfer both SY and PT tokens to the Pool contract with a `forward_payload`, including an `add_liquidity` operation. On the first token transfer, a Deposit contract is deployed to store the Pool address, recipient address, and the amount of SY or PT transferred by the user. On the second token transfer, the Deposit contract checks if both SY and PT are deposited and a slippage parameter is provided, then sends a `provide_lp` message to the Pool contract to mint LP tokens to the recipient.

```cpp
if (op == op::add_liquidity) {
    sy_balance += in_msg_body~load_coins();
    pt_balance += in_msg_body~load_coins();
    int min_lp_out = in_msg_body~load_coins();
    int with_unwrap = in_msg_body~load_uint(1);
    ;; TODO: consider to use min balance limit (i.e. 1000 instead of 0)
    if ((min_lp_out > 0) & (sy_balance > 0) & (pt_balance > 0)) {
        send_provide_lp(owner_addr, recipient_addr, sy_balance, pt_balance,
                         min_lp_out, query_id, with_unwrap);
        sy_balance = 0;
        pt_balance = 0;
    }
    save_data(owner_addr, recipient_addr, sy_balance, pt_balance);
    return ();
}
```

_Figure 9.1: The `add_liquidity` operation handler in the Deposit contract_  
`contracts/AMM/lp_deposit/deposit.fc#77-92`

However, an attacker can exploit this process by making the second transfer before the legitimate user, sending a negligible amount of tokens while specifying the same receiver as the user and an arbitrary value for `min_lp_out`. Because of the same pool and receiver combination, the attacker’s `add_liquidity` message is forwarded to the user’s Deposit contract, which then sends a `provide_lp` operation message to the pool. Since the amounts of PT and SY are imbalanced, the user will only receive the LP tokens minted by the minimum of the PT and SY tokens while losing all the tokens deposited in the first token transfer. The slippage protection is bypassed because the attacker controls the `min_lp_out` parameter.

## Exploit Scenario  
An attacker exploits a vulnerability in the two-phase deposit process by targeting users who have deposited 10,000 SY tokens into the Pool contract but have not yet deposited their PT tokens. After identifying a victim’s Deposit contract, the attacker sends a negligible amount of PT tokens, just enough to mint 1 nanoton LP token, to the pool while setting the victim as the recipient and specifying an extremely low `min_lp_out` value of 1, to bypass the slippage protection.

When the Deposit contract processes the `add_liquidity` operation, it executes the `send_provide_lp` function using the attacker’s manipulated parameters. The victim receives 1 nanoton LP token while losing their entire 10,000 SY token deposit.

## Recommendations  
- **Short term:** Include the address of the SY or PT token sender address in the Deposit contract’s `init_state` to make it unique for a combination of every Pool, recipient, and token sender.
- **Long term:** Create a system state specification with the state transition diagrams to document all the valid system states, specifically the intermediate temporary states. Follow the specification to ensure correct access control for all of the state transition functions.

---
