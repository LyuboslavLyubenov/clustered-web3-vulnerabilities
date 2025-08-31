# Cluster -1233

**Rank:** #279  
**Count:** 29  

## Label
Failure to provide fund withdrawal mechanisms leads to permanent, irrecoverable loss of assets due to lack of exit paths, exposing contracts to financial loss and undermining user trust and liquidity.

## Cluster Information
- **Total Findings:** 29

## Examples

### Example 1

**Auto Label:** Failure to provide fund withdrawal mechanisms leads to permanent, irrecoverable loss of assets due to lack of exit paths, exposing contracts to financial loss and undermining user trust and liquidity.  

**Original Text Preview:**

## Diﬃculty: Low

## Type: Undeﬁned Behavior

## Description
The Treasury contract currently lacks functionality for withdrawing ERC-20 tokens; any such tokens sent to the treasury would be permanently locked in the contract. The Otim protocol allows users to pay fees with either native ETH or any ERC-20 token registered by Otim. These fees are directed to the Treasury contract.

```solidity
function withdraw(address target, uint256 value) external onlyOwner {
    // check the target is not the zero address
    if (target == address(0)) {
        revert InvalidTarget();
    }

    // check the contract has enough balance to withdraw
    if (value > address(this).balance) {
        revert InsufficientBalance();
    }

    // withdraw the funds
    (bool success, bytes memory result) = target.call{value: value}("");
    
    // check if the withdrawal was successful
    if (!success) {
        revert WithdrawalFailed(result);
    }
}
```
*Figure 4.1: The withdraw function in Treasury.sol*

However, the Treasury contract currently supports only ETH withdrawals via the withdraw function; it lacks a function for withdrawing ERC-20 tokens.

## Exploit Scenario
The Otim protocol accepts USDC as a fee token; these fees accumulate in the Treasury contract over time. When the protocol team attempts to access these funds, they discover that there is no mechanism to withdraw ERC-20 tokens from the treasury. The USDC becomes permanently locked in the contract, resulting in financial losses for the protocol.

## Recommendations
- Short term: Add a function to withdraw ERC-20 tokens from the Treasury contract.
- Long term: Develop test cases specifically for token recovery scenarios. In addition, review how funds flow between users and components and document these flows comprehensively to improve system security.

---
### Example 2

**Auto Label:** Failure to provide fund withdrawal mechanisms leads to permanent, irrecoverable loss of assets due to lack of exit paths, exposing contracts to financial loss and undermining user trust and liquidity.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The `GigaNameNFT` contract is supposed to receive ETH when the minter role calls `mintUsername()` to mint a **GigaNameNFT** for a player, however, this contract doesn't implement a mechanism to enable the manager from withdrawing collected ETH (nor does the parent `GameNFT` contract), which would result in permanently locking the collected ETH in the contract.

## Recommendations

Update `GigaNameNFT` contract to implement a withdrawal mechanism for the collected ETH.

---
### Example 3

**Auto Label:** Failure to provide fund withdrawal mechanisms leads to permanent, irrecoverable loss of assets due to lack of exit paths, exposing contracts to financial loss and undermining user trust and liquidity.  

**Original Text Preview:**

**Severity:** Low

**Path:** src/AaveLendingModule:withdraw#L75-L77

**Description:** The STEXAMM leverages supplying token1 (WETH) collateral to a lending platform (Aave) to earn extra yield. The depositing happens manually by the owner, but the withdrawing happens automatically upon a call to `STEXAMM.sol:withdraw` to fulfil the callers withdrawal.

The code assumes that the withdraw function of Aave can always return the provided collateral but this is not the case if the liquidity is being borrowed on Aave. Aave often utilises a utilisation factor of 90%, which means that only 10% of the supplied liquidity can be withdrawn at the time. If a higher amount is requested, it would simply revert.

Since HyperEVM is a new chain, we can assume that the STEXAMM will be a major supplied of wHYPE to Aave on HyperEVM and potentially hold more than 10% of the supplied liquidity. In that case, larger withdrawals could result in funds not being withdrawable and temporary insolvency/DoS.
```
    function withdraw(uint256 _amount, address _recipient) external onlyOwner {
        pool.withdraw(asset, _amount, _recipient);
    }
```

**Remediation:**  The function `supplyToken1ToLendingPool` allows the owner to withdraw all liquidity from the SovereignPool and supply it to Aave, we recommend that this function should be capped by a sensible percentage of the total reserves to reduce the chance of Aave blocking withdrawals.

**Status:** Acknowledged

- - -

---
