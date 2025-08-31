# Cluster -1216

**Rank:** #113  
**Count:** 152  

## Label
Centralized administrative control through single-point ownership enables unauthorized manipulation of critical functions, exposing systems to governance abuse, fund withdrawal, and irreversible attacks.

## Cluster Information
- **Total Findings:** 152

## Examples

### Example 1

**Auto Label:** Centralized administrative control through single-point ownership enables unauthorized manipulation of critical functions, exposing systems to governance abuse, fund withdrawal, and irreversible attacks.  

**Original Text Preview:**

##### Description
The issue has been identified within the [HanjiLOB](https://github.com/longgammalabs/hanji-contracts/blob/70b15ec4d9e7578248141604503843716a67d875/src/HanjiLOB.sol#L55) and [HanjiWatchDog](https://github.com/longgammalabs/hanji-contracts/blob/f699fdc9d9d74f699757cbdc8872f3d909ad5832/src/HanjiWatchDog.sol) contracts. 

The current design of the contract grants significant centralized control to the administrator and pauser roles. The administrator has extensive powers, including the ability to upgrade the contract to any arbitrary address, pause or unpause the contract, and configure critical settings. The pauser role can also pause and unpause the contract. This concentration of control introduces centralization risks, as a single compromised or malicious administrator or pauser could disrupt or manipulate the contract, leading to potential security breaches or operational failures. Additionally, allowing the pauser role to unpause the contract could be risky if the contract is in an unstable state, as it could inadvertently enable harmful operations.

Moreover, the owner of the HanjiWatchDog contract can upgrade the contract's implementation. If the new implementation is flawed or if the HanjiWatchDog contract is compromised, the creation of new orders in the HanjiLOB contract may be blocked.

The issue is classified as **medium** severity due to the potential impact on the contract's security and operational integrity if these roles are exploited.

##### Recommendation
We recommend using a multisig wallet for both the administrator and pauser roles. This would require multiple approvals for critical actions like upgrading the contract, pausing, or unpausing, thereby reducing the likelihood of malicious or erroneous actions. Additionally, we advise disabling the ability for the pauser role to unpause the contract, reserving this capability solely for the administrator role under the protection of a multisig wallet. This would help ensure that the contract can only be resumed under secure and verified conditions.

---
### Example 2

**Auto Label:** Centralized administrative control through single-point ownership enables unauthorized manipulation of critical functions, exposing systems to governance abuse, fund withdrawal, and irreversible attacks.  

**Original Text Preview:**

##### Description
This issue has been identified regarding the protocol’s upgradeability and the market maker’s control over user funds. 

**Admin Centralization**: As the contract is upgradeable, the admin can modify core protocol logic at any time. If a single, potentially compromised individual or entity controls this role, they could introduce malicious code or alter system parameters without user consent, leading to a total takeover of user funds.

**Market Maker Privilege**: The market maker can execute trades on its own LOB positions, potentially at rates harmful to liquidity providers. For instance, a 1%-below-market swap with an address under its control could drain a large portion of the pool’s value if made repeatedly. This risk grows if the market maker is less trusted than the admin but still has considerable influence.

The issue is classified as **high** severity because either the admin or the market maker can perform actions that jeopardize all participant funds, up to an almost complete loss.

##### Recommendation
We recommend:
1. Implementing a multisig with a timelock for critical administrative (upgrade) functions. This restricts any single party from pushing harmful updates immediately.
2. Introducing a safety deposit mechanism for the market maker. If the market maker’s trades cause protocol losses, they are first covered by its deposit. If losses approach the deposit limit, further market maker actions should be blocked, and the deposit seized if necessary. 

These measures reduce the risks of centralized exploitation and ensure the market maker is financially liable for damaging trades.


***

---
### Example 3

**Auto Label:** Centralized administrative control through single-point ownership enables unauthorized manipulation of critical functions, exposing systems to governance abuse, fund withdrawal, and irreversible attacks.  

**Original Text Preview:**

The constants `DEV_PUBKEY`, `ADMIN_PUBKEY`, and `MINT_AUTHORITY_PUBKEY` are all set to the same value:

```rust
pub const DEV_PUBKEY: Pubkey = pubkey!("BjjFpCbTrFVn3ZgcdCv4jTLAzbbDCMV1Vo115XJSJ7XG");
pub const ADMIN_PUBKEY: Pubkey = pubkey!("BjjFpCbTrFVn3ZgcdCv4jTLAzbbDCMV1Vo115XJSJ7XG");
pub const MINT_AUTHORITY_PUBKEY: Pubkey = pubkey!("BjjFpCbTrFVn3ZgcdCv4jTLAzbbDCMV1Vo115XJSJ7XG");
```

This design poses a risk because it centralizes all administrative privileges into a single key. If this key is compromised, an attacker could gain control over all administrative functions, including:

- Development operations (`DEV_PUBKEY`)
- Administrative actions (`ADMIN_PUBKEY`)
- Minting authority (`MINT_AUTHORITY_PUBKEY`)

If the shared key is compromised, the entire system is at risk. An attacker could perform unauthorized actions, such as minting tokens, modifying configurations, or accessing sensitive data.

Recommendation:

Assign unique public keys to `DEV_PUBKEY`, `ADMIN_PUBKEY`, and `MINT_AUTHORITY_PUBKEY` to ensure role separation.

---
