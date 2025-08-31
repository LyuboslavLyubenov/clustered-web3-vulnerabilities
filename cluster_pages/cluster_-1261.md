# Cluster -1261

**Rank:** #157  
**Count:** 87  

## Label
Predictable contract addresses enable front-running and arbitrage attacks, allowing attackers to manipulate token prices or exploit pool creation timing through deterministic address generation and insufficient validation.

## Cluster Information
- **Total Findings:** 87

## Examples

### Example 1

**Auto Label:** Deterministic address derivation flaws enable attackers to predict, collide, or reuse contract addresses, leading to pool misidentification, front-running, fund theft, or transaction failures.  

**Original Text Preview:**

## Severity

High Risk

## Description

An attacker can exploit the deterministic nature of contract deployment using the `CREATE` opcode to precompute the address of the `DaosToken` deployed within the `DaosLive::finalize()` function. This predictability enables a front-running attack that severely impacts the liquidity and valuation of the token in Uniswap v3 pools.

The `DaosToken` is deployed via the `CREATE` opcode inside the `finalize()` function of the `DaosLive` contract. The use of `CREATE` makes the address of the deployed contract predictable using the sender address and nonce:

> **Source:** [evm.codes](https://www.evm.codes/#f0)  
> The destination address is calculated as the rightmost 20 bytes (160 bits) of the Keccak-256 hash of the RLP encoding of the sender address, followed by its nonce.  
> `address = keccak256(rlp([sender_address, sender_nonce]))[12:]`

By leveraging this predictability, an attacker can execute the following exploit sequence:

---

## Exploit Scenario

1. The attacker precomputes the future address of the `DaosToken` contract as it is deployed via the `CREATE` opcode using this: `keccak256(rlp([deployer_address, deployer_nonce]))[12:]`.

2. The attacker creates a Uniswap v3 liquidity pool between WETH and the precomputed `DaosToken` address using the same fee tier and tick range expected in the `finalize()` function.

3. The attacker adds liquidity to the newly created pool by supplying a very small amount of WETH and a large amount of spoofed `DaosToken`.

4. The spoofed token is accepted by Uniswap because the address exists even though the actual contract code is not yet deployed.

5. The attacker waits for the `DaosLive::finalize()` function to be executed by the protocol.

6. Upon execution, the `DaosToken` contract is deployed at the precomputed address, and the protocol adds a significant amount of real `DaosToken` and WETH as liquidity to the same pool.

7. The attacker removes their liquidity position from the pool, receiving a large share of the real `DaosToken` just added by the protocol.

8. The attacker then swaps all the extracted `DaosToken` for WETH in the same pool.

9. The swap operation drains a significant portion of WETH from the pool, resulting in a skewed token price.

10. The final state of the pool has almost no WETH and an imbalanced supply of `DaosToken`, severely devaluing the token and harming the protocol’s liquidity.

## Location of Affected Code

File: [contracts/DaosLive.sol](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosLive.sol)

```solidity
function finalize(
  // code
  bytes memory data = abi.encodeWithSelector(
      IDaosToken.initialize.selector,
      address(_factory),
      name,
      symbol
  );
  BeaconProxy tokenBeacon = new BeaconProxy(_factory.tokenBeacon(), data);
  token = address(tokenBeacon);
  _factory.emitTokenCreated(token);

  // code
}
```

## Impact

- **Loss of Protocol Funds:** Protocol-owned WETH is extracted via manipulated liquidity removal.
- **Price Collapse:** The `DaosToken` becomes nearly worthless on Uniswap due to the attacker draining liquidity.

## Recommendation

To mitigate this issue:

**Use `CREATE2` with a salt:**

- This allows the `DaosLive` owner to control and obscure the final address more securely.

## Team Response

Fixed.

---
### Example 2

**Auto Label:** Deterministic address generation flaws enable address collisions, allowing attackers to deploy identical contracts, bypass validation, and exploit fund withdrawals or front-running.  

**Original Text Preview:**

`DeployUsdxlUtils`: The `_deployGsm()` function should use proxy address (`_getUsdxlToken()`) instead of its implementation when deploying new Gsm:

```solidity
   function _deployGsm(
        address token,
        address gsmOwner,
        uint256 maxCapacity,
        IDeployConfigTypes.HypurrDeployRegistry memory deployRegistry
    ) internal returns (address gsmProxy) {
        --snip--

        // Deploy GSM implementation
        Gsm gsmImpl = new Gsm(address(usdxlToken), address(token), address(fixedPriceStrategy));

        --snip--
```

Recommendations:

Use `_getUsdxlToken()` instead of `address(usdxlToken)`.

---
### Example 3

**Auto Label:** Deterministic address generation flaws enable address collisions, allowing attackers to deploy identical contracts, bypass validation, and exploit fund withdrawals or front-running.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The `DeployUsdxlUtils._getUsdxlATokenProxy()` and `DeployUsdxlUtils._getUsdxlVariableDebtTokenProxy()` functions incorrectly return implementation contract addresses instead of proxy addresses.

```solidity
//File: src/deployments/utils/DeployUsdxlUtils.sol

function _getUsdxlATokenProxy() internal view returns (address) {
    return address(usdxlAToken);    // Returns implementation instead of proxy
}

function _getUsdxlVariableDebtTokenProxy() internal view returns (address) {
    return address(usdxlVariableDebtToken); // Returns implementation instead of proxy
}
```

This causes four main issues:

1. Incorrect contract exports in deployment artifacts in the `_initializeUsdxlReserve()` function.
2. Incorrect token configurations in the `_setUsdxlAddresses()` function, leaving the USDXL pool's `AToken` and `VariableDebtToken` unconfigured.
3. Incorrect facilitator configurations for the USDXL token in the `_addUsdxlATokenAsEntity()` function.
4. Incorrect discount token and strategy configurations in the `_setDiscountTokenAndStrategy()` function.

## Recommendation

Track the actual proxy addresses that are configured in the USDXL pool instead of using implementation addresses. This ensures that token configurations are applied to the correct contract instances that the pool interacts with.

To implement this:

1. Get and track proxy addresses from pool's reserve data after pool initialization:

```diff
function _initializeUsdxlReserve(
    address token,
    IDeployConfigTypes.HypurrDeployRegistry memory deployRegistry
)
    internal
{
    --- SNIPPED ---
    // set reserves configs
    _getPoolConfigurator(deployRegistry).initReserves(inputs);

+   IPoolAddressesProvider poolAddressesProvider = _getPoolAddressesProvider(deployRegistry);
    //@audit DataTypes should be additional imported
+   DataTypes.ReserveData memory reserveData = IPool(poolAddressesProvider.getPool()).getReserveData(token);

    //@audit Introduce new two state variables to track proxy addresses
+   usdxlATokenProxy = UsdxlAToken(reserveData.aTokenAddress);
+   usdxlVariableDebtTokenProxy = UsdxlVariableDebtToken(reserveData.variableDebtTokenAddress);

    // export contract addresses
    DeployUsdxlFileUtils.exportContract(instanceId, "usdxlATokenProxy", _getUsdxlATokenProxy());
    DeployUsdxlFileUtils.exportContract(instanceId, "usdxlVariableDebtTokenProxy", _getUsdxlVariableDebtTokenProxy());
}
```

2. Update getter functions to return proxy addresses:

```diff
function _getUsdxlATokenProxy() internal view returns (address) {
-    return address(usdxlAToken);
+    return address(usdxlATokenProxy);
}

function _getUsdxlVariableDebtTokenProxy() internal view returns (address) {
-    return address(usdxlVariableDebtToken);
+    return address(usdxlVariableDebtTokenProxy);
}
```

3. Update treasury configuration to use proxy:

```diff
function _setUsdxlAddresses(IDeployConfigTypes.HypurrDeployRegistry memory deployRegistry)
    internal
{
-    usdxlAToken.updateUsdxlTreasury(deployRegistry.treasury);
+    UsdxlAToken(_getUsdxlATokenProxy()).updateUsdxlTreasury(deployRegistry.treasury);

    UsdxlAToken(_getUsdxlATokenProxy()).setVariableDebtToken(_getUsdxlVariableDebtTokenProxy());
    UsdxlVariableDebtToken(_getUsdxlVariableDebtTokenProxy()).setAToken(_getUsdxlATokenProxy());
}
```

---
