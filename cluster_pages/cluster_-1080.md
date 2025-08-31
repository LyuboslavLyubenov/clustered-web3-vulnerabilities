# Cluster -1080

**Rank:** #160  
**Count:** 83  

## Label
Insufficient or incorrect accounting of user funds leads to double withdrawals, unaccounted surpluses, and unauthorized borrowing, causing severe user losses and enabling attacker exploitation through flawed state tracking and validation.

## Cluster Information
- **Total Findings:** 83

## Examples

### Example 1

**Auto Label:** Inadequate accounting of user balances and liquidity flows leads to unfair rewards, unspent funds, or incorrect token issuance, compromising fairness, accuracy, and protocol integrity.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

The `Burve::mint` function interacts with the `uniswapV3MintCallback`, which transfers `token0` and `token1` to the Uniswap pool. However, **fee-on-transfer** tokens (tokens that deduct a percentage as a fee on each transfer) are not properly accounted for in this implementation.

When using fee-on-transfer tokens, the amount received by the pool **will be less than the expected amount**, causing the minting process to fail due to an arithmetic underflow.

### **Proof of Concept (PoC)**

To simulate the issue, I modified the `Burve::uniswapV3MintCallback` function by subtracting a fixed amount from the transferred values to mimic a fee deduction:

```solidity
function uniswapV3MintCallback(uint256 amount0Owed, uint256 amount1Owed, bytes calldata data) external {
    address source = abi.decode(data, (address));
    TransferHelper.safeTransferFrom(token0, source, address(pool), amount0Owed - 10);
    TransferHelper.safeTransferFrom(token1, source, address(pool), amount1Owed - 10);
}
```

### **Test Case & Logs**

Test:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.27;

import {Test, console} from "forge-std/Test.sol";
import {Burve, TickRange} from "../src/Burve.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract BurveTest is Test {
    Burve public burve;
    address public pool = 0xc473e2aEE3441BF9240Be85eb122aBB059A3B57c;
    address public USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address public holderu = 0xf89d7b9c864f589bbF53a82105107622B35EaA40;
    address public WETH = 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1;
    address public holderw = 0x70d95587d40A2caf56bd97485aB3Eec10Bee6336;
    uint128[] public weights = [4249, 3893, 9122];
    TickRange public range = TickRange(-199800, -194880);
    TickRange[] public ranges = [range, range, range];
    address public alice = makeAddr("random_user");

    function setUp() public {
        // Create main net fork
        vm.createSelectFork({urlOrAlias: "arbitrum_api", blockNumber: 260000000});
        burve = new Burve(pool, address(0), ranges, weights);
        vm.prank(holderu);
        IERC20(USDC).transfer(alice, 1e11);
        vm.prank(holderw);
        IERC20(WETH).transfer(alice, 1e20);
    }

    function test_fee() public {
        vm.startPrank(alice);
        IERC20(USDC).approve(address(burve), 1e11);
        IERC20(WETH).approve(address(burve), 1e20);
        burve.mint(alice, 100);
    }
}
```

Logs:

```
    │   │   │   └─ ← [Return] 81792443700 [8.179e10]
    │   │   ├─ [20564] Burve::uniswapV3MintCallback(90917 [9.091e4], 1, 0x000000000000000000000000e2f70c5cbd298fb122db24264d8923e348cedae3)
    │   │   │   ├─ [14480] 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1::transferFrom(random_user: [0xE2F70c5cbD298Fb122db24264d8923E348CeDaE3], 0xc473e2aEE3441BF9240Be85eb122aBB059A3B57c, 90907 [9.09e4])
    │   │   │   │   ├─ [13728] 0x8b194bEae1d3e0788A1a35173978001ACDFba668::transferFrom(random_user: [0xE2F70c5cbD298Fb122db24264d8923E348CeDaE3], 0xc473e2aEE3441BF9240Be85eb122aBB059A3B57c, 90907 [9.09e4]) [delegatecall]
    │   │   │   │   │   ├─ emit Transfer(from: random_user: [0xE2F70c5cbD298Fb122db24264d8923E348CeDaE3], to: 0xc473e2aEE3441BF9240Be85eb122aBB059A3B57c, value: 90907 [9.09e4])
    │   │   │   │   │   ├─ emit Approval(owner: random_user: [0xE2F70c5cbD298Fb122db24264d8923E348CeDaE3], spender: Burve: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], value: 99999999999999909093 [9.999e19])
    │   │   │   │   │   └─ ← [Return] true
    │   │   │   │   └─ ← [Return] true
    │   │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
