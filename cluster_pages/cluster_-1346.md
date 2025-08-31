# Cluster -1346

**Rank:** #220  
**Count:** 49  

## Label
Common vulnerability type: **Insufficient validation and improper state handling leading to denial-of-service, fee loss, or unauthorized asset manipulation through edge-case arithmetic or hook exploitation.**

## Cluster Information
- **Total Findings:** 49

## Examples

### Example 1

**Auto Label:** Common vulnerability type: **Insufficient validation and improper state handling leading to denial-of-service, fee loss, or unauthorized asset manipulation through edge-case arithmetic or hook exploitation.**  

**Original Text Preview:**

## Severity

Low Risk

## Description

The `PositionManager.addLiquidity()` provides a way for the admin to increase the liquidity of a position. The function uses `_addLiquidityUniV3()` that hardcodes the `amount0Min` and `amount1Min` arguments to 0 and exposes the admin to potential slippage.

## Location of Affected Code

File: [PositionManager.sol]()

```solidity
function _addLiquidityUniV3(uint256 tokenId, uint256 amountAdd0, uint256 amountAdd1)
    internal
    returns (uint128 liquidity, uint256 amount0, uint256 amount1)
{
    UniV3Nft memory nft = _getNftUniV3(tokenId);
    _setAllowanceMaxIfNeeded(IERC20(nft.token0), amountAdd0, address(nftManager));
    _setAllowanceMaxIfNeeded(IERC20(nft.token1), amountAdd1, address(nftManager));

    INonfungiblePositionManager.IncreaseLiquidityParams memory params = INonfungiblePositionManager
        .IncreaseLiquidityParams({
        tokenId: tokenId,
        amount0Desired: amountAdd0,
        amount1Desired: amountAdd1,
@>      amount0Min: 0,
@>      amount1Min: 0,
        deadline: block.timestamp
    });

    (liquidity, amount0, amount1) = nftManager.increaseLiquidity(params); // @audit check how increaseLiquidity works
}
```

## Impact

Worst case scenario admin will provide less liquidity than expected and the excess tokens will be returned to the `PositionManager.sol`. From this point on these excess tokens cannot be used for anything except to be converted to Volt tokens and sent to the staking contract to be distributed as rewards.

## Recommendation

Provide slippage protection for the `increaseLiquidity()`.

## Team Response

Acknowledged.

---
### Example 2

**Auto Label:** **Improper allowance or balance checks leading to incorrect swap execution, partial fulfillment, or denial of service due to logic errors in amount comparisons or state validation.**  

**Original Text Preview:**

## Severity

Low Risk

## Description

In `UniswapNFTManager.sol`, `callback()` approves Uniswap NFT to spend amount to burn + 1 by only sending amount to burn as amount desired in the Uniswap NFT increase liquidity calls. This could lead to significant consequences.

## Location of Affected Code

File: [src/managers/UniswapNFTManager.sol](https://github.com/cryptovash/sammin-core/blob/c49807ae2965cf6d121a10507a43de1d64ba1e70/periphery/src/managers/UniswapNFTManager.sol)

```solidity
ERC20 token0 = borrower.TOKEN0();
ERC20 token1 = borrower.TOKEN1();

(uint256 burned0, uint256 burned1, , ) = borrower.uniswapWithdraw(
    lower,
    upper,
    uint128(liquidity),
    address(this)
);

>1 token0.safeApprove(address(UNISWAP_NFT), burned0 + 1);
>2 token1.safeApprove(address(UNISWAP_NFT), burned1 + 1);
UNISWAP_NFT.increaseLiquidity(
    IUniswapPositionNFT.IncreaseLiquidityParams({
        tokenId: tokenId,
>3      amount0Desired: burned0,
>4      amount1Desired: burned1,
        amount0Min: 0,
        amount1Min: 0,
        deadline: block.timestamp
    })
);

token0.safeTransfer(owner, token0.balanceOf(address(this)));
token1.safeTransfer(owner, token1.balanceOf(address(this)));
```

## Impact

The contract has no other way to approve tokens, as a result, there will be a build-up of allowance to Uniswap NFTs in the contract. This means if the NFT gets compromised, the allowance buildup will make it possible to siphon any available tokens from the manager.

Also, with certain tokens (tokens with approval race protection), the extra allowance will cause permanent DOS as the next approval call will require the previous allowance to be 0, which it will not be in this case. See USDT on Ethereum mainnet as an example. Berachain is a relatively new chain, so newly listed tokens may include this design.

## Recommendation

Zero out approvals after operations to be safe, or approve only the needed amount.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** **Improper allowance or balance checks leading to incorrect swap execution, partial fulfillment, or denial of service due to logic errors in amount comparisons or state validation.**  

**Original Text Preview:**

## Context
(No context files were provided by the reviewer)

## Description
In the `SheepDog` contract's `buySheep` function, the swap operation uses an incorrect token amount, leading to a guaranteed revert every time this function is called. The function calculates `balGasToken` as the total `wGasToken` balance of the contract and determines `buyAmount` as `balGasToken` minus a 5% team fee. 

However, when invoking `IRouter(router).swapExactTokensForTokensSimple`, it specifies `balGasToken` as the amount to swap instead of `buyAmount`. This mismatch occurs because the router is only approved to spend `buyAmount`, yet the swap attempts to use the full `balGasToken` balance, which exceeds the approved amount. As a result, the transaction will revert due to insufficient allowance, preventing the contract from purchasing SHEEP tokens as intended and disrupting the yield mechanism for depositors.

## Recommendation
Correct the swap operation in the `buySheep` function by replacing `balGasToken` with `buyAmount` in the `IRouter(router).swapExactTokensForTokensSimple` call to align the swapped amount with the approved amount.

## Additional Information
- **Ceazor Snack Sandwich**: Fixed in `36539b3`.
- **Cantina**: Fix OK.

---
