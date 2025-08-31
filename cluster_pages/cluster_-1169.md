# Cluster -1169

**Rank:** #301  
**Count:** 24  

## Label
Missing account and mint validation enables attackers to spoof token accounts, redirect stakes, or forge valid operations—undermining system integrity and leading to unauthorized token transfers or incorrect state calculations.

## Cluster Information
- **Total Findings:** 24

## Examples

### Example 1

**Auto Label:** Failure to accurately validate or adjust token amounts leads to asset loss or unauthorized access through improper state checks or unaccounted value transfers.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex-judging/issues/626 

## Found by 
0xEkko, 0xiehnnkta, 0xlucky, 10ap17, Abhan1041, AnomX, CoheeYang, Cybrid, EgisSecurity, Ocean\_Sky, Yaneca\_b, ZeroTrust, anirruth\_, bladeee, cccz, elolpuer, hieutrinh02, hunt1, iamandreiski, malm0d, miracleworker0118, newspacexyz, roadToWatsonN101, rsam\_eth, seeques, shivansh2580, tyuuu, wellbyt3

### Summary

During zrc-20 `depositAndCall` operation, [onCall](https://github.com/sherlock-audit/2025-05-dodo-cross-chain-dex/blob/main/omni-chain-contracts/contracts/GatewayTransferNative.sol#L357) function of `GatewayTransferNative` contract will revert due to missing update of `amount` to be used in swap in DODO Router. This amount is already deducted by platform fees, therefore need to be updated prior to the swap operation. The swap execution will always pull up a whole amount but in reality this is already deducted by fees, therefore guaranteed revert due to insufficient funds. 

This can also be opportunity as attack vector for malicious attacker depending on situation like if the contract `GatewayTransferNative` has refund stored in it prior `depositAndCall` operation.

### Root Cause

The real root cause is the missing update on the `amount` to be swapped in DODO Router. This `amount` should be deducted by platform fees prior to the swap operation to avoid revert. Look at line 370, the platform fees is already physically transferred to the protocol treasury, however this amount was never updated prior to the swap operation in line 395, this will cause guaranteed revert.

```Solidity
File: GatewayTransferNative.sol
357:     function onCall(
358:         MessageContext calldata context,
359:         address zrc20, // this could be wzeta token
360:         uint256 amount, // @audit this should be updated prior swap but no deduction happened
361:         bytes calldata message
362:     ) external override onlyGateway {
363:         // Decode the message
364:         // 32 bytes(externalId) + bytes message
365:         (bytes32 externalId) = abi.decode(message[0:32], (bytes32)); 
366:         bytes calldata _message = message[32:];
367:         (DecodedNativeMessage memory decoded, MixSwapParams memory params) = SwapDataHelperLib.decodeNativeMessage(_message);
368: 
369:         // Fee for platform
370:         uint256 platformFeesForTx = _handleFeeTransfer(zrc20, amount); // @audit , fees already transferred to treasury
371:         address receiver = address(uint160(bytes20(decoded.receiver)));
372: 
~ skip
392:             );
393:         } else {
394:             // Swap on DODO Router
395:             uint256 outputAmount = _doMixSwap(decoded.swapData, amount, params); //@audit amount here is not deducted by fees, therefore revert.
396: 
```

Detail of `_doMixSwap` below
```Solidity
File: GatewayTransferNative.sol
425:     function _doMixSwap(
426:         bytes memory swapData, 
427:         uint256 amount,  //@note, expected to be equal in line 438
428:         MixSwapParams memory params
429:     ) internal returns (uint256 outputAmount) {
430:         if (swapData.length == 0) {
431:             return amount;
432:         }
433: 
434:         IZRC20(params.fromToken).approve(DODOApprove, amount);
435:         return IDODORouteProxy(DODORouteProxy).mixSwap{value: msg.value}(
436:             params.fromToken,
437:             params.toToken,
438:             params.fromTokenAmount, //@note amount to be used in swapping expected to be equal to amount in line 427
439:             params.expReturnAmount,
440:             params.minReturnAmount,
441:             params.mixAdapters,
442:             params.mixPairs,
443:             params.assetTo,
444:             params.directions,
445:             params.moreInfo,
446:             params.feeData,
447:             params.deadline
448:         );
449:     }
```

Inside `mixSwap` function found in [DODORouteProxy contract](https://zetachain-testnet.blockscout.com/address/0x026eea5c10f526153e7578E5257801f8610D1142?tab=contract)
```Solidity
_deposit(msg.sender, …, _fromTokenAmount);
…
IDODOApproveProxy.claimTokens(token, from, to, _fromTokenAmount); // _fromTokenAmount used in claiming tokens
```

Inside `claimTokens` function found in [DODOApprove contract](https://zetachain-testnet.blockscout.com/address/0x143bE32C854E4Ddce45aD48dAe3343821556D0c3?tab=contract)
```Solidity
File: DODOApprove.sol
72:     function claimTokens(
73:         address token,
74:         address who,
75:         address dest,
76:         uint256 amount //@ note , this is the _fromTokenAmount
77:     ) external {
78:         require(msg.sender == _DODO_PROXY_, "DODOApprove:Access restricted");
79:         if (amount > 0) {
80:             IERC20(token).safeTransferFrom(who, dest, amount); //@audit, this will just revert due to insufficient funds
81:         }
82:     }
```
 
The protocol may argue that the `fromTokenAmount` in_doMixSwap parameters can easily be changed by the user in order for swap to be successful. This may be correct but there are two situations we need to emphasize here.

1. If the `GatewayTransgerNative` has no refunds stored. - User is expected to use that `fromTokenAmount` is equal to the `amount` to be swap or transfer as the user is no longer expected to compute again the net remaining as this is expected for the logic contract to do its job for smooth experience of the users. The impact of this issue is just revert. If they want the swap to be successful, they will just change `fromTokenAmount` manually but unnecessary computation burden for user. 

2. If the `GatewayTransferNative` has refunds stored. - This could be an attack vector by users who knows the vulnerability. They won't change the `fromTokenAmount` and allows to use the over-allowance given to the user. Over-allowance because the allowance given exceeded the amount that should be allowed to be transferred. For example , the whole amount is 100 which is given allowance however it should be 95 only (net of 5 fees) , because 5 for fees is already transferred to treasury. If they are able to transferred the whole 100, there is a tendency that they will be able to steal some refunds amounting 5 deposited there during that time.

Please be reminded that `GatewayTransferNative` is storing funds for refunds. This could be in danger to be stolen via this vulnerability. Look at the mapping variable storage below.

```Solidity
File: GatewayTransferNative.sol
32:     mapping(bytes32 => RefundInfo) public refundInfos; // externalId => RefundInfo, storage for refund records
```

### Internal Pre-conditions

1. When cross-chain operation involves DODO router swap call in destination chain Zetachain.

### External Pre-conditions

none

### Attack Path

Scenario A:
This could be the scenario if the the `GatewayTransferNative` has no refunds stored.

 Step                             | Contract balance (zrc-20) | Allowance set     | RouteProxy tries to pull | Outcome                                       |
|----------------------------------|-------------------------|-------------------|--------------------------|-----------------------------------------------|
|  Deposited 1,000 zrc-20 amount from `depositAndCall`                        | 1 000                   | –                 | –                        | –                                             |
| `_handleFeeTransfer` (0.5 %)     | **995**                 | –                 | –                        | Treasury receives 5                           |
| `_doMixSwap`              | 995                     | **approve 1 000** | –                        | approved the amount which still includes the fees transferred.                                            |
| `_deposit → claimTokens`        | 995                     | 1 000             | **1 000**                | **Revert (insufficient balance)**             |
| Swap aborted                     | –                       | –                 | –                        | `onCall` reverts, swap operation cancelled        |

Scenario B:
This could be the scenario if the the `GatewayTransferNative` has refunds stored.

 Step                             | Contract balance (zrc-20) | Allowance set     | RouteProxy tries to pull | Outcome                                       |
|----------------------------------|-------------------------|-------------------|--------------------------|-----------------------------------------------|
| 5 zrc-20 tokens refund currently stored  in contract                     | 5                   | –                 | –                        | –                                             |
| Deposited 1,000 zrc-20 amount from `depositAndCall`                       | 1 005                   | –                 | –                        | –                                             |
| `_handleFeeTransfer` (0.5 %)     | **1000**                 | –                 | –                        | Treasury receives 5                           |
| `_doMixSwap`               | 1000                     | **approve 1 000** | –                        | approved the amount which still includes the fees transferred.                                            |
| `_deposit → claimTokens`        | 1000                     | 1 000             | **1 000**                | **OK (sufficient balance)**             |
| Swap successful                     | –                       | –                 | –                        | `onCall` success, excess of 5 is pulled out from refund       |

As you can see here, the refund of 5 zrc-20 tokens is pulled out from the last step (included in 1000 amount). This can be equivalent to stolen funds as the contract has no remaining funds left.

### Impact

The onCall function will either revert or can cause loss of funds.

### PoC

see attack path

### Mitigation

Update the amount prior the swap operation.

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/Skyewwww/omni-chain-contracts/pull/29

---
### Example 2

**Auto Label:** Failure to validate account ownership or token identity leads to unauthorized access, mint manipulation, or invalid state transitions, enabling attackers to bypass authorization and exploit token or account mismatches.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2025-04-zetachain-cross-chain-judging/issues/421 

## Found by 
0xkmr\_, OhmOudits, adeolu, berndartmueller, chinepun, g, mahdiRostami

### Summary

The Gateway Solana program contains a critical vulnerability in its `handle_spl` function within the deposit.rs file. While the program requires a whitelist entry account for the SPL token being deposited, it does not verify that the token being transferred from the user's account actually matches the whitelisted token specified in the transaction context. This mismatch allows an attacker to deposit a non-whitelisted token while claiming it's a different, whitelisted token. The ZetaChain node, which processes these deposits, will incorrectly identify the deposited token based on the mint address in the transaction context rather than the actual token being transferred, potentially leading to incorrect cross-chain asset transfers and economic damage.


snippet of the DepositSplToken context below 

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/contexts.rs#L76C1-L89C1
```rust 
#[derive(Accounts)]
pub struct DepositSplToken<'info> {
    /// The account of the signer making the deposit.
    #[account(mut)]
    pub signer: Signer<'info>,

    /// Gateway PDA.
    #[account(mut, seeds = [b"meta"], bump)]
    pub pda: Account<'info, Pda>,

    /// The whitelist entry account for the SPL token.
    #[account(seeds = [b"whitelist", mint_account.key().as_ref()], bump)]
    pub whitelist_entry: Account<'info, WhitelistEntry>,

    /// The mint account of the SPL token being deposited.
    pub mint_account: Account<'info, Mint>,

    /// The token program.
    pub token_program: Program<'info, Token>,

    /// The source token account owned by the signer.
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,

    /// The destination token account owned by the PDA.
    #[account(mut)]
    pub to: Account<'info, TokenAccount>,

    /// The system program.
    pub system_program: Program<'info, System>,
}
```

`mint_account` which is used in the `whitelist_entry` can be different from the token_program and to.mint or from.mint account. 

### Root Cause

The deposit process for SPL tokens involves multiple components that interact as follows:

1. The process begins when a user calls the `deposit_spl_token` function in lib.rs:

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/lib.rs#L236-L242
```rust
pub fn deposit_spl_token(
    ctx: Context<DepositSplToken>,
    amount: u64,
    receiver: [u8; 20],
    revert_options: Option<RevertOptions>,
) -> Result<()> {
    instructions::deposit::handle_spl(ctx, amount, receiver, revert_options, DEPOSIT_FEE)
}
```

2. This function calls `handle_spl` in deposit.rs, which performs the actual deposit operation:

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/instructions/deposit.rs#L68-L77
```rust
pub fn handle_spl(
    ctx: Context<DepositSplToken>,
    amount: u64,
    receiver: [u8; 20],
    revert_options: Option<RevertOptions>,
    deposit_fee: u64,
) -> Result<()> {
    verify_payload_size(None, &revert_options)?;
    let token = &ctx.accounts.token_program; //@audit no check for if token is whitelisted, or is for same one represented by ctx.accounts.mint_account
    //this disparity can affect the final messaqge emitted, and can affect node processing. i.e if two whitelisted tokens usdc and weth, attacker can ust mint_account for weth and send in 10 usdc
    //msg willl emit weth as mint_account and node will process deposit on the other side as weth deposit
    let from = &ctx.accounts.from;
    
    // ... rest of the function ...
}
```

3. The `DepositSplToken` context includes a `whitelist_entry` account that is derived from the mint address:

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/contexts.rs#L76C1-L105
```rust
#[derive(Accounts)]
pub struct DepositSplToken<'info> {
    // ... other fields ...

    /// The whitelist entry account for the SPL token.
    #[account(seeds = [b"whitelist", mint_account.key().as_ref()], bump)]
    pub whitelist_entry: Account<'info, WhitelistEntry>,

    /// The mint account of the SPL token being deposited.
    pub mint_account: Account<'info, Mint>,

    /// The token program.
    pub token_program: Program<'info, Token>,

    /// The source token account owned by the signer.
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,
    
    // ... other fields ...
}
```

And there is no check in the logic to verify that token_program is the valid one belonging to the mint_account provided and the same for to.mint and from.mint. Full function logic below 

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/instructions/deposit.rs#L68-L119
```rust 
pub fn handle_spl(
    ctx: Context<DepositSplToken>,
    amount: u64,
    receiver: [u8; 20],
    revert_options: Option<RevertOptions>,
    deposit_fee: u64,
) -> Result<()> {
    verify_payload_size(None, &revert_options)?;
    let token = &ctx.accounts.token_program; 
    let from = &ctx.accounts.from;

    let pda = &mut ctx.accounts.pda;
    require!(!pda.deposit_paused, Errors::DepositPaused);
    require!(receiver != [0u8; 20], Errors::EmptyReceiver);

    let cpi_context = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer {
            from: ctx.accounts.signer.to_account_info().clone(),
            to: pda.to_account_info().clone(),
        },
    );
    system_program::transfer(cpi_context, deposit_fee)?;

    let pda_ata = get_associated_token_address(&ctx.accounts.pda.key(), &from.mint);
    require!(
        pda_ata == ctx.accounts.to.to_account_info().key(), 
        Errors::DepositToAddressMismatch
    );

    let xfer_ctx = CpiContext::new(
        token.to_account_info(),
        anchor_spl::token::Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(), 
            authority: ctx.accounts.signer.to_account_info(),
        },  
    );
    transfer(xfer_ctx, amount)?;

    msg!(
            "Deposit SPL executed: amount = {}, fee = {}, receiver = {:?}, pda = {}, mint = {}, revert options = {:?}",
            amount,
            deposit_fee,
            receiver,
            ctx.accounts.pda.key(),
            ctx.accounts.mint_account.key(),
            revert_options
        );

    Ok(())
}
```

The logic sets `let token = &ctx.accounts.token_program;`, then fetches the pda_ata via `    let pda_ata = get_associated_token_address(&ctx.accounts.pda.key(), &from.mint);` after which it does a cip_contrxt transfer like below 

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/protocol-contracts-solana/programs/gateway/src/instructions/deposit.rs#L98-L106
```rust 
    let xfer_ctx = CpiContext::new(
        token.to_account_info(),
        anchor_spl::token::Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(), 
            authority: ctx.accounts.signer.to_account_info(),
        }, //@audit no freeze authority check too for if pda_ata is frozen before transfer. 
    );
    transfer(xfer_ctx, amount)?;
```
here the token being transferred is `token` or `ctx.accounts.token_program`. the pda_ata of the program is checked to be same as one generated for the from.mint account representing the token. 

token_program, from.mint and to.mint can actually represent the same token while mint_account may not. if token_program, from.mint and to.mint represent same token the token transfer will be sucessfull. if mint_account represents another whitelisted token, the whitelist check will succeed too. 

4. After the deposit is processed, the ZetaChain node observes the transaction and extracts the deposit information:

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/zetaclient/chains/solana/observer/inbound.go#L224-L251
```go
// In FilterInboundEvents function (node/zetaclient/chains/solana/observer/inbound.go)
deposit, err := solanacontracts.ParseInboundAsDepositSPL(tx, i, txResult.Slot)
if err != nil {
    return nil, errors.Wrap(err, "error ParseInboundAsDepositSPL")
} else if deposit != nil {
    seenDepositSPL = true
    events = append(events, &clienttypes.InboundEvent{
        // ... other fields ...
        CoinType: coin.CoinType_ERC20,
        Asset:    deposit.Asset,  // This is the mint address from the transaction
        // ... other fields ...
    })
}
```

After this, the parsing is done here 

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L134-L146


https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L149-L173

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L312-L333

5. The node extracts the SPL token mint address from the instruction accounts:

https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L329-L338
```go
// In getSignerAndSPLFromDepositSPLAccounts function (node/pkg/contracts/solana/inbound.go)
// the accounts are [signer, pda, whitelist_entry, mint_account, token_program, from, to]
signer := instructionAccounts[0].PublicKey.String()
spl := instructionAccounts[3].PublicKey.String()  // This is the mint_account from the transaction

return signer, spl, nil
```

The issue is that the `handle_spl` function does not verify that the token in the user's `from` account matches the mint address specified in the transaction context. The Anchor framework ensures that a whitelist entry exists for the provided `mint_account`, but there's no check to ensure that the token being transferred has the same mint.


The mint_account is THEN taken as the spl token account and is returned by [getSignerAndSPLFromDepositSPLAccounts](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L338). this is then returned as the asset deposited in [parseAsDepositSPL()](https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L186)


https://github.com/sherlock-audit/2025-04-zetachain-cross-chain/blob/main/node/pkg/contracts/solana/inbound.go#L170-L186
```rust 
	sender, spl, err := getSignerAndSPLFromDepositSPLAccounts(tx, &instruction)
	if err != nil {
		return nil, err
	}


	receiver, err := parseReceiver(inst.Receiver)
	if err != nil {
		return nil, err
	}


	return &Inbound{
		Sender:           sender,
		Receiver:         receiver,
		Amount:           inst.Amount,
		Memo:             []byte{},
		Slot:             slot,
		Asset:            spl, // <-- see here this is mint_account, may not be the token program or from.mint transferred

```

### Internal Pre-conditions


1. The Gateway program has a whitelist mechanism for SPL tokens, implemented through the `whitelist_entry` account in the `DepositSplToken` context.

2. Multiple tokens have been whitelisted in the system (e.g., USDC and WETH).

3. The `handle_spl` function is called with a token program and mint account that should be checked for consistency.

4. The node processes SPL token deposits by extracting the mint address from the transaction context, not from the actual token being transferred.

### External Pre-conditions

none

### Attack Path


1. An attacker identifies two whitelisted tokens in the system, for example, USDC and WETH.

2. The attacker creates a transaction that includes:
   - A legitimate whitelisted token's mint address (e.g., WETH) in the `mint_account` field
   - The corresponding whitelist entry for WETH
   - But actually transfers a different token (e.g., USDC) from their `from` account to the `to` account which is the program pda_ata. already created since USDC is whitelisted 

3. The `handle_spl` function does not verify that the token being transferred matches the mint address specified in the transaction.

4. The transaction is processed successfully, and the log message emits information that a WETH deposit was made, even though USDC was actually transferred.

5. The ZetaChain node observes this transaction and processes it as a WETH deposit based on the mint address in the transaction context.

6. This results in a mismatch between the actual token deposited (USDC) and what the system records and processes (WETH), potentially leading to incorrect asset transfers and accounting.

### Impact

1. **Token Spoofing**: Attackers can deposit one token (e.g., USDC) while making the system believe they deposited another token (e.g., WETH).

2. **Cross-Chain Asset Mismatch**: The ZetaChain node will process the deposit based on the mint address in the transaction, not the actual token being transferred, potentially leading to incorrect asset transfers on the ZetaChain side.

3. **Economic Damage**: If the value of the spoofed token is different from the actual token being transferred, this could lead to economic damage or exploitation of the protocol.

4. **Protocol Inconsistency**: The mismatch between the actual token deposited and what the system records could lead to inconsistencies in the protocol state.



### Mitigation

1. **Implement Token Validation**: Add explicit validation in the `handle_spl` function to ensure the token being deposited matches the mint address specified in the transaction:

```rust
pub fn handle_spl(
    ctx: Context<DepositSplToken>,
    amount: u64,
    receiver: [u8; 20],
    revert_options: Option<RevertOptions>,
    deposit_fee: u64,
) -> Result<()> {
    verify_payload_size(None, &revert_options)?;
    
    // Ensure the token being deposited matches the mint address in the transaction
    require!(
        ctx.accounts.from.mint == ctx.accounts.mint_account.key(),
        Errors::TokenMintMismatch
    );
    
    let token = &ctx.accounts.token_program;
    let from = &ctx.accounts.from;
    
    // ... rest of the function ...
}
```

## Discussion

**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/zeta-chain/protocol-contracts-solana/pull/114


**gjaldon**

The mint of the from token account is now [validated](https://github.com/zeta-chain/protocol-contracts-solana/pull/114/files\#diff-e9aa90c11c8df3130c0cc027f1d5fd6f429d44a4c1ec35eaeaa40cc280841fc2R97) against the mint_account in the signed payload.

---
### Example 3

**Auto Label:** Missing account and mint validation enables attackers to spoof token accounts, redirect stakes, or forge valid operations—undermining system integrity and leading to unauthorized token transfers or incorrect state calculations.  

**Original Text Preview:**

## Staking Action Vulnerability

The `staking_action::stake` function fails to properly verify the collection associated with the staked NFT. Currently, the function checks whether the `collection.key` field in `nft_metadata` matches a predefined collection address (`CLAYNO_COLLECTION_ADDRESS`). However, this validation is insufficient, as it only checks the key field of the collection structure but overlooks the `verified` field. The `verified` field ensures that the collection has been officially validated by the authority responsible for the collection.

## Code Snippet

```rust
/// Stakes an NFT by delegating it to the global authority PDA.
pub fn stake(ctx: Context<StakingAction>) -> Result<()> {
    [...]
    // Deserialize Metadata to verify collection
    let nft_metadata = Metadata::safe_deserialize(&mut ctx.accounts.nft_metadata.to_account_info().data.borrow_mut()).unwrap();
    if let Some(collection) = nft_metadata.collection {
        if collection.key.to_string() != CLAYNO_COLLECTION_ADDRESS {
            return Err(error!(StakingError::WrongCollection));
        }
    } else {
        return Err(error!(StakingError::InvalidMetadata));
    };
    [...]
}
```

This may only be set to true if the authority has run one of the Token Metadata Verify instructions over the NFT. Thus, only verifying the key field enables an attacker to create an NFT with a `collection.key` that matches the expected `CLAYNO_COLLECTION_ADDRESS`. Consequently, the NFT appears valid even if it is not really part of the intended collection. This means that it is possible for an attacker to create a fake NFT with the correct key but without having the `verified` field set to true, which implies the collection is not validated, enabling the attacker to stake an invalid NFT.

## Remediation

Ensure that the `collection.verified` field is set to true along with verifying that `collection.key` matches the mint address of the appropriate collection parent.

## Patch

Fixed in commit `0b87d6e`.

---