```

**notes:** same applies for multi token pool as well

---
### Example 2

**Auto Label:** Inadequate accounting of user balances and liquidity flows leads to unfair rewards, unspent funds, or incorrect token issuance, compromising fairness, accuracy, and protocol integrity.  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

The Burve contract allows users to provide liquidity to both the island pool and Uniswap V3 pools. When adding liquidity to the island pool, it will transfer tokens in from the sender (which is the Burve contract itself) before providing liquidity to Uniswap.

```solidity
    function mint(address recipient, uint128 liq) external {
        for (uint256 i = 0; i < distX96.length; ++i) {
            uint128 liqAmount = uint128(shift96(liq * distX96[i], true));
            mintRange(ranges[i], recipient, liqAmount);
        }

        _mint(recipient, liq);
    }
```

However, when users call `Burve.mint()`, it does not transfer the necessary tokens beforehand. This results in a failed transaction, preventing users from successfully providing liquidity to the island pool through the Burve contract.

## Recommendations

Ensure the required tokens are transferred and approved before calling `island.mint()`.

---
### Example 3

**Auto Label:** **Incorrect state updates due to flawed logic and missing boundaries, leading to erroneous balances, locked assets, and compromised protocol stability.**  

**Original Text Preview:**

## Severity

**Impact:** Medium

**Likelihood:** Medium

## Description

- In `processWithdrawals()` function: when the `_withdrawShares = 0`, the `s.totalLiquidity` is increased by the liquidity of the collected-and-deposited fees of the position, where it assumes that the totalLiquidity of the position will not be decreased when `_reviseHedge()` is called:

```javascript
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
             s.totalLiquidity = _totalLiquidity;
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

Where:

```javascript
 function _reviseHedge(
        uint256 _totalLiquidity,
        bytes memory path0,
        bytes memory path1
    ) internal virtual {
        uint256 _newHedgeLiquidity = (_totalLiquidity * s.hedgeSize) / 1e18;

        uint256 _prevHedgeLiquidity = _loanData.liquidity;

        if (_newHedgeLiquidity > _prevHedgeLiquidity) {
            _increaseHedge(
                _newHedgeLiquidity - _prevHedgeLiquidity,
                path0,
                path1
            );
        } else if (_newHedgeLiquidity < _prevHedgeLiquidity) {
            _decreaseHedge(
                _prevHedgeLiquidity - _newHedgeLiquidity,
                path0,
                path1
            );
        }
    }
```

- Given that there's a setter for the `hedgeSize` (where it's used to calculate the `_newHedgeLiquidity`), then the `s.totalLiquidity` should be updated based on the liquidity position in case the `hedgeSize` was updated (also, in `_reviseHedge()`, the hedge liquidity is fetched based on the last updated data without checking if the hedge has accrued interest over time).

- So if the new `hedgeSize` is decreased and results in `_newHedgeLiquidity < _prevHedgeLiquidity` , then the hedge will be decreased, which might result in the hedge incurring losses, hence affecting the liquidity, which will affect the next process that's going to be executed after the withdrawal: if the next process is a deposit, then this incorrect `s.totalLiquidity` will result in minting wrong shares for that deposited period as the liquidity doesn't reflect the actual one (since it's the cashed one + liquidity from the collected-and-deposited fees of the LP position).

- Same issue in `DepositProcess.processDeposits()` function.

## Recommendations

```diff
 function processWithdrawals(
        bytes memory path0,
        bytes memory path1
    ) external virtual override {
        //...
        uint256 _liquidity = _depositFees(_tokenId, path0, path1);
        _totalLiquidity += _liquidity;

        if (_withdrawShares == 0) {
            // if no withdrawals we only compound fees
            if (_liquidity > 0) {
                _reviseHedge(_totalLiquidity, path0, path1); // liquidity only goes up here
            }
-            s.totalLiquidity = _totalLiquidity;
+            s.totalLiquidity = _getTotalLiquidity(s.tokenId);
            FundingVault(s.withdrawVault).disburseClaims();
            return;
        }

        //...
    }
```

---
