# Cluster -1142

**Rank:** #170  
**Count:** 73  

## Label
Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.

## Cluster Information
- **Total Findings:** 73

## Examples

### Example 1

**Auto Label:** Failure to enforce initialization or finalization states allows unauthorized state modifications, leading to unintended behavior, loss of funds, or compromised immutability in critical contract operations.  

**Original Text Preview:**

The `internal` function `_init` within `ERC721WithURIBuilderUpgradeable.sol` lacks the `onlyInitializing` modifier. This function is called by the `initializer` function (`init`) of the upgradeable implementation contract `PositionsManager.sol`.

While the main `PositionsManager.init` function _is_ protected by the `initializer` modifier, and the underlying `__ERC721_init` called by `_init` also has protection, the absence of the guard on `_init` itself poses a latent risk. If future upgrades to `PositionsManager` were to introduce new functions that mistakenly call `_init` again after initial deployment, there would be no direct protection at the `_init` level to prevent this call attempt.

Recommendation:

Add the `onlyInitializing` modifier to the `ERC721WithURIBuilderUpgradeable._init` function to explicitly enforce its initialization-only context and adhere to standard upgradeable contract patterns.

```solidity
    function _init(string memory name_, string memory symbol_) internal onlyInitializing {
        __ERC721_init(name_, symbol_);
    }
```

---
### Example 2

**Auto Label:** **Improper validation and state mutation lead to unintended asset flows, silent failures, or denial of service, compromising contract integrity and user asset safety.**  

**Original Text Preview:**

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The first `bond` caller could cause DoS by providing a dust amount of token if they provide `_amount` that is enough to make `_tokensMinted` but cause the rest of `_transferAmt` to be 0.

```solidity
    function _bond(address _token, uint256 _amount, uint256 _amountMintMin, address _user) internal {
        require(_isTokenInIndex[_token], "IT");
        uint256 _tokenIdx = _fundTokenIdx[_token];

        bool _firstIn = _isFirstIn();
        uint256 _tokenAmtSupplyRatioX96 =
            _firstIn ? FixedPoint96.Q96 : (_amount * FixedPoint96.Q96) / _totalAssets[_token];
        uint256 _tokensMinted;
        if (_firstIn) {
            _tokensMinted = (_amount * FixedPoint96.Q96 * 10 ** decimals()) / indexTokens[_tokenIdx].q1;
        } else {
            _tokensMinted = (_totalSupply * _tokenAmtSupplyRatioX96) / FixedPoint96.Q96;
        }
        uint256 _feeTokens = _canWrapFeeFree(_user) ? 0 : (_tokensMinted * fees.bond) / DEN;
        require(_tokensMinted - _feeTokens >= _amountMintMin, "M");
        _totalSupply += _tokensMinted;
        _mint(_user, _tokensMinted - _feeTokens);
        if (_feeTokens > 0) {
            _mint(address(this), _feeTokens);
            _processBurnFee(_feeTokens);
        }
        uint256 _il = indexTokens.length;
        for (uint256 _i; _i < _il; _i++) {
>>>         uint256 _transferAmt = _firstIn
                ? getInitialAmount(_token, _amount, indexTokens[_i].token)
                // (_amount * FixedPoint96.Q96) / _totalAssets[_token];
                : (_totalAssets[indexTokens[_i].token] * _tokenAmtSupplyRatioX96) / FixedPoint96.Q96;
>>>         _totalAssets[indexTokens[_i].token] += _transferAmt;
            _transferFromAndValidate(IERC20(indexTokens[_i].token), _user, _transferAmt);
        }
        _internalBond();
        emit Bond(_user, _token, _amount, _tokensMinted);
    }
```

```solidity
    function getInitialAmount(address _sourceToken, uint256 _sourceAmount, address _targetToken)
        public
        view
        override
        returns (uint256)
    {
        uint256 _sourceTokenIdx = _fundTokenIdx[_sourceToken];
        uint256 _targetTokenIdx = _fundTokenIdx[_targetToken];
        return (_sourceAmount * indexTokens[_targetTokenIdx].weighting * 10 ** IERC20Metadata(_targetToken).decimals())
            / indexTokens[_sourceTokenIdx].weighting / 10 ** IERC20Metadata(_sourceToken).decimals();
    }
```

