# Cluster -1156

**Rank:** #107  
**Count:** 159  

## Label
Insufficient bounds checking and error handling enable attackers to trigger denial-of-service attacks, inflate gas costs, or exploit memory copying, leading to system stalls, fund drain, or unauthorized operations.

## Cluster Information
- **Total Findings:** 159

## Examples

### Example 1

**Auto Label:** **Improper bounds checking leads to unbounded or under-distributed token issuance, enabling supply overflow, over-issuance, and loss of control.**  

**Original Text Preview:**

`maxMint` currently returns `type(uint256).max`, which is incorrect. When users deposit to the vault, it checks if the `assets` deposited do not exceed `maxPerDeposit`. Instead of returning `type(uint256).max`, `maxMint` should return the shares corresponding to `maxPerDeposit`.

```diff
    function maxMint(address) public view virtual override returns (uint256) {
-        return type(uint256).max;
+        _convertToShares(maxPerDeposit, Math.Rounding.Down);
    }
```

---
### Example 2

**Auto Label:** Missing validation and bounds checks in state mutations enable unauthorized over-issuance or overflow attacks, leading to fund loss or protocol failure.  

**Original Text Preview:**

## Difficulty: Low

## Type: Testing

## Description

The YTMinter contract owner can update the storage values of the `sy_minter_address` and `sy_wallet_address` variables using two different operations. Not keeping these variables in sync can result in users losing funds.

The YTMinter contract allows the owner to update the `sy_minter_address` with the `change_sy_minter_address` operation:

```cpp
if (op == op::change_sy_minter_address) {
    check_sender_is_owner(sender_address);
    storage::sy_minter_address = in_msg_body~load_msg_addr();
    save_data();
    throw(successExitCode);
}
```
*Figure 4.1: The admin function to update the SY minter configuration*  
*contracts/YT/maintance.fc#L135-L140*

The YTMinter contract also allows the owner to update the `sy_wallet_address` with the `change_jetton_addresses` operation:

```cpp
if (op == op::change_jetton_addresses) {
    check_sender_is_owner(sender_address);
    cell new_jetton_addresses = in_msg_body~load_ref();
    slice jetton_addresses = new_jetton_addresses.begin_parse();
    if (jetton_addresses.slice_empty?() == 0) {
        storage::sy_wallet_address = jetton_addresses~load_msg_addr();
        storage::pt_minter_address = jetton_addresses~load_msg_addr();
        storage::pt_wallet_address = jetton_addresses~load_msg_addr();
        save_data();
    }
    throw(successExitCode);
}
```
*Figure 4.2: The admin function to update Jetton wallet addresses*  
*contracts/YT/maintance.fc#L38-L49*

The SYMinter and SYWallet contracts are interconnected contracts, and their stored values in the YTMinter contracts must remain in sync. If the owner updates the `sy_minter_address` without updating the `sy_wallet_address` value, users can lose funds from some actions.

Specifically, when a user transfers their underlying asset to the SYMinter contract with the `wrap_and_mint_pt_yt` operation, then the SY tokens are transferred to the YTMinter contract with the `mint_pt_yt` operation. If the user wraps and mints from a matured YTMinter, and the YTMinter owner updates the `sy_minter_address` in the time between the `internal_transfer` message reaches the YT minter contract’s SYWallet contract and the `transfer_notification` message reaches the YTMinter contract, then the SY tokens are sent back to the updated `sy_minter_address`. This will lead to loss of funds because the updated SYMinter contract will not authorize the SYWallet associated with the old SYMinter contract.

## Exploit Scenario

The YTMinter owner updates the `sy_minter_address` but does not update the `sy_wallet_address`. A user tries to mint a mature YT token and loses her funds.

## Recommendations

- **Short term:** Update the owner functions to update the `sy_minter_address` and `sy_wallet_address` from a single function.
- **Long term:** Review all the admin functions to ensure that token configurations remain in sync and other configuration changes do not break the system state.

---
### Example 3

**Auto Label:** **Improper bounds checking leads to unbounded or under-distributed token issuance, enabling supply overflow, over-issuance, and loss of control.**  

**Original Text Preview:**

##### Description
This issue has been identified within the `_validateCap` function of the `CapRiskSteward` contract.
If a vault currently does not have a limit, the function `AmountCap.wrap(currentCap).resolve()` returns `type(uint256).max`. Consequently, when calculating `currentCapResolved * allowedAdjustFactor / 1e18`, the operation [will always revert](https://github.com/euler-xyz/evk-periphery/blob/a438c1e910f7292adbd527b4ea72742d3f736f23/src/Governor/CapRiskSteward.sol#L134) due to an overflow.
The issue is classified as **Low** severity because it can cause unintended revert, preventing valid cap adjustment.
<br/>
##### Recommendation
We recommend implementing a check to handle scenarios where `currentCap` is not set.

> **Client's Commentary:**
> In the no limit case, there could be no valid cap adjustment, because the domain of caps is [0, uint112.max] U {uint256.max}. Any sane adjustment factor (2^60-2^70) would not allow adjusting from uint256.max to the sane range regardless.
> 
> Anyway, we are going to fix this by handling the unlimited scenario specifically, for the purposes of code cleanliness.


---

---
