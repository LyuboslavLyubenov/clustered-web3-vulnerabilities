# Cluster -1111

**Rank:** #340  
**Count:** 19  

## Label
Hardcoded addresses limit flexibility, expose contracts to misconfiguration, and prevent dynamic updates—increasing attack surface and undermining cross-chain deployability and maintainability.

## Cluster Information
- **Total Findings:** 19

## Examples

### Example 1

**Auto Label:** Hardcoded addresses bypass dynamic address resolution, leading to brittleness against protocol upgrades or migrations and risking operational failures or security exposure.  

**Original Text Preview:**

In the `ArbitrumStrategyManager` contract, the `_aaveV3Pool` variable is defined as an `immutable` address and is set to a hardcoded value of `0x794a61358D6845594F94dc1DB02A252b5b4814aD` during deployment.

```solidity
    /// @dev Address of the Aave V3 Pool
    address internal immutable _aaveV3Pool;
```

```solidity
    address public constant AAVE_V3_POOL =
        0x794a61358D6845594F94dc1DB02A252b5b4814aD;
    address public constant MERKL_DISTRIBUTOR =
        0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae;

    function run() public {
        vm.startBroadcast();

        manager = new ArbitrumStrategyManager(
            ADMIN,
            AAVE_V3_POOL,
            TREASURY,
            MERKL_DISTRIBUTOR,
            HYPERNATIVE
        );

        vm.stopBroadcast();
    }
```

However, hardcoding the Aave V3 pool address poses a risk:

The `AaveV3 pool` address is typically managed through the `Aave Addresses Provider`, which allows for updates to the pool address if necessary.

[Link](https://arbiscan.io/address/0xa97684ead0e402dc232d5a977953df7ecbab3cdb#code)
[Link](https://aave.com/docs/developers/smart-contracts/pool-addresses-provider)

> `setAddress` Sets the address of the protocol contract stored at the given
> id, replacing the address saved in the addresses map.

```solidity
  /// @inheritdoc IPoolAddressesProvider
  function setAddress(bytes32 id, address newAddress) external override onlyOwner {
    address oldAddress = _addresses[id];
    _addresses[id] = newAddress;
    emit AddressSet(id, oldAddress, newAddress);
  }
```

```solidity
  /// @inheritdoc IPoolAddressesProvider
  function getPool() external view override returns (address) {
    return getAddress(POOL);
  }
```

Even though `setAddress` should be called with care, it's still possible that it is called and does a hard replacement of the current pool address in the addresses map.

Currently, the `pool` address is hardcoded and may not catch up with the address upgrade.

By hardcoding the address, the contract may interact with an outdated pool, leading to potential issues in functionality and security.

A recommended approach is to use `IPoolAddressesProvider(addressProviderAddress).getPool()` to reference `pool` instead of `hardcoding`

---
### Example 2

**Auto Label:** Hardcoded addresses bypass dynamic address resolution, leading to brittleness against protocol upgrades or migrations and risking operational failures or security exposure.  

**Original Text Preview:**

## AaveV3Farm Vulnerability Report

## Severity
**Low Risk**

## Context
*AaveV3Farm.sol#L25*

## Description
The `PoolDataProvider` address is one that can change due to AAVE governance proposals and should not be an immutable value. See recent upgrade (AAVE v3 Governance proposal 252).

## Impact Explanation
This information is only used in `maxDeposit` referenced elsewhere in the system only by `FarmRebalancer.singleMovement`. In the event of a catastrophic upgrade on the Aave side, `CoreControlled` allows for relevant handling and the broken farm could be removed from the registry and replaced by a new farm. For this reason, the overall severity is low.

## Recommendation
Whenever the `PoolDataProvider` is needed, fetch it from the `AddressProvider` using `IAddressProvider(_addressProvider).getPoolDataProvider()`. Broadly, anticipate AAVE changes that can impact the Farm wrapper. Monitoring of the governance forum along with on-chain monitoring is advised. The `CoreControlled` functionality of these Farms does assist in migrations if needed in the future.

## Additional Information
- **infiniFi:** Fixed in `54c7d35`.
- **Spearbit:** Fix verified.

---
### Example 3

**Auto Label:** Hardcoded addresses bypass dynamic address resolution, leading to brittleness against protocol upgrades or migrations and risking operational failures or security exposure.  

**Original Text Preview:**

## Summary

The Aave pool is hardcoded and its not recommended by aave doc.

## Vulnerability Details

The owner of protocol needs to provide Aave pool address when deploying `AaveDIVAWrapper` contract in its `constructor()`.

we can see this aave pool address is stored in `immutable` variable in `AaveDIVAWrapperCore` contract.

```Solidity
    address private immutable _aaveV3Pool; // Pool contract address

    /**
     * @dev Initializes the AaveDIVAWrapper contract with the addresses of DIVA Protocol, Aave V3's Pool
     * contract and the owner of the contract.
     * @param diva_ Address of the DIVA Protocol contract.
     * @param aaveV3Pool_ Address of the Aave V3 Pool contract.
     * @param owner_ Address of the owner for the contract, who will be entitled to claim the yield.
     * Retrievable via Ownable's `owner()` function or this contract's `getContractDetails` functions.
     */
    constructor(address diva_, address aaveV3Pool_, address owner_) Ownable(owner_) {
        // Validate that none of the input addresses is zero to prevent unintended initialization with default addresses.
        // Zero address check on `owner_` is performed in the OpenZeppelin's `Ownable` contract.
        if (diva_ == address(0) || aaveV3Pool_ == address(0)) {
            revert ZeroAddress();
        }

        // Store the addresses of DIVA Protocol and Aave V3 in storage.
        _diva = diva_;
        _aaveV3Pool = aaveV3Pool_;
    }
```

However the Aave doc mentions that `PoolAddressProvider` contract should be queried everytime to provide the current pool address. Because if the pool contract were migrated to a new address, it would disrupt the core logic of this protocol.
<https://aave.com/docs/developers/smart-contracts/pool-addresses-provider>

instance of this issue in other contest:
<https://github.com/hats-finance/Origami-0x998f1b716a5022be026ca6b919c0ddf45ca31abd/issues/58>

## Impact

Breaking of core logic and every calls to Aave pool can be revert

## Tools Used

Manual Review

## Recommendations

Consider using `PoolAddressProvider` contract.

---
