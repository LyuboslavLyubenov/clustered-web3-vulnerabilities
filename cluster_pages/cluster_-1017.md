# Cluster -1017

**Rank:** #405  
**Count:** 11  

## Label
Missing input and state validation leads to unauthorized fund withdrawals, invalid state transitions, and potential financial loss through improper checks on participation, game status, and balance constraints.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Missing state validation and input checks enable unauthorized state transitions or invalid operations, leading to critical failures in contract logic and potential misuse of funds.  

**Original Text Preview:**

##### Description

The `burnAndBridge()` and `burnAndBridgeMulti()` functions in AssetController.sol lack input validation for the amount parameter. This allows transactions to be executed with zero amounts:

```
function burnAndBridge(
    address recipient,
    uint256 amount,
    bool unwrap,
    uint256 destChainId,
    address bridgeAdapter
) public payable nonReentrant whenNotPaused {
    uint256 _currentLimit = burningCurrentLimitOf(bridgeAdapter);
    if (_currentLimit < amount) revert IXERC20_NotHighEnoughLimits();
    _useBurnerLimits(bridgeAdapter, amount);
    IXERC20(token).burn(_msgSender(), amount);// @audit - amount is not checked for > 0// ...
}

function burnAndBridgeMulti(
    address recipient,
    uint256 amount,
    bool unwrap,
    uint256 destChainId,
    address[] memory adapters,
    uint256[] memory fees
) public payable nonReentrant whenNotPaused {
// ...
    IXERC20(token).burn(_msgSender(), amount);// @audit - amount is not checked for > 0// ...
}
```

  

The absence of zero-amount validation allows unnecessary transactions that consume gas without transferring value

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C (0.0)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:N/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

It is recommended to add a zero-amount check at the start of both functions:

```
function burnAndBridge(...) public payable nonReentrant whenNotPaused {
    if (amount == 0) revert InvalidAmount();
// rest of the function
}

function burnAndBridgeMulti(...) public payable nonReentrant whenNotPaused {
    if (amount == 0) revert InvalidAmount();
// rest of the function
}
```

##### Remediation

**SOLVED:** Zero-amount check has been added.

##### Remediation Hash

<https://github.com/LucidLabsFi/demos-contracts-v1/commit/cb840eb7f7df4b63ac7207a41fdb74af617fb6b4>

##### References

[LucidLabsFi/demos-contracts-v1/contracts/modules/chain-abstraction/AssetController.sol#L235](https://github.com/LucidLabsFi/demos-contracts-v1/blob/main/contracts/modules/chain-abstraction/AssetController.sol#L235)

---
### Example 2

**Auto Label:** Missing input and state validation leads to unauthorized fund withdrawals, invalid state transitions, and potential financial loss through improper checks on participation, game status, and balance constraints.  

**Original Text Preview:**

`emergencyEnd` function can be used with a player who has no game.

The function must check that the player is currently in a game.

```diff
     /// @dev In emergency scenarios where players are locked, this function can be used to unlock player and pay
     function emergencyEnd(address player, uint256 payout) external onlyAdmin whenNotPaused {
+        if (_strEq(playerGameId[player], "")) revert PlayerNotInGame(player);
         playerGameId[player] = "";

         balance[player] += payout;
```

---
### Example 3

**Auto Label:** Missing input and state validation leads to unauthorized fund withdrawals, invalid state transitions, and potential financial loss through improper checks on participation, game status, and balance constraints.  

**Original Text Preview:**

The `endGame` function can revert when players lose money and their balance doesn't cover the loss.

This is likely to happen when a player is slashed (reducing balance).
Then, `endGame` will revert due to insufficient balance.

```solidity
    function endGame(string calldata gameId, PlayerPayout[] memory playerPayouts) external onlyAdmin whenNotPaused {
        ...

        for (uint256 i = 0; i < playerPayouts.length; i++) {
            ...

            if (playerPayout.payout > playerPayout.stake) {
                // Player won money case
                ...
            } else {
                // Player lost money case

                uint256 deduction = playerPayout.stake - playerPayout.payout;

                if (balance[player] < deduction) revert InsufficientBalance(); // @POC: balance can be lower than deduction, reverting the transaction

                // deduct money from players' balance
                balance[player] -= deduction;
            }

            ...
        }

        ...
    }
```

When a player is slashed, the player should be out of the game.

---
