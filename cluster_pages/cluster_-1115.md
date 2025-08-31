# Cluster -1115

**Rank:** #38  
**Count:** 522  

## Label
Inadequate timestamp validation and bounds checking lead to incorrect time-based logic, enabling malicious manipulation of time-dependent operations and compromising security, correctness, and deterministic behavior.

## Cluster Information
- **Total Findings:** 522

## Examples

### Example 1

**Auto Label:** Inadequate timestamp validation and bounds checking lead to incorrect time-based logic, enabling malicious manipulation of time-dependent operations and compromising security, correctness, and deterministic behavior.  

**Original Text Preview:**

In the `ChainUtils.convertBlocksToSeconds` function, a hardcoded value of 1000 milliseconds (1 second) per block is used for Ape chain. However, according to current network metrics, the average block time on Ape chain is approximately 1.5 seconds per block, not 1 second.

```solidity
    function convertBlocksToSeconds(uint256 _blocks) internal view returns (uint256) {
        uint256 millisecondsPerBlock;
        ...
        } else if (block.chainid == APECHAIN_MAINNET) {
            millisecondsPerBlock = 1000; // @audit 1500 miliseconds
        ...

        return Math.mulDiv(_blocks, millisecondsPerBlock, 1000, Math.Rounding.Up);
    }
```

It's recommended to update the `millisecondsPerBlock` value for Ape chain to 1500 milliseconds to match the current network's average block time

Reference: https://apescan.io/chart/blocktime

---
### Example 2

**Auto Label:** Inadequate timestamp validation and bounds checking lead to incorrect time-based logic, enabling malicious manipulation of time-dependent operations and compromising security, correctness, and deterministic behavior.  

**Original Text Preview:**

## Severity

Medium Risk

## Description

The `DaosLive` contract allows the owner to **arbitrarily block refunds** by repeatedly extending the `endTime` of the DAO using the `extendTime()` function. Since refunds are only available after `endTime`, this introduces a **centralization risk** and **griefing vector** for contributors.

### Attack Scenario

1. Users contribute to the DAO expecting to be able to claim refunds if the project does not finalize.
2. The `endTime` parameter determines when refunds can be claimed.
3. The contract allows the owner to call `extendTime()` and increase `endTime`.
4. The owner can call `extendTime()` repeatedly before each expiration.
5. This results in a loop where `endTime` is always in the future, **blocking refunds forever**.

## Location of Affected Code

File: [contracts/DaosLive.sol#L333](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosLive.sol#L333)

```solidity
function extendTime(uint256 _endTime) external onlyOwner {
    if (goalReached) revert GoalReached();
    if (_endTime < block.timestamp) revert InvalidEndTime();
    endTime = _endTime;
    _factory.emitExtendTime(endTime);
}
```

## Impact

- Refunds can be indefinitely delayed, effectively trapping user funds and denying contributors their right to exit.
- The contract introduces a significant centralization risk, as the owner can unilaterally manipulate the refund timeline.
- This creates a potential griefing or malicious vector, where the owner may extend the `endTime` continuously to prevent withdrawals or even by setting it to a very high value that is not possible to reach in the near future.

## Recommendation

- Impose a strict cap on the total amount of time that `endTime` can be extended.
- Require governance approval (e.g., DAO vote or multisig) for any extension beyond a predefined threshold.

## Team Response

Fixed.

---
### Example 3

**Auto Label:** Inadequate timestamp validation and bounds checking lead to incorrect time-based logic, enabling malicious manipulation of time-dependent operations and compromising security, correctness, and deterministic behavior.  

**Original Text Preview:**

## Severity

Critical Risk

## Description

In the following code snippet, using `IUniRouter.exactInputSingle()` for token swapping:

```solidity
IUniRouter uniRouter = IUniRouter(_factory.uniRouter());
uniRouter.exactInputSingle{value: snipeAmount}(
    IUniRouter.ExactInputSingleParams({
        tokenIn: wethAddr,
        tokenOut: token,
        fee: UNISWAP_V3_FEE,
        recipient: address(this),
        amountIn: snipeAmount,
        amountOutMinimum: 0,
        sqrtPriceLimitX96: 0
    })
);
```

The `deadline` parameter is missing during the initialization of `ExactInputSingleParams`. As a result, the default value of `0` is used for `deadline`, which is always in the past. This causes the Uniswap `exactInputSingle()` function to **revert** the transaction due to deadline expiry in the below Uniswapv3 code.

[SwapRouter.sol](https://github.com/Uniswap/v3-periphery/blob/464a8a49611272f7349c970e0fadb7ec1d3c1086/contracts/SwapRouter.sol#L115C5-L129C6)

```solidity
function exactInputSingle(ExactInputSingleParams calldata params)
    external
    payable
    override
@>  checkDeadline(params.deadline)
    returns (uint256 amountOut)
{
    amountOut = exactInputInternal(
        params.amountIn,
        params.recipient,
        params.sqrtPriceLimitX96,
        SwapCallbackData({path: abi.encodePacked(params.tokenIn, params.fee, params.tokenOut), payer: msg.sender})
    );
    require(amountOut >= params.amountOutMinimum, 'Too little received');
}
```

[PeripheryValidation.sol](https://github.com/Uniswap/v3-periphery/blob/v1.0.0/contracts/base/PeripheryValidation.sol)

```solidity
abstract contract PeripheryValidation is BlockTimestamp {
    modifier checkDeadline(uint256 deadline) {
        require(_blockTimestamp() <= deadline, 'Transaction too old');
        _;
    }
}
```

## Location of Affected Code

File: [contracts/DaosLive.sol#L314](https://github.com/ED3N-Ventures/daoslive-sc/blob/9a1856db2060b609a17b24aa72ab35f2cdf09031/contracts/DaosLive.sol#L314)

## Impact

- All token snipe attempts will fail and revert unless `deadline` is correctly set.
- Results in denial of service for token snipe functionality, along with failure of `finalize()` function.

## Recommendation

Set the `deadline` parameter explicitly using a valid timestamp (e.g., `block.timestamp` at least the current timestamp).

```diff
exactInputSingleParams({
    tokenIn: wethAddr,
    tokenOut: token,
    fee: UNISWAP_V3_FEE,
    recipient: address(this),
    amountIn: snipeAmount,
    amountOutMinimum: 0,
    sqrtPriceLimitX96: 0,
+   deadline: block.timestamp
})
```

## Team Response

Fixed.

---
