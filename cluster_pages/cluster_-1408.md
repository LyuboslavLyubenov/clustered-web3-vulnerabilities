# Cluster -1408

**Rank:** #452  
**Count:** 7  

## Label
Rounding errors in integer arithmetic cause imprecise balance tracking and residual dust, leading to inconsistent state, incorrect event emissions, and potential underpayment of user deposits.

## Cluster Information
- **Total Findings:** 7

## Examples

### Example 1

**Auto Label:** Rounding errors in integer arithmetic cause imprecise balance tracking and residual dust, leading to inconsistent state, incorrect event emissions, and potential underpayment of user deposits.  

**Original Text Preview:**

1. When users deposit ETH through the LidoStrategy contract, the contract assumes the returned stETH will equal the deposited ETH amount. However, it may not be true.

```solidity
    function deposit(uint256 _value) public returns(uint256){
        (bool success,bytes memory returndata) = LIDO_DEPOSIT_ADDRESS.call{value:_value}(abi.encodeWithSignature("submit(address)", address(this)));
        ...
        return 100;
    }
```

First, in Lido's `_submit` function, the ETH amount is converted to shares which rounds down:

```solidity
    function _submit(address _referral) internal returns (uint256) {
        require(msg.value != 0, "ZERO_DEPOSIT");
        ...

        uint256 sharesAmount = getSharesByPooledEth(msg.value);

        _mintShares(msg.sender, sharesAmount);

        ...
    }
```

When checking stETH balance, another round of conversion occurs which can also round down.

```solidity
    function balanceOf(address _account) external view returns (uint256) {
        return getPooledEthByShares(_sharesOf(_account));
    }
```

2. The same rounding issue happens when stETH is transferred to the DepositETH contract, the transfer `_value` can be less than the receive amount.

```solidity
  function _depositERC(address _tokenAddress, string memory _receiver, uint256 _value, uint32 _destID,uint256 _lzFee) internal onlyWhitelisted(_tokenAddress){
        ...
        IERC20(_tokenAddress).transferFrom(msg.sender, address(this), _value);
        ...
    }
```

It's recommended to:

- bookkeep in shares instead of stETH
- using wstETH instead of stETH
- calculate the actual stETH received by the difference between the balance before and after transfer.
  For reference: https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case

---
### Example 2

**Auto Label:** Rounding errors in integer arithmetic cause imprecise balance tracking and residual dust, leading to inconsistent state, incorrect event emissions, and potential underpayment of user deposits.  

**Original Text Preview:**

## Corner Case

**Severity:** Low Risk  
**Context:** CoreAdapter.sol#L68-L72, EthereumGeneralAdapter1.sol#L111-L115  
**Description:** 

When `EthereumGeneralAdapter1.wrapStEth()` is called with `amount = type(uint256).max`, the stETH balance of the contract is passed to `WST_ETH.wrap()`:

```solidity
if (amount == type(uint256).max) amount = IERC20(ST_ETH).balanceOf(address(this));
require(amount != 0, ErrorsLib.ZeroAmount());
uint256 received = IWstEth(WST_ETH).wrap(amount);
```

This is meant to wrap the contract's entire stETH balance into wstETH. Logically, this means that there should not be any stETH remaining in the contract. However, for certain stETH balances, passing `stETH.balanceOf()` to `wstETH.wrap()` directly could leave dust amounts of stETH behind due to the 1-2 wei corner case.

## Under the hood:

- `stETH.balanceOf()` calls `stETH.getPooledEthByShares()` to convert the shares held by the contract into ETH.
- `wstETH.wrap()` `! stETH.transferFrom()` calls `stETH.getSharesByPooledEth()` to convert the amount passed into shares.

This results in the following math:

- `amount = getPooledEthByShares(shares) = shares * totalPooledEther / totalShares`
- `sharesTransferred = getSharesByPooledEth(amount) = amount * totalShares / totalPooledEther`

where:
- **shares** is the amount of stETH held by the contract in shares.
- **sharesTransferred** is the amount of shares transferred by `wstETH.wrap()`.

