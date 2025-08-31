# Cluster -1331

**Rank:** #372  
**Count:** 15  

## Label
Arbitrary token issuance without supply constraints enables centralization, supply manipulation, and bridge inconsistencies, allowing malicious actors to create or alter token supply beyond authorized limits.

## Cluster Information
- **Total Findings:** 15

## Examples

### Example 1

**Auto Label:** Unauthorized minting leading to denial-of-service and economic imbalance through unchecked access and weak restrictions on token issuance.  

**Original Text Preview:**

## Security Vulnerability Report

## Severity
**High Risk**

## Context
`MintController.sol#L99`

## Description
The mint function in the `InfiniFiGatewayV1` contract allows anyone to mint iUSD receipt tokens to an arbitrary address specified by the `_to` parameter. This functionality introduces a significant griefing vector that can disrupt user and contract operations within the system. Specifically, each call to mint triggers an external call to 

```solidity
ReceiptToken(receiptToken).restrictActionUntil(_to, block.timestamp + restrictionDuration);
```

which restricts the target address from performing certain actions, such as transferring receipt tokens, for a period defined by the `restrictionDuration` state variable. The `restrictionDuration` is currently set to a minimal value of 1, making the cost of repeatedly invoking this function extremely low.

This design enables an attacker to grief users and contracts by minting small amounts of iUSD to targeted addresses, thereby repeatedly extending their restriction periods. For users, this can block operations like `mintAndStake`, `mintAndLock`, and `createPosition` in the `InfiniFiGatewayV1` contract, as well as interfere with voting mechanisms. These operations can be front-run by an attacker calling 

```solidity
mint(<InfiniFiGatewayV1 address>, 1);
```

causing subsequent user transactions to fail if they involve restricted actions.

Additionally, this vulnerability impacts contracts that transfer iUSD, such as the `YieldSharing` contract. In the `accrue` function, if a positive yield is detected, `_handlePositiveYield` is called, which includes a transfer of receipt tokens to a performance fee recipient:

```solidity
ReceiptToken(receiptToken).transfer(performanceFeeRecipient, fee);
```

If an attacker mints iUSD to the `YieldSharing` contract address (e.g., via `mint(<YieldSharing address>, 1)`), the contract becomes restricted, and this transfer fails, effectively causing a temporary denial of service (DoS) for the `accrue` function. An attacker could front-run every accrue call with such a mint, delaying or blocking yield processing indefinitely.

The root of this issue lies in the `ReceiptToken` contract's `_update` function, which enforces action restrictions during token transfers:

```solidity
function _update(address _from, address _to, uint256 _value) internal override {
    if (_from != address(0) && _to != address(0)) {
        // check action restrictions if the transfer is not a burn nor a mint
        _checkActionRestriction(_from);
    }
    return ERC20._update(_from, _to, _value);
}
```

This function checks if the sender (`_from`) is restricted whenever a transfer occurs (excluding minting and burning). If restricted, the transfer reverts, halting any operation dependent on it. Since both `LockedPositionTokens` and `StakedTokens` inherit from `ActionRestriction`, they are similarly vulnerable if targeted by minting, potentially disrupting their respective functionalities.

In summary, the ability to mint iUSD to any address allows an attacker to impose repeated restrictions with minimal cost, denying users access to key operations and causing temporary DoS in contracts that rely on transferring iUSD.

## Recommendation
Consider removing the possibility of minting to other addresses and always mint directly to the caller. Alternatively, if minting to other addresses is a required feature, consider increasing the `minMintAmount` to a much higher value so the griefing vector is not feasible due to the increased costs. Finally, consider having a set of whitelisted addresses that will not suffer the transfer restriction constrain and add all the protocol addresses to this whitelist.

## Fixes
- **infiniFi**: Fixed in commit `e893df4` by removing `ActionRestriction` from iUSD and siUSD tokens.
- **Spearbit**: Fix verified.

---
### Example 2

**Auto Label:** Missing access control enables unauthorized minting, allowing attackers to bypass validation and issue tokens without proper authorization or identity checks.  

**Original Text Preview:**

##### Description

The `mint_stable_coin` function allows to mint a certain amount of stable coins depending on the amount of collateral deposited by the user. The validations are performed by the **Custody** contract, verifying that the user has already deposited the corresponding collateral, however, this control can be bypassed by directly calling this entry point in the **Central Control** contract.

The access control of the `mint_stable_coin`function is not correctly implemented. It is a nested "if" where the first condition checks if `info.sender` is different from the Custody contract and the second checks if it is equal to the `minter`value.

Since the `minter`value is an input included in the message, it can be set by the attacker to the same value as `info.sender` and bypass the access control. After this, there is no validation on `collateral_amount`and `collateral_contract` values, so the minting process is performed with any amount chosen by the attacker.

This is the affected code snippet:

![Vulnerable code](https://halbornmainframe.com/proxy/audits/images/6602ad6353b13d194e08efaa)

##### Proof of Concept

Attacker transaction:

```
injectived tx wasm execute <CDP_CENTRAL_CONTROL_ADDRESS>'{"mint_stable_coin":{"minter":"<USER_ADDRESS", "stable_amount": "1000", "collateral_amount": "1000", "collateral_contract": "<COLLATERAL_CONTRACT>"}}' --chain-id <CHAIN_ID> --node <NETWORK> --from user  --gas=auto --gas-prices=1000000000000inj
```

Executed on testnet:

```
injectived tx wasm execute inj1aflk2ftzem4emzjmv5ve27af4m3n0ggsz6z45w '{"mint_stable_coin":{"minter":"inj1ph5h805jd4cmgj2jemhgln9ssjkk744ca44avl", "stable_amount": "1000", "collateral_amount": "1000", "collateral_contract": "inj1t4uk4ntn3l0wa7dcqsjl5enenzaaer2alrfmcm"}}' --chain-id injective-888 --node https://testnet.sentry.tm.injective.network:443 --from user  --gas=auto --gas-prices=1000000000000inj 
```

Link to the transaction in the explorer:

<https://testnet.explorer.injective.network/transaction/0x43edfd119a82e0787f7af20e9b88c849a88b6cdfef0950ec6c64964cc2cb98c2/>

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:C/Y:N/R:N/S:U (10.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:C/Y:N/R:N/S:U)

##### Recommendation

It is recommended to modify the access control by separating the nested "if" into two conditions to avoid being bypassed, or to include more controls regarding the amount of the collateral and the collateral contract after the first access control.

Remediation Plan
----------------

**SOLVED:** The condition `if sender_raw == config.custody_contract` has been added before storing the collateral information included in the message.

##### Remediation Hash

<https://github.com/GryphonProtocolDAO/gryphon-cdp-contracts/commit/65e51d51bf9b0995fd6f61917a5505ecf8d6d9eb>

---
### Example 3

**Auto Label:** Unrestricted minting authority enables centralized control, allowing unauthorized token issuance, inflation, and loss of supply integrity through inadequate access checks and lack of governance validation.  

**Original Text Preview:**

**Severity**: Informational	

**Status**: Acknowledged

**Description**

The `mint` function is only available for 2 defined addresses, `LITCRAFT_MINTER` and `FOREVVER_MINTER`. These are the only 2 addresses that can mint tokens. Having a centralized scenario can imply several risks as losing the control of one of the addresses and not being able to mint more tokens.

---
