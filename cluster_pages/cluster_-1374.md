# Cluster -1374

**Rank:** #139  
**Count:** 103  

## Label
Lack of access control and input validation enabling unauthorized modifications or bypasses of critical security checks, leading to fund exposure, protocol manipulation, or unauthorized operations.

## Cluster Information
- **Total Findings:** 103

## Examples

### Example 1

**Auto Label:** **Missing authorization and input validation enable unauthorized access, fund manipulation, and bypass of critical security checks.**  

**Original Text Preview:**

## Severity

**Impact:** Low

**Likelihood:** High

## Description

The `removeLiquidity()` function currently allows any user to call the function and remove liquidity on behalf of another user, as it does not properly restrict access to the user performing the action. This could lead to unauthorized removal of liquidity from another user's account, potentially resulting in loss of assets, especially in scenarios where malicious actors attempt to exploit the function.

```solidity
    function removeLiquidity(address _tokenOut, uint256 _lpAmount, uint256 _minOut, address _to)
        external
        override
        nonReentrant
        whenNotPaused
    {
        _requireAmount(_lpAmount);
        _validateAsset(_tokenOut);

        if (_lpAmount > _getUserUnlockedLP(_to)) revert PoolErrors.LiquidityLocked();

        // remove liquidity
        (, uint256 netOutAmount, uint256 pFee, uint256 tokenOutPrice) = _calcRemoveLiquidity(_tokenOut, _lpAmount);
        _removeLiquidityFromPool(_minOut, netOutAmount, _tokenOut, tokenOutPrice, pFee);

        uint256 rewardsTokenOut = _isFullyExiting(_to, _lpAmount) ? _claimAllRewards(_to, _tokenOut) : 0;

        // PLP management
        _updateLockData(false, _lpAmount, _to);
        _exitAndBurnPLP(_lpAmount, _to);

        _doTransferOut(_tokenOut, _to, netOutAmount + rewardsTokenOut);
        _emitPLPPrice();
        emit LiquidityRemoved(_to, _tokenOut, _lpAmount, netOutAmount, rewardsTokenOut, pFee);
    }
```

## Recommendations

Implement access control to ensure that only the user who owns the liquidity can remove their liquidity. This can be done by checking if the `_to` address is the same as the caller (`msg.sender`) before proceeding with the liquidity removal.

---
### Example 2

**Auto Label:** **Insufficient input or state validation leads to asset loss, inconsistent state, or unauthorized operations through unchecked data or token mismatches.**  

**Original Text Preview:**

Whenever users call `openPosition()` or `closePosition()` they pass the `path` variable which is used to perform swaps. The issue is that the code doesn't verify the value of `path` to make sure that the first and last addresses in the path list are equal to the tokens that are going to be swapped. In some cases if the user specifies a wrong value for the last address in the path then the code would receive the wrong tokens and those tokens would stay in the contract address and the user would lose funds. It's better to verify the `path` parameter's value to avoid such edge cases.

---
### Example 3

**Auto Label:** **Insufficient input or state validation leads to asset loss, inconsistent state, or unauthorized operations through unchecked data or token mismatches.**  

**Original Text Preview:**

When an ownership update is triggered in the base chain of the NFT, the response `_updateDelegations()` is executed. In this function, `multicallData` is filled with the calls to update the delegations of the tokens for the stale owner and the new owner, and then `multicall()` is called.

```solidity
    function _updateDelegations(bytes calldata _message) internal {
        // message should now be an abi encoded array of abi encoded (address, address, address, uint256) tuples
        (address collectionAddress, address[] memory staleOwners, address[] memory newOwners, uint256[] memory tokenIds)
        = abi.decode(_message, (address, address[], address[], uint256[]));
@>      bytes[] memory multicallData = new bytes[](tokenIds.length * 2);

@>      for (uint256 i = 0; i < tokenIds.length; ++i) {
            address staleOwner = staleOwners[i];
            address newOwner = newOwners[i];
            uint256 tokenId = tokenIds[i];

            if (staleOwner != address(0)) {
                multicallData[i] = abi.encodeWithSelector(
                    IDelegateRegistry.delegateERC721.selector,
                    staleOwner,
                    collectionAddress,
                    tokenId,
                    _GLOBAL_RIGHTS_WITH_MAX_EXPIRY,
                    false
                );
            }

@>          if (IERC721(collectionAddress).ownerOf(tokenId) == address(this)) {
                multicallData[i + tokenIds.length] = abi.encodeWithSelector(
                    IDelegateRegistry.delegateERC721.selector,
                    newOwner,
                    collectionAddress,
                    tokenId,
                    _GLOBAL_RIGHTS_WITH_MAX_EXPIRY,
                    true
                );

                delegatedOwners[collectionAddress][tokenId] = newOwner;
            } else {
                // if token is not owned by this contract, there should be no delegation
                delegatedOwners[collectionAddress][tokenId] = address(0);
            }
        }

@>      IDelegateRegistry(DELEGATE_REGISTRY).multicall(multicallData);
    }
```

The code applies conditionals to determine what calls to include in the `multicallData` array. What is important to note here is that if any of the calls are not populated in the `multicallData` array, `multicall()` will revert, which will cause the ownership update to fail.

For this to happen, either `staleOwner` (taken from `delegatedOwners`) has to be `address(0)` or `IERC721(collectionAddress).ownerOf(tokenId)` has to be different from the beacon address. Both of these conditions should never happen, as a successful ownership update on the base chain implies that a shadow NFT is unlocked, and thus, the NFT in the base chain is locked.

Therefore, the conditional statements do not have any effects, as they will always be true. However, the potential reversion of `multicall()` if the `multicallData` array is not populated with all the calls should be taken into account in the case of future changes to the code. If at any point in the future, it is expected that the conditions might not be met, the size of the `multicallData` array should be reduced to only include the calls that are populated.

---
