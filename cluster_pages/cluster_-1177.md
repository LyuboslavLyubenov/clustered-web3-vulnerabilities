# Cluster -1177

**Rank:** #302  
**Count:** 24  

## Label
Unauthorized administrative control enabling unchecked state modifications, fund manipulation, and loss of user safeguards through insufficient access checks and lack of validation or transparency.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Privilege escalation via unchecked admin access enables unauthorized fund drainage and liquidity manipulation, compromising system integrity and user capital safety.  

**Original Text Preview:**

## fatBERA Contract Withdraw Principal Function Analysis

## Context
`fatBERA.sol#L190-L196`

## Description
The function `withdrawPrincipal` allows the admin to withdraw principal assets from the contract, thereby reducing the value of `depositPrincipal`. However, once withdrawals and redeems are enabled, this functionality can introduce significant issues. If users attempt to withdraw their deposits while the contract's deposit principal has been reduced through admin withdrawals, an underflow may occur when trying to deduct the withdrawal amount from `depositPrincipal`. This would cause the transaction to revert.

Even before an underflow occurs, another failure mode is possible: the contract may simply not have enough of the underlying asset available to transfer to withdrawing users. Since withdrawals rely on `safeTransfer`, any missing balance in the contract due to prior principal withdrawals would cause withdrawal transactions to fail. Users expecting to redeem their shares may find that liquidity is missing, leading to a non-functional withdrawal mechanism.

## Recommendation
The contract should ensure that `withdrawPrincipal` does not allow the admin to withdraw funds that are needed for future withdrawal requests. Implementing a minimum reserve check would prevent principal withdrawals from depleting the assets required to satisfy user withdrawals. A function to compute the maximum safe principal withdrawal based on current deposits and expected withdrawal demand would also mitigate this issue. If `withdrawPrincipal` is only meant for exceptional circumstances, additional restrictions should be put in place to clarify its intended use and prevent liquidity depletion.

## HoneyJar
Withdrawing will most likely call a withdraw on the validator itself instead of transferring from the fatBERA contract. The current withdraw functions were there for testing. I should have just had them do nothing to avoid confusing users. We will be completely rewriting the withdraw and redeem functions when they are enabled; we just don't know the withdrawal patterns yet. `withdrawPrincipal` will be called frequently by our validator operator so that they can deposit it into one of the validators we are running.

## Cantina Managed
Acknowledged.

---
### Example 2

**Auto Label:** Unauthorized administrative control enabling unchecked state modifications, fund manipulation, and loss of user safeguards through insufficient access checks and lack of validation or transparency.  

**Original Text Preview:**

## Summary

`Collateral.Data` information is saved in two locations. Protocol access this information from both places but admin can update only one of them resulting in a DoS for admin rights.

## Vulnerability Details

Collateral data is handled by [Collateral](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Collateral.sol#L14) library and saved at `COLLATERAL_LOCATION` storage location. It is configured by [MarketMakingEngineConfigurationBranch ::configureCollateral](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/MarketMakingEngineConfigurationBranch.sol#L505-L513) function and can be called multiple times by `admin`.

Same data structure is stored is [Vault](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Vault.sol#L107) leaf and is part of the bigger vault's data struct stored at `VAULT_LOCATION` storage location.
It is initialized when the vault is [created](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Vault.sol#L464-L480). If the vault exist this function [can't be called again](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/leaves/Vault.sol#L467-L469). The vault's `self.collateral` data can't be updated.

```solidity
    function create(CreateParams memory params) internal {
        Data storage self = load(params.vaultId);


        if (self.id != 0) {
            revert Errors.VaultAlreadyExists(params.vaultId);
        }
        self.id = params.vaultId;
        self.depositCap = params.depositCap;
        self.withdrawalDelay = params.withdrawalDelay;
        self.indexToken = params.indexToken;
@>        self.collateral = params.collateral; // @audit set collateral data
        self.depositFee = params.depositFee;
        self.redeemFee = params.redeemFee;
        self.engine = params.engine;
        self.isLive = true;
    }
```

Two critical information are accessed from `colllateral.Data` stored in `vault`:

* `isEnable` status in [VaultRouterBranch::initiateWithdrawal](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L438-L440) and [deposit](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L290)
* `priceAdapter` in [VaultRouterBranch::getVaultCreditCapacity](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/VaultRouterBranch.sol#L81-L82) and [StabilityBranch::initiateSwap](https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/StabilityBranch.sol#L233)

In case admin wants to update `priceAdapter` (eg. in case of a vulnerability in existing adaptor) or to disable the collateral, he can't.
The protocol can reach a situation where an asset is disabled in `market-making-engine` (via  `configureCollateral`) while is still active in a Vault.
Same for `priceAdapter`, it can have an adapter in a specific vault and a different adapter to be configured in engine for same asset.
In case of a vulnerable adaptor the vault must be shut down and re-created.

## Impact

Key vault parameters are inaccessible for admin to update, resulting the relaunch the vault and the associated  hassle.

## Tools Used

## Recommendations

There are two options. Either remove `Collateral.data` from Vault structure and update the code where required to interogate collateral data only from `COLLATERAL_LOCATION`.
Or, at least, add a new function (or update existing `Vault::update`) to allow admin to update these two parameters.

---
### Example 3

**Auto Label:** Unauthorized administrative access enabling arbitrary asset transfers, bypassing validation and access controls, leading to severe fund drain and system compromise.  

**Original Text Preview:**

The protocol is subject to the following asset transfer risks which become a threat in case the `DEFAULT_ADMIN_ROLE` is compromised / acts maliciously:

- Arbitrary transfer of ETH and ERC20s from `Treasury` contract via direct transfer or allowances.
- Arbitrary transfer of ETH, ERC721s and ERC20s from `LidoTreasuryConnector` contract.
- Arbitrary transfer of ETH and ERC20s from `AaveV3TreasuryConnector` contract.
- Arbitrary transfer of ERC20s from `LPExternalRequestsManager` contract.

---