This will cause the total supply to increase to a non-zero value, while the `_totalAssets` for assets other than the provided `_token` remains zero. This can lead to a DoS because if other users try to `bond` using tokens different from the attacker’s used token, the function will revert due to `_totalAssets` being zero. If users use the same tokens as the attacker, the remaining other tokens will never be used or increased, since `_totalAssets` is always zero when calculating `_transferAmt`.

```solidity
    function _bond(address _token, uint256 _amount, uint256 _amountMintMin, address _user) internal {
        require(_isTokenInIndex[_token], "IT");
        uint256 _tokenIdx = _fundTokenIdx[_token];

        bool _firstIn = _isFirstIn();
        uint256 _tokenAmtSupplyRatioX96 =
>>>         _firstIn ? FixedPoint96.Q96 : (_amount * FixedPoint96.Q96) / _totalAssets[_token];
        uint256 _tokensMinted;
        if (_firstIn) {
            _tokensMinted = (_amount * FixedPoint96.Q96 * 10 ** decimals()) / indexTokens[_tokenIdx].q1;
        } else {
            _tokensMinted = (_totalSupply * _tokenAmtSupplyRatioX96) / FixedPoint96.Q96;
        }
        uint256 _feeTokens = _canWrapFeeFree(_user) ? 0 : (_tokensMinted * fees.bond) / DEN;
        require(_tokensMinted - _feeTokens >= _amountMintMin, "M");
        _totalSupply += _tokensMinted;
        _mint(_user, _tokensMinted - _feeTokens);
        if (_feeTokens > 0) {
            _mint(address(this), _feeTokens);
            _processBurnFee(_feeTokens);
        }
        uint256 _il = indexTokens.length;
        for (uint256 _i; _i < _il; _i++) {
            uint256 _transferAmt = _firstIn
                ? getInitialAmount(_token, _amount, indexTokens[_i].token)
                // (_amount * FixedPoint96.Q96) / _totalAssets[_token];
 >>>            : (_totalAssets[indexTokens[_i].token] * _tokenAmtSupplyRatioX96) / FixedPoint96.Q96;
            _totalAssets[indexTokens[_i].token] += _transferAmt;
            _transferFromAndValidate(IERC20(indexTokens[_i].token), _user, _transferAmt);
        }
        _internalBond();
        emit Bond(_user, _token, _amount, _tokensMinted);
    }
```

## Recommendations

Consider to revert when the calculated `_transferAmt` is 0.

---
### Example 3

**Auto Label:** Failure to validate or check contract state before execution leads to incorrect state updates, unauthorized operations, or silent failures, risking asset loss, inconsistent accounting, or exploitation during invalid or transitional states.  

**Original Text Preview:**

##### Description

The logic in the `stake` function includes two inefficiencies:

  

1. **Redundant Balance Comparison**: The function calculates `stakingAmount` by comparing the contract’s token balance before and after the transfer. This approach is unnecessary and gas-inefficient. Instead, the `_amount` parameter can be used directly, or it can be replaced with more accurate validations, such as checking the sender's balance and allowance.
2. **Unnecessary Conditional Update**: The user’s stake (`userStake.amount`) is updated with conditional logic based on whether `userStake.amount > 0`. This separation is redundant, as a direct addition would achieve the same result with less complexity.

  

Code Location
-------------

Code of `stake` function from **Staking.sol** contract.

```
Stake storage userStake = stakeInfo[_beneficiary];

uint256 balanceBefore = IERC20(stakingToken).balanceOf(address(this));
IERC20(stakingToken).safeTransferFrom(
	msg.sender,
	address(this),
	_amount
);
uint256 balanceAfter = IERC20(stakingToken).balanceOf(address(this));

uint256 stakingAmount = balanceAfter - balanceBefore;

// Update user's stake
if (userStake.amount > 0) {
	userStake.amount += stakingAmount;
} else {
	userStake.amount = stakingAmount;
}

totalStaked += stakingAmount;
```

##### BVSS

[AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N (0.0)](/bvss?q=AO:A/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N)

##### Recommendation

It is recommended to:

  

1. Replace the balance comparison logic with either directly using the `_amount` parameter or validating the sender’s balance and allowance beforehand to ensure they have sufficient tokens and approval.
2. Simplify the logic for updating `userStake.amount` by directly adding the staking amount to the existing value, regardless of whether it is initially zero.

##### Remediation

**ACKNOWLEDGED:** The **Plume team** has acknowledged this finding.

---
