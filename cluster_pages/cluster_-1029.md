# Cluster -1029

**Rank:** #197  
**Count:** 56  

## Label
Failure to validate or properly handle return values from critical operations leads to silent failures, loss of execution data, and undetected state changes, undermining security, transparency, and correctness.

## Cluster Information
- **Total Findings:** 56

## Examples

### Example 1

**Auto Label:** Inconsistent or meaningless return values that misrepresent execution outcomes, leading to unpredictable behavior, incorrect state assumptions, and increased attack surface.  

**Original Text Preview:**

**Description:** `AmAmm::_updateAmAmmWrite` currently returns the manager of the top bid and its associated payload; however, in all invocations of this function the return values are never used. If they are not required, these unused return values should be removed.

**Bacon Labs:** Fixed in [PR \#7](https://github.com/Bunniapp/biddog/pull/7).

**Cyfrin:** Verified, the unused return value has been removed.

---
### Example 2

**Auto Label:** Inconsistent or meaningless return values that misrepresent execution outcomes, leading to unpredictable behavior, incorrect state assumptions, and increased attack surface.  

**Original Text Preview:**

**Description:** Remove obsolete `return` statements when already using named returns:
```diff
    function _mintSoulBoundToken(address account) internal returns (uint256 tokenId) {
        tokenId = _incrementTokenIdCounter(1);
        _safeMint(account, tokenId);
-       return tokenId;
    }

    function _incrementTokenIdCounter(uint256 count) internal returns (uint256 startId) {
        startId = s_tokenIdCounter;
        s_tokenIdCounter += count;
-       return startId;
    }
```

**Evo:**
Fixed in commit [f594ae0](https://github.com/contractlevel/sbt/commit/f594ae004d4afc80f19e17c0f61d50caa00a4811).

**Cyfrin:** Verified.

---
### Example 3

**Auto Label:** Inconsistent or meaningless return values that misrepresent execution outcomes, leading to unpredictable behavior, incorrect state assumptions, and increased attack surface.  

**Original Text Preview:**

##### Description

Throughout the contract, the return values of the functions are not named. Using named output parameters can improve code readability and make it easier to understand the purpose of each return value, especially when the function has multiple return values.

##### BVSS

[AO:S/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N (0.0)](/bvss?q=AO:S/AC:L/AX:L/R:N/S:U/C:N/A:N/I:N/D:N/Y:N)

##### Recommendation

Use named output parameters in all functions to improve code readability and maintainability, complementing the Natspec documentation.

##### Remediation

**ACKNOWLEDGED:** The **Jigsaw Protocol team** made a business decision to acknowledge this finding and not alter the contracts.

##### References

[jigsaw-finance/jigsaw-strategies-v1/src/pendle/PendleStrategy.sol#L226](https://github.com/jigsaw-finance/jigsaw-strategies-v1/blob/9ecef78ef8cb421640c0b3bb449b3fa43ce35f5a/src/pendle/PendleStrategy.sol#L226)

[jigsaw-finance/jigsaw-strategies-v1/src/pendle/PendleStrategy.sol#L294](https://github.com/jigsaw-finance/jigsaw-strategies-v1/blob/9ecef78ef8cb421640c0b3bb449b3fa43ce35f5a/src/pendle/PendleStrategy.sol#L294)

---
