# Cluster -1314

**Rank:** #444  
**Count:** 8  

## Label
Unchecked input validation enables attackers to manipulate core economic parameters, distorting oracle data and violating protocol invariants with high impact on financial integrity and system stability.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Unauthorized deployment of unverified oracles or pool parameters, enabling attackers to inject faulty logic, manipulate invariants, or bypass ownership checks through spoofed or malicious contracts.  

**Original Text Preview:**

## MarginalV1LBLiquidityReceiverDeployer.sol

## Context
Line 25

## Description
The `MarginalV1LBLiquidityReceiverDeployer.deploy()` function allows deployment of a `MarginalV1LBLiquidityReceiver` contract. Access to the function is restricted by the `onlyPoolSupplier` modifier, which permits calling the function only from the supplier contract:

```solidity
modifier onlyPoolSupplier(address pool) {
    if (msg.sender != IMarginalV1LBPool(pool).supplier())
        revert Unauthorized();
    _;
}
```

The pool address is passed to the `MarginalV1LBLiquidityReceiverDeployer.deploy()` function by the caller. Thus, the restriction can be bypassed if the contract at the pool address implements the `supplier()` method and returns the address of the `MarginalV1LBLiquidityReceiverDeployer.deploy()` caller.

## Recommendation
Consider storing the address of the supplier contract in an immutable variable instead of trusting a pool to return it.

## Marginal
Fixed in commit 79c48742.

## Cantina Managed
Fix is verified.

---
### Example 2

**Auto Label:** Unauthorized deployment of unverified oracles or pool parameters, enabling attackers to inject faulty logic, manipulate invariants, or bypass ownership checks through spoofed or malicious contracts.  

**Original Text Preview:**

## Security Analysis of VeloDeployFactory

## Context
VeloDeployFactory.sol#L60

## Description
The Mellow protocol utilizes oracle `SecurityParams` to prevent price manipulation attacks. Currently, it is possible for the admin to set empty `SecurityParams` for a `tickSpacing` using the `updateDepositParams` function. Hence, it is also possible for the OPERATOR to deploy an `LpWrapper` for a `CLPool` with empty `SecurityParams`. That kind of `LpWrapper` will be susceptible to price manipulation attacks. Since only one `LpWrapper` can be deployed per `CLPool` and `LpWrapper` manages pooled funds of users, that scenario won't be ideal.

## Recommendation
Consider enforcing that the length of `DepositParams.SecurityParams` is not zero:

```solidity
function updateDepositParams(
    int24 tickSpacing,
    ICore.DepositParams memory params
) external {
    _requireAdmin();
    if (params.securityParams.length == 0) revert InvalidParams();
    _tickSpacingToDepositParams[tickSpacing] = params;
}
```

## Mellow
Fixed `736eef90`.

## Cantina Managed
The `VeloDeployFactory.createStrategy` now validates that `params.securityParams.length` is not zero.

---
### Example 3

**Auto Label:** Unchecked input validation enables attackers to manipulate core economic parameters, distorting oracle data and violating protocol invariants with high impact on financial integrity and system stability.  

**Original Text Preview:**

The [`ReserveFeed` contract](https://github.com/Ion-Protocol/ion-protocol/blob/98e282514ac5827196b49f688938e1e44709505a/src/oracles/reserve/ReserveFeed.sol#L4) tracks exchange rates for each collateral type. The [`setExchangeRate` function](https://github.com/Ion-Protocol/ion-protocol/blob/98e282514ac5827196b49f688938e1e44709505a/src/oracles/reserve/ReserveFeed.sol#L7) does not contain any access control or input validation. This would allow anyone to set any price for any collateral type.


Consider implementing access control on the `setExchangeRate` function to ensure that only trusted users can update the tracked rates.


***Update:** Resolved in [pull request #20](https://github.com/Ion-Protocol/ion-protocol/pull/20).*

---
