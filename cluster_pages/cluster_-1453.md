# Cluster -1453

**Rank:** #404  
**Count:** 12  

## Label
Insufficient input validation and improper state transitions enable attackers to manipulate fees, exploit timing, or bypass safeguards, leading to unauthorized revenue capture, overcharging, or denial of service.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Inadequate input validation and data handling lead to exploitable state manipulation, enabling attackers to manipulate prices, bypass security checks, or hijack critical system functions.  

**Original Text Preview:**

##### Description

The EMoney Appchain project is currently using an outdated version of the Cosmos SDK (v0.46.13). This version is affected by multiple security vulnerabilities of varying severity.

**Identified Vulnerabilities:**

1. ASA-2024-002 (GHSA-2557-x9mg-76w8)

- Severity: Moderate

- Description: Default `PrepareProposalHandler` may produce invalid proposals when used with default `SenderNonceMempool`

- Published: February 20, 2024

2. ASA-2024-003 (GHSA-4j93-fm92-rp4m)

- Severity: Moderate

- Description: Missing `BlockedAddressed` Validation in Vesting Module

- Published: February 20, 2024

3. ASA-2024-005 (GHSA-86h5-xcpx-cfqc)

- Severity: Low

- Description: Potential slashing evasion during re-delegation

- Published: February 27, 2024

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

Update the Cosmos SDK to the latest stable version that addresses all these vulnerabilities.

  

### Remediation Plan

**ACKNOWLEDGED:** The **Emoney team** acknowledged the issue.

##### References

[EMoney-Network/EMoneyChain/go.mod#L8](https://github.com/EMoney-Network/EMoneyChain/blob/872703eaad7051654bf13265516732850afd5aac/go.mod#L8)

[cosmos/cosmos-sdk](https://github.com/cosmos/cosmos-sdk/security)

---
### Example 2

**Auto Label:** Inadequate input validation and data handling lead to exploitable state manipulation, enabling attackers to manipulate prices, bypass security checks, or hijack critical system functions.  

**Original Text Preview:**

##### Description

The **Emoney AppChain** system lacks comprehensive **CosmosSDK** simulations and invariants for its modules. More complete use of the simulation feature would make it easier to fuzz test the entire blockchain and help ensure that invariants hold.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:L/A:N/D:N/Y:N/R:N/S:C (3.1)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:L/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

Eventually, extend the simulation module to cover all operations that can occur in a real Emoney Chain deployment, along with all possible error states, and run it many times before each release.

Make sure of the following:

***- All module operations are included in the simulation module.***

***- The simulation uses some accounts (e.g., between 5 and 20) to increase the likelihood of an interesting state change.***

***- The simulation uses the currencies/tokens that will be used in the production network.***

***- The simulation continues to run when a transaction fails.***

***- All paths of the transaction code are executed. (Enable code coverage to see how often individual lines are executed.)***

  

### Remediation Plan

**RISK ACCEPTED:** The **Emoney team** accepted the risk of the issue.

##### References

[EMoney-Network/EMoneyChain/tree/872703eaad7051654bf13265516732850afd5aac](https://github.com/EMoney-Network/EMoneyChain/tree/872703eaad7051654bf13265516732850afd5aac/x)

---
### Example 3

**Auto Label:** Insufficient input validation and improper state transitions enable attackers to manipulate fees, exploit timing, or bypass safeguards, leading to unauthorized revenue capture, overcharging, or denial of service.  

**Original Text Preview:**

## Difficulty: High

## Type: Patching

## Description
Some host operations for reading and writing to a WASM program’s memory charge ink before it is clear whether the operations will be successful or how much ink should really be charged.

For example, in the `read_return_data` function, the user is charged for the operation to write `size` bytes at the start of the host operation. However, the data to be written is the returned data of `size data.len()`, which could actually be smaller than the originally provided size. If `data.len()` is smaller than `size`, the user will be charged more ink than they should be.

```rust
pub(crate) fn read_return_data<E: EvmApi>(
    mut env: WasmEnvMut<E>,
    dest: u32,
    offset: u32,
    size: u32,
) -> Result<u32, Escape> {
    let mut env = WasmEnv::start(&mut env, EVM_API_INK)?;
    env.pay_for_write(size.into())?;
    let data = env.evm_api.get_return_data(offset, size);
    assert!(data.len() <= size as usize);
    env.write_slice(dest, &data)?;
    let len = data.len() as u32;
    trace!("read_return_data", env, [be!(dest), be!(offset)], data, len)
}
```

_Figure 5.1: Ink is charged for writing `size` bytes, even though the data to be written could be smaller than `size`. (arbitrator/stylus/src/host.rs#L273-L289)_

## Exploit Scenario
A WASM contract calls `read_return_data`, passing in a very large `size` parameter (100 MB). The EVM API, however, returns only 32 bytes, and the user is overcharged.

## Recommendations
- **Short term**: Modify the `read_return_data` function to require the user to have enough ink available for writing `size` bytes but to charge ink for writing only `data.len()` bytes. Make similar changes in all host operations that charge ink preemptively.
- **Long term**: Review the way ink is charged across different components and levels of abstraction. Make sure it is consistent and follows how the EVM works. Document any discrepancies in the charging of ink.

---
