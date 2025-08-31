# Cluster -1053

**Rank:** #165  
**Count:** 79  

## Label
Inadequate access control and inconsistent governance authorization lead to unauthorized function execution, permission gaps, and asset manipulation, enabling attackers to compromise token ownership, freeze funds, or exploit governance mechanisms.

## Cluster Information
- **Total Findings:** 79

## Examples

### Example 1

**Auto Label:** Failure to enforce initialization or finalization states allows unauthorized state modifications, leading to unintended behavior, loss of funds, or compromised immutability in critical contract operations.  

**Original Text Preview:**

The `internal` function `_init` within `ERC721WithURIBuilderUpgradeable.sol` lacks the `onlyInitializing` modifier. This function is called by the `initializer` function (`init`) of the upgradeable implementation contract `PositionsManager.sol`.

While the main `PositionsManager.init` function _is_ protected by the `initializer` modifier, and the underlying `__ERC721_init` called by `_init` also has protection, the absence of the guard on `_init` itself poses a latent risk. If future upgrades to `PositionsManager` were to introduce new functions that mistakenly call `_init` again after initial deployment, there would be no direct protection at the `_init` level to prevent this call attempt.

Recommendation:

Add the `onlyInitializing` modifier to the `ERC721WithURIBuilderUpgradeable._init` function to explicitly enforce its initialization-only context and adhere to standard upgradeable contract patterns.

```solidity
    function _init(string memory name_, string memory symbol_) internal onlyInitializing {
        __ERC721_init(name_, symbol_);
    }
```

---
### Example 2

**Auto Label:** Hardcoded seeds and missing access control enable unauthorized initialization, allowing attackers to seize governance authority and manipulate system state.  

**Original Text Preview:**

The following unresolved `TODOs` introduce crucial issues in deployment and configuration logic:

1. Hardcoded `address(0)` as a proxy admin in `DeployUsdxlUtils._deployGsm()`. Currently, the proxy admin is hardcoded as `address(0)`, meaning no one can manage upgrades or administrative functions of the proxy.

```solidity
//File: (usdxl-core) src/deployments/utils/DeployUsdxlUtils.sol

function _deployGsm() internal returns (address) {
    AdminUpgradeabilityProxy proxy = new AdminUpgradeabilityProxy(
        address(gsmImpl),
@>      address(0), // TODO: set admin to timelock
        ""
    );
    --- SNIPPED ---
}
```

2. Duplicate strategy contracts in `DeployHyFiConfigEngine._getUniqueStrategiesOnPool()`.

```solidity
//File: (hypurrfi-deployment) script/DeployHyFiConfigEngine.s.sol

library DeployRatesFactoryLib {
@>   // TODO check also by param, potentially there could be different contracts, but with exactly same params
    function _getUniqueStrategiesOnPool(IPool pool, address[] memory reservesToSkip) {...}
```

The function currently checks for duplicate strategies only by contract address, but not by actual parameters. However, in `V3RateStrategyFactory.initialize()`, strategies are identified using a hash of their parameters. This means the same configuration can be registered multiple times under different contracts, leading to unnecessary duplication.

```solidity
//File: (hypurrfi-deployment) lib/aave-helpers/src/v3-config-engine/V3RateStrategyFactory.sol

function initialize(IDefaultInterestRateStrategy[] memory liveStrategies) external initializer {
for (uint256 i = 0; i < liveStrategies.length; i++) {
    RateStrategyParams memory params = getStrategyData(liveStrategies[i]);

    bytes32 hashedParams = strategyHashFromParams(params);

@>  _strategyByParamsHash[hashedParams] = address(liveStrategies[i]);
@>  _strategies.push(address(liveStrategies[i]));

    emit RateStrategyCreated(address(liveStrategies[i]), hashedParams, params);
}
}
```

Recommendation

- For `DeployUsdxlUtils._deployGsm()` function:
  If the proxy admin is meant to be a contract (such as a timelock contract), deploy it as part of the script and assign it properly. Otherwise, pass the proxy admin address as a parameter to `_deployGsm()` instead of hardcoding `address(0)`.

- For `DeployHyFiConfigEngine._getUniqueStrategiesOnPool()`:
  Before adding a new unique strategy, check if another strategy with the same parameters already exists.

---
### Example 3

**Auto Label:** Failure to enforce proper initialization or ownership transfer leads to unauthorized control, arbitrary parameter setting, or missing governance oversight, enabling attackers to manipulate contract state or exploit governance gaps.  

**Original Text Preview:**

The `_deployUsdxl()` function is designed to deploy the **usdxl token** and the required contracts to initialize the **usdx reserve**, however, it was noticed that when the **`usdxlAToken`** is deployed, it is not initialized in the script, which allows any malicious actor to initialize it with unintended, incorrect, or irrelevant parameters as the `usdxlAToken.initialize()` function is unrestrictred.

Recommendation: ensure that the **`usdxlAToken`** is properly initialized within the script during the deployment process.

---
