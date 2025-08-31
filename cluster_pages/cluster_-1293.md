# Cluster -1293

**Rank:** #26  
**Count:** 695  

## Label
Use of unchecked arithmetic operations bypasses overflow/underflow checks, introducing risks of incorrect state transitions, exploitable integer overflows, and compromised arithmetic integrity in critical financial calculations.

## Cluster Information
- **Total Findings:** 695

## Examples

### Example 1

**Auto Label:** Unsafe arithmetic operations without overflow/underflow checks lead to incorrect state updates, enabling data corruption, unauthorized access, or exploitable behavior through arithmetic wrapping.  

**Original Text Preview:**

In the [`_swapAndBridge` function](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L575), the adjusted output amount is [calculated](https://github.com/across-protocol/contracts/blob/b84dbfae35030e0f2caa5509b632c10106a32330/contracts/SpokePoolPeriphery.sol#L604-L606) as the product of `depositData.outputAmount` and `returnAmount` divided by `minExpectedInputTokenAmount`. If `depositData.outputAmount * returnAmount` exceeds `2^256–1`, the transaction will revert immediately on the multiply step, even when the eventual division result would fit. This intermediate overflow is invisible to users, who only see a generic failure without an explanatory error message.

Consider using OpenZeppelin’s [`Math.mulDiv(a, b, c)`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/48bd2864c6c696bf424ee0e2195f2d72ddd1a86c/contracts/utils/math/Math.sol#L204) to compute `floor(a*b/c)` without intermediate overflow. Alternatively, consider documenting the possible overflow scenario.

***Update:** Resolved in [pull request #1020](https://github.com/across-protocol/contracts/pull/1020) at commit [`e872f04`](https://github.com/across-protocol/contracts/pull/1020/commits/e872f045bd2bbdb42d8ca74c2133d0afa63ae07b) by documenting the potential overflow scenario.*

---
### Example 2

**Auto Label:** Unsafe arithmetic operations without overflow/underflow checks lead to incorrect state updates, enabling data corruption, unauthorized access, or exploitable behavior through arithmetic wrapping.  

**Original Text Preview:**

In the CLGauge contract, both the `deposit` and `withdraw` functions perform unsafe type casting from `uint128` to `int128` when interacting with the pool's stake function. If the liquidity value is large (exceeding 2^127-1), this conversion could result in overflow, leading to incorrect liquidity values being staked or withdrawn from the pool.

```solidity
    function deposit(
        uint256 nfpTokenId,
        uint256 tokenId
    ) public lock actionLock {
        ...
        // stake nfp liquidity in pool
        pool.stake(int128(_liquidity), _tickLower, _tickUpper, true); // @audit unsafe cast to int128 can lead to overflow
        ...
    }
```

```solidity
    function withdraw(uint nfpTokenId) public lock actionLock {
        ...
        if (_liquidity != 0)
            pool.stake(-int128(_liquidity), _tickLower, _tickUpper, true); // @audit unsafe cast to int128 can lead to overflow
        ...
    }
```

It's recommended to use SafeCast library to cast liquidity to int128.

---
### Example 3

**Auto Label:** Unsafe arithmetic operations without overflow/underflow checks lead to incorrect state updates, enabling data corruption, unauthorized access, or exploitable behavior through arithmetic wrapping.  

**Original Text Preview:**

In the CLGauge contract, both the `deposit` and `withdraw` functions perform unsafe type casting from `uint128` to `int128` when interacting with the pool's stake function. If the liquidity value is large (exceeding 2^127-1), this conversion could result in overflow, leading to incorrect liquidity values being staked or withdrawn from the pool.

```solidity
    function deposit(
        uint256 nfpTokenId,
        uint256 tokenId
    ) public lock actionLock {
        ...
        // stake nfp liquidity in pool
        pool.stake(int128(_liquidity), _tickLower, _tickUpper, true); // @audit unsafe cast to int128 can lead to overflow
        ...
    }
```

```solidity
    function withdraw(uint nfpTokenId) public lock actionLock {
        ...
        if (_liquidity != 0)
            pool.stake(-int128(_liquidity), _tickLower, _tickUpper, true); // @audit unsafe cast to int128 can lead to overflow
        ...
    }
```

It's recommended to use SafeCast library to cast liquidity to int128.

---
