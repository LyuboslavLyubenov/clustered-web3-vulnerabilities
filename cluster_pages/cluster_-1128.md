# Cluster -1128

**Rank:** #441  
**Count:** 8  

## Label
Failure to enforce critical parameters or validate configurations during pool creation or factory transitions, leading to unbounded control, incorrect behavior, and potential fund exposure.

## Cluster Information
- **Total Findings:** 8

## Examples

### Example 1

**Auto Label:** Failure to validate pool existence before acting on asset or pool identifiers, leading to incorrect state propagation, misuse of default modules, and unauthorized state changes.  

**Original Text Preview:**

The `public` `getAssetModule` function in the `UniswapV4HooksRegistry` contract does not check for the existence of the provided `assetId`. This oversight may result in an empty `poolKey` and cause the function to erroneously return the default asset module (`DEFAULT_UNISWAP_V4_AM`) even when the `assetId` does not exist.

```solidity
    function getAssetModule(uint256 assetId) public view returns (address assetModule) {
>>>     (PoolKey memory poolKey,) = POSITION_MANAGER.getPoolAndPositionInfo(assetId);

        // Check if we can use the default Uniswap V4 AM.
>>>     if (
            Hooks.hasPermission(uint160(address(poolKey.hooks)), Hooks.BEFORE_REMOVE_LIQUIDITY_FLAG)
                || Hooks.hasPermission(uint160(address(poolKey.hooks)), Hooks.AFTER_REMOVE_LIQUIDITY_FLAG)
        ) {
            // If not a specific Uniswap V4 AM must have been set.
            // Returns the zero address if no Asset Module is set.
            assetModule = hooksToAssetModule[address(poolKey.hooks)];
        } else {
            // If BEFORE_REMOVE_LIQUIDITY_FLAG and AFTER_REMOVE_LIQUIDITY_FLAG are not set,
            //Then we use the default Uniswap V4 AM.
            // The NoOP hook "AFTER_REMOVE_LIQUIDITY_RETURNS_DELTA_FLAG" is by default not allowed,
            // as it can only be accessed if "AFTER_REMOVE_LIQUIDITY_FLAG" is implemented.
            assetModule = DEFAULT_UNISWAP_V4_AM;
        }
    }
```

The absence of a check can lead to misleading information being returned from public functions. Test:

```solidity
    function testFuzz_Success_getAssetModule_InvalidTokenId(uint96 tokenId)
        public
    {
        // Calling getAssetModule()
        address assetModule = v4HooksRegistry.getAssetModule(tokenId);

        // It returns DefaultUniswapV4AM. It should returns `address(0)` or revert() since tokenId does not exist.
        assertEq(assetModule, address(uniswapV4AM));
    }
```

Implement a check to verify the existence of the `poolKey`. If the `poolKey` is empty, the call function `getAssetModule` should revert or return a zero address to clearly indicate the non-existence.

---
### Example 2

**Auto Label:** Failure to validate pool existence before acting on asset or pool identifiers, leading to incorrect state propagation, misuse of default modules, and unauthorized state changes.  

**Original Text Preview:**

The `BunniHook::setAmAmmEnabledOverride` function allows an override to be set for the amAmm availability of a pool without verifying if the pool actually exists. This can lead to situations where overrides are set for non-existent pools, causing confusion and potential misconfiguration in the protocol.

```solidity
File: BunniHook.sol
267:     function setAmAmmEnabledOverride(PoolId id, BoolOverride boolOverride) external onlyOwner {
268:         amAmmEnabledOverride[id] = boolOverride;
269:         emit SetAmAmmEnabledOverride(id, boolOverride);
270:     }
```

The owner can activate a pool via `BunniHook::setAmAmmEnabledOverride`, and the function `BunniHook::getAmAmmEnabled` will return true even when the pool does not exist. Since this function is `external`, it can cause discrepancies for those who use this function.

```solidity
File: BunniHook.sol
340:     function getAmAmmEnabled(PoolId id) external view override returns (bool) {
341:         return _amAmmEnabled(id);
342:     }
```

To mitigate this issue, it is recommended to implement a check to verify if the pool exists before allowing the override to be set.

---
### Example 3

**Auto Label:** Failure to enforce critical parameters or validate configurations during pool creation or factory transitions, leading to unbounded control, incorrect behavior, and potential fund exposure.  

**Original Text Preview:**

## Vulnerability Report

**Severity:** Low Risk  
**Context:** ERC4626HyperdriveDeployerCoordinator.sol#L67-L73  
**Description:** The `ERC4626HyperdriveDeployerCoordinator._checkPoolConfig` function checks the minimum share reserves twice.  
**Recommendation:** The second check should be on the `_deployConfig.minimumTransactionAmount` amount instead.  
**DELV:** Fixed in PR 772.  
**Spearbit:** Verified.

---