Due to rounding down, `sharesTransferred < shares` is possible, which ends up leaving a dust amount of stETH behind.

Subsequent calls in the same bundle might expect the stETH balance of the contract to be 0 after `wrapStEth()` is called (e.g., a subsequent call also uses `stETH.balanceOf()` to determine the amount of that operation). However, since this is not the case, their execution could be affected.

Note that the same situation also occurs when `CoreAdapter.erc20Transfer()` is called with `amount = type(uint256).max` to transfer out the contract's stETH balance.

## Proof of Concept:

The following proof of concept demonstrates that 1 wei of stETH is left behind when 1e18 of stETH is transferred:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import "lib/forge-std/src/Test.sol";
import {ForkTest} from "test/fork/helpers/ForkTest.sol";
import {Bundler} from "src/Bundler.sol";
import {EthereumGeneralAdapter1, IERC20} from "src/adapters/EthereumGeneralAdapter1.sol";

contract StEthRoundingTest is ForkTest {
    function setUp() public override {
        // Deploy adapter and bundler
        bundler = new Bundler();
        ethereumGeneralAdapter1 = new EthereumGeneralAdapter1(
            address(bundler),
            address(0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb),
            address(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2),
            address(0x6B175474E89094C44Da98b954EedeAC495271d0F),
            address(0x7f39C581F595B53c5cb19bD0b3f8dA6c935E2Ca0),
            address(0x58D97B57BB95320F9a05dC918Aef65434969c2B2),
            address(0x9D03bb2092270648d7480049d0E58d2FcF0E5123)
        );
    }

    function testWrapStEth() public {
        IERC20 stETH = IERC20(ethereumGeneralAdapter1.ST_ETH());
        // Mint 1e18 of stETH to adapter
        deal(address(stETH), address(ethereumGeneralAdapter1), 1e18);
        assertEq(stETH.balanceOf(address(ethereumGeneralAdapter1)), 1e18 - 1); // -1 here is due to rounding

        // Call wrapStEth() with amount = uint256.max
        bundle.push(_wrapStEth(type(uint256).max, RECEIVER));
        bundler.multicall(bundle);
        // Adapter has 1 wei of stETH leftover
        assertEq(stETH.balanceOf(address(ethereumGeneralAdapter1)), 1);
    }
}
```

It can be run with:

```bash
forge test -vvv --match-path test/StEthRoundingTest.t.sol --fork-url $ETHEREUM_RPC_URL --block-number 21388419
```

## Recommendation:

Document this limitation in the natspec of `EthereumGeneralAdapter1.wrapStEth()` and `CoreAdapter.erc20Transfer()`.

## Morpho:

Acknowledged. Note that it is possible to directly wrap ETH to wstETH using wstETH's receive function.

## Spearbit:

Acknowledged.

---
### Example 3

**Auto Label:** Rounding errors in integer arithmetic cause imprecise balance tracking and residual dust, leading to inconsistent state, incorrect event emissions, and potential underpayment of user deposits.  

**Original Text Preview:**

## Context
**File:** LeverageMacroBase.sol#L226

## Description
From the Lido official documentation about the stETH/wstETH integration guide, there is a specific paragraph about the "1-2 wei corner case".

The stETH balance calculation includes integer division, and there is a common case when the whole stETH balance can't be transferred from the account while leaving the last 1-2 wei on the sender's account. The same thing can actually happen at any transfer or deposit transaction. In the future, when the stETH/share rate will be greater, the error can become a bit bigger. To avoid it, one can use `transferShares` to be precise. Lido itself suggests using `transferShares` instead of `transferFrom` to avoid this 1-2 wei corner case.

## Recommendation
BadgerDAO could consider using `stETH.transferShares` instead of `stETH.transfer` in the `LeverageMacroBase.sweepToCaller` execution. If BadgerDAO decides to choose so, it should also update the `collateralBal` value properly to get the correct share balance of the contract by executing `stETH.sharesOf(address(this));`.

## Acknowledgments
- **BadgerDAO:** Acknowledged.
- **Cantina:** Acknowledged.

---
