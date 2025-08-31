# Cluster -1024

**Rank:** #457  
**Count:** 6  

## Label
Inconsistent function signatures and return types lead to incorrect interface IDs and function selectors, enabling interoperability failures and signature forgery through malformed or padded call data.

## Cluster Information
- **Total Findings:** 6

## Examples

### Example 1

**Auto Label:** Missing function calls due to interface mismatches cause transaction reverts, leading to denial of service and failure of critical business logic.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The function `pendingRedeemRequest()` tries to call `OmoVault.pendingRedeemRequest()`, but this function **does not exist in OmoVault**. This will cause the transaction to revert.

```solidity
    function pendingRedeemRequest(
        uint256 vaultId,
        uint256 requestId,
        address owner
    ) external view returns (uint256 pendingShares) {
        address vault = vaultFactory.getVault(vaultId);
        return IOmoVault(vault).pendingRedeemRequest(requestId, owner);
    }
```

## **Recommendations**

add `pendingRedeemRequest` function to `OmoVault`.

---
### Example 2

**Auto Label:** Missing function calls due to interface mismatches cause transaction reverts, leading to denial of service and failure of critical business logic.  

**Original Text Preview:**

**Severity**: Medium

**Status**: Resolved

**Description**

In the SideMarginFundingAccount contract, the marginAccountDepositFromFundingAccount() function, which is called on line: 132, does not exist in the TradeableSideVault contract. This can lead to the function reverting, always leading to incorrect execution of function than intended and thus leading to Denial of Service.

**Recommendation**: 

It is advised to define the function properly in the expected contract and review business and operational logic.

---
### Example 3

**Auto Label:** Inconsistent function signatures and return types lead to incorrect interface IDs and function selectors, enabling interoperability failures and signature forgery through malformed or padded call data.  

**Original Text Preview:**

#### Resolution



After the official end of this engagement, the community has come to the decision that `is_valid_signature` should return the StarkNet constant `VALIDATED` (which is a `felt252` that represents the string `'VALID'`) in the affirmative case and `0` otherwise. Therefore, the interface ID to be returned is now set to be `0x2ceccef7f994940b3962a6c67e0ba4fcd37df7d131417c604f91e03caecc1cd`. The Argent team has changed the implementation accordingly and provided us with the updated commit hash. We have reviewed the changes and updated the final commit hash of the report to this new version.


#### Description


As an analog to Ethereum Improvement Proposals (EIPs), there are StarkNet Improvement Proposals (SNIPs), and SNIP-5 – similar in intention and technique to ERC-165 – defines how to publish and detect what interfaces a smart contract implements. As in ERC-165, this is achieved with the help of an *interface identifier*.


Specifically, this ID is defined as the XOR of the “extended function selectors” in the interface. While not going into all the details here, a function’s extended selector is the `starknet_keccak` hash of its signature, where some special rules define how to deal with the different data types. The details can be found in the proposal. Compliant contracts implement a `supports_interface` function that takes a `felt252` and returns `true` if the contract implements the interface with this ID and `false` otherwise. For example, `argent_account.cairo` defines the following `supports_interface` function:


**contracts/account/src/argent\_account.cairo:L506-L522**



```
fn supports\_interface(self: @ContractState, interface\_id: felt252) -> bool {
 if interface\_id == ERC165\_IERC165\_INTERFACE\_ID {
 true
 } else if interface\_id == ERC165\_ACCOUNT\_INTERFACE\_ID {
 true
 } else if interface\_id == ERC165\_OUTSIDE\_EXECUTION\_INTERFACE\_ID {
 true
 } else if interface\_id == ERC165\_IERC165\_INTERFACE\_ID\_OLD {
 true
 } else if interface\_id == ERC165\_ACCOUNT\_INTERFACE\_ID\_OLD\_1 {
 true
 } else if interface\_id == ERC165\_ACCOUNT\_INTERFACE\_ID\_OLD\_2 {
 true
 } else {
 false
 }
}

```
In this issue, we’re interested in `ERC165_ACCOUNT_INTERFACE_ID`, which is defined as follows:


**contracts/lib/src/account.cairo:L3-L4**



```
const ERC165\_ACCOUNT\_INTERFACE\_ID: felt252 =
 0x32a450d0828523e159d5faa1f8bc3c94c05c819aeb09ec5527cd8795b5b5067;

```
This ID corresponds to an interface with the following function signatures:



```
 fn \_\_validate\_\_(Array<Call>) -> felt252;
 fn \_\_execute\_\_(Array<Call>) -> Array<Span<felt252>>;
 fn is\_valid\_signature(felt252, Array<felt252>) -> bool;

```
Note that `is_valid_signature` returns a `bool`. However, in the actual `IAccount` interface, this function returns a `felt252`:


**contracts/lib/src/account.cairo:L10-L17**



```
// InterfaceID: 0x32a450d0828523e159d5faa1f8bc3c94c05c819aeb09ec5527cd8795b5b5067
trait IAccount<TContractState> {
 fn \_\_validate\_\_(ref self: TContractState, calls: Array<Call>) -> felt252;
 fn \_\_execute\_\_(ref self: TContractState, calls: Array<Call>) -> Array<Span<felt252>>;
 fn is\_valid\_signature(
 self: @TContractState, hash: felt252, signatures: Array<felt252>
 ) -> felt252;
}

```
If we check out the implementation of `is_valid_signature`, we see that it returns the magic value `0x1626ba7e` known from ERC-1271 if the signature is valid and `0` otherwise:


**contracts/account/src/argent\_account.cairo:L214-L222**



```
fn is\_valid\_signature(
 self: @ContractState, hash: felt252, signatures: Array<felt252>
) -> felt252 {
 if self.is\_valid\_span\_signature(hash, signatures.span()) {
 ERC1271\_VALIDATED
 } else {
 0
 }
}

```
**contracts/lib/src/account.cairo:L8**



```
const ERC1271\_VALIDATED: felt252 = 0x1626ba7e;

```
The ID for this interface would be `0x2ceccef7f994940b3962a6c67e0ba4fcd37df7d131417c604f91e03caecc1cd`. Note that, unlike in ERC-165, in SNIP-5, the return type of a function does matter for the interface identifier. Hence, the actual `IAccount` interface defined and implemented and the published interface ID do not match.


#### Recommendation


At the end of this engagement, the community has not come to a decision yet whether `is_valid_signature` should return a `bool` or a `felt252` (i.e., the magic value `0x1626ba7e` in the affirmative case and `0` otherwise). Depending on the outcome, either the actual interface and its implementation or the published interface ID must be changed to achieve consistency between the two.


#### Remark


SNIP-5 is not very clear on how to deal with the new Cairo syntax introduced in v2.0.0 of the compiler. Specifically, with this new syntax, interface traits have a generic parameter `TContractState`, and all non-static functions in the interface have a first parameter `self` of type `TContractState` or `@TContractState` for `view` functions. How to deal with this parameter in the derivation of the interface identifier is not (yet) explicitly specified in the proposal, but the Argent team has assured us that the understanding in the community is to ignore this parameter for the extended function selectors and hence the interface ID.

---
