# Cluster -1104

**Rank:** #398  
**Count:** 12  

## Label
Failure to validate token uniqueness and identity leads to invalid pool states, enabling incorrect balance updates, duplicate pair registrations, and potential fund extraction through improper address comparison or indexing.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** Failure to validate token uniqueness and identity leads to invalid pool states, enabling incorrect balance updates, duplicate pair registrations, and potential fund extraction through improper address comparison or indexing.  

**Original Text Preview:**

##### Description

The `addPairInfo` function fails to validate and enforce critical constraints:

* **Token Equality**: No check ensures that `token0` and `token1` are distinct.
* **Token Standardization**: Tokens are not ordered consistently (e.g., lexicographically or by asset type).
* **Duplicate Pairs**: Multiple identical trading pairs (e.g., `ETH/USD`) can be added due to lack of uniqueness checks.

##### BVSS

[AO:A/AC:L/AX:L/C:N/I:L/A:N/D:N/Y:N/R:N/S:C (3.1)](/bvss?q=AO:A/AC:L/AX:L/C:N/I:L/A:N/D:N/Y:N/R:N/S:C)

##### Recommendation

* **Validate Token Equality**: Add a requirement that `token0` and `token1` are distinct.

```
    require(token0 != token1, "Token0 and Token1 must be different");

```

* **Enforce Standardized Token Order**: Ensure `token0` is always the smaller address or the "base" token by predefined logic.

```
    if (token0 > token1) {
        (token0, token1) = (token1, token0);
    }

```

* **Prevent Duplicate Pairs**: Use a mapping to ensure each pair (`token0`, `token1`) is unique.

```
    mapping(address => mapping(address => bool)) public pairExists;

    require(!pairExists[token0][token1], "Trading pair already exists");
    pairExists[token0][token1] = true;

```

These changes ensure the integrity and consistency of trading pair configurations.

##### Remediation

**RISK ACCEPTED**: This problem does exist. However, the administrator will pay attention to the parameters.

---
### Example 2

**Auto Label:** Insufficient validation of chain state and token parameters leads to balance mismatches, unauthorized token inflation, and asset lockups through improper state synchronization and missing input checks.  

**Original Text Preview:**

If a chain is compromised, it can not transfer beyond its balance. However, there's a clever way to increase its balance arbitrarily.

A compromised chain can deploy token manager to a token controlled by the attacker on a different chain (or in the future, deploy ITS with their minter address). The balance on that chain would be untracked and the attacker can bridge an unlimited amount to the compromised chain to increase its balance. This is very severe because the main purpose of the hub and audit is to reduce damage to the wider ecosystem when a chain is compromised.

### Proof of Concept

- There are 3 chains: Ethereum, BNB and WoofChain.
- `TokenA` is an ITS on Ethereum and WoofChain, but not on BNB.
- `TokenA` balance on Ethereum is Untracked cause it is the original.
- `TokenA` balance on WoofChain is 1,000.
- WoofChain is compromised.
- We can't bridge more than the 1,000 balance.

So the goal is to create an untracked balance that we can bridge from. We don't want Ethereum, cause we would have to buy and own the real tokens. So instead:

- Deploy FAKE Token on BNB. We can create `UINT256_MAX` totally owned by us.
- Deploy token manager from WoofChain to BNB with the token address set to the FAKE token.
- `TokenA` balance on BNB is now Untracked.
- Transfer a billion `TokenA` from BNB to WoofChain (FAKE token would be burnt/locked).
- `TokenA` balance on WoofChain is now `1,000,000,1000`.
- Transfer the billion `TokenA` to Ethereum.
- The hub confirms that it is less than the balance.
- Ethereum mints the bridged billion `TokenA`.

In fact, transferring to WoofChain to increase the tracked balance is unnecessary. We could have bridged from BNB to the original chain --Ethereum-- if there's an ITS connection between them. According to the "Publicly Known Issues" section of README, the hub would be updated to not track balance when a minter is specified. This would enable the bug in a different function because the attacker could specify their address as the minter.

Examples of a compromise that could happen include:

- Bad precompiles or revert handling: [Godwoken](https://medium.com/risk-dao/how-i-could-drain-an-entire-blockchain-post-mortem-on-a-bug-in-godwoken-chain-2451f83f72d2), [evmos](https://www.asymmetric.re/blog/evmos-precompile-state-commit-infinite-mint)
- Invalid transactions: [frontier](https://github.com/polkadot-evm/frontier/security/advisories/GHSA-hw4v-5x4h-c3xm)
- Consensus or malicious validator: [NEAR](https://hackenproof.com/blog/for-hackers/near-rewards-1-8-million-to-ethical-hackers-at-hackenproof#h2\_2)
- Compiler bugs: [vyper](https://medium.com/rektify-ai/the-vyper-compiler-saga-unraveling-the-reentrancy-bug-that-shook-defi-86ade6c54265)
- Event spoofing
- Contract compromise: e.g., admin key, take over, bug
- Amplifier/routing architecture compromise: External Gateway, Gateway for X on Axelar, Prover, Verifier, Relayer

### Recommended Mitigation Steps

Token hub should store the original chain of each ITS tokenId and allow token or manager deployment from the original chain only. This would limit the access of remote chains to transfers.

### Assessed type

Access Control

**[milapsheth (Axelar) confirmed and commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/76#issuecomment-2335033700):**
 > The report is valid, although we consider the severity to be Medium since the issue can't be exploited on it's own and requires a severe chain compromise to occur (since ITS by itself doesn't allow deploying token manager for trustless factory tokens). While ITS hub isn't meant to protect against all possible scenarios, it could handle this case by storing the original chain and restricting deployments from the origin chain as the report suggests. This is the same issue discussed in [#43](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/43) and [#77](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/77), although the exploit here is different.

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/76#issuecomment-2362279845):**
 > The Warden specifies a potential scenario in which a blockchain compromise could affect a balance greater than the original one that was bridged to it.
> 
> I do not believe that a blockchain compromise is meant to reduce the severity of the vulnerability as the ITS system is meant to integrate with as many chains as possible in a "permissionless" manner. As tokens can be directly affected by the described vulnerability, I believe a high-risk severity rating is appropriate. To note, the root cause is different from #77 and #43 and thus, merits its own submission.

**[milapsheth (Axelar) commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/76#issuecomment-2388259103):**
 > @0xsomeone - Our reply [here](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/77#issuecomment-2388178827) is relevant for this report, as well. ITS Hub is meant to limit damage in certain scenarios for chains connected to Axelar that have ITS deployed. But chains added by Axelar governance have to still meet a quality and security standard. Furthermore, ITS Hub explicitly whitelists ITS addresses by chain that it trusts. A compromised chain is inherently risky to all connected chains and apps, so ITS hub doesn't allow arbitrary permissionless connections. So without a concrete exploit that can steal user funds, we consider this report to be a Medium severity issue.

**[0xsomeone (judge) decreased severity to Medium and commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/76#issuecomment-2396605567):**
 > @milapsheth - After reviewing the codebase's documentation, I am inclined to agree that a blockchain compromise is considered a low-likelihood event rendering this submission to be of medium severity.

***

---
### Example 3

**Auto Label:** Insufficient validation of chain state and token parameters leads to balance mismatches, unauthorized token inflation, and asset lockups through improper state synchronization and missing input checks.  

**Original Text Preview:**

<https://github.com/code-423n4/2024-08-axelar-network/blob/4572617124bed39add9025317d2c326acfef29f1/interchain-token-service/contracts/InterchainTokenFactory.sol#L176>

<https://github.com/code-423n4/2024-08-axelar-network/blob/4572617124bed39add9025317d2c326acfef29f1/interchain-token-service/contracts/InterchainTokenService.sol#L342>

<https://github.com/code-423n4/2024-08-axelar-network/blob/4572617124bed39add9025317d2c326acfef29f1/interchain-token-service/contracts/InterchainTokenFactory.sol#L269>

### Impact

Token deployers can block the token bridge or limit the bridgeable amount without having Operator or FlowLimiter permissions. Especially, Canonical tokens can be attacked by anyone, not just the token register.

### Proof of Concept

ITSHub does not track the balance of the token on the original chain. It only tracks the balance of other chains which is deployed by remote deployment. This is because on the original chain, minters are usually registered to allow token minting, making balance tracking difficult.

```rust
fn apply_balance_tracking(
    storage: &mut dyn Storage,
    source_chain: ChainName,
    destination_chain: ChainName,
    message: &ItsMessage,
) -> Result<(), Error> {
    match message {
        ItsMessage::InterchainTransfer {
            token_id, amount, ..
        } => {
            // Update the balance on the source chain
@>          update_token_balance(
                storage,
                token_id.clone(),
                source_chain.clone(),
                *amount,
                false,
            )
@>          .change_context_lazy(|| Error::BalanceUpdateFailed(source_chain, token_id.clone()))?;

            // Update the balance on the destination chain
@>          update_token_balance(
                storage,
                token_id.clone(),
                destination_chain.clone(),
                *amount,
                true,
            )
@>          .change_context_lazy(|| {
                Error::BalanceUpdateFailed(destination_chain, token_id.clone())
            })?
        }
        // Start balance tracking for the token on the destination chain when a token deployment is seen
        // No invariants can be assumed on the source since the token might pre-exist on the source chain
        ItsMessage::DeployInterchainToken { token_id, .. } => {
@>          start_token_balance(storage, token_id.clone(), destination_chain.clone(), true)
                .change_context(Error::InvalidStoreAccess)?
        }
        ...
    };

    Ok(())
}
```

The balance of token on destination chain is initialized to 0 only when a new token is remotely deployed. And when tokens move, only the balance of the chain where the balance data has been initialized is updated. In normal cases, the balance of the original chain is not tracked.

```rust
pub fn start_token_balance(
    storage: &mut dyn Storage,
    token_id: TokenId,
    chain: ChainName,
    track_balance: bool,
) -> Result<(), Error> {
    let key = TokenChainPair { token_id, chain };

    match TOKEN_BALANCES.may_load(storage, key.clone())? {
        None => {
            let initial_balance = if track_balance {
@>              TokenBalance::Tracked(Uint256::zero())
            } else {
                TokenBalance::Untracked
            };

            TOKEN_BALANCES
                .save(storage, key, &initial_balance)?
                .then(Ok)
        }
        Some(_) => Err(Error::TokenAlreadyRegistered {
            token_id: key.token_id,
            chain: key.chain,
        }),
    }
}

pub fn update_token_balance(
    storage: &mut dyn Storage,
    token_id: TokenId,
    chain: ChainName,
    amount: Uint256,
    is_deposit: bool,
) -> Result<(), Error> {
    let key = TokenChainPair { token_id, chain };

    let token_balance = TOKEN_BALANCES.may_load(storage, key.clone())?;

    match token_balance {
        Some(TokenBalance::Tracked(balance)) => {
            let token_balance = if is_deposit {
                balance
@>                  .checked_add(amount)
                    .map_err(|_| Error::MissingConfig)?
            } else {
                balance
@>                  .checked_sub(amount)
                    .map_err(|_| Error::MissingConfig)?
            }
            .then(TokenBalance::Tracked);

            TOKEN_BALANCES.save(storage, key.clone(), &token_balance)?;
        }
@>      Some(_) | None => (),
    }

    Ok(())
}
```

However, there is a way to initialize the ITSHub balance of the original chain. When initializing the balance, it is set to 0, not the actual amount of tokens minted on the original chain. Therefore, if the balance of the original chain is initialized, the `ITSHub.update_token_balance` will fail due to underflow, making it impossible to send tokens from the original chain to other chains or limiting the amount that can be bridged.

You can initialize the ITSHub balance of the source chain because the `destinationChain` parameter in `InterchainTokenFactory.deployRemoteInterchainToken` does not be checked if it is the source chain of the token. The same issue exists when directly calling `InterchainTokenService.deployInterchainToken` to deploy tokens. This allows token deployment requests from remote chains to the original chain with the same `tokenId`.

While the token with the `tokenId` already exists on the original chain, it's possible to approve the GMP to the original chain, but the transaction will be failed when executed. However, the ITSHub balance initialization occurs during the process of transmitting the GMP to the destination chain, so it doesn't matter if the transaction fails on the original chain. Moreover, the initialized balance is not deleted even if the GMP execution fails.

The same issue occurs when deploying a canonical interchain token through `InterchainTokenFactory.deployRemoteCanonicalInterchainToken`, and it has the most critical severity. The `tokenId` of the canonical token is determined by the address of the original token, and the caller's address does not affect it. Therefore, anyone can call `InterchainTokenFactory.deployRemoteCanonicalInterchainToken` to deploy a specific canonical interchain token. This means anyone can prevent a specific canonical token from being bridged out from the original chain. Since a canonical token can only be registered once, this token can no longer be used normally with the Axelar bridge.

Let's look at an example of what happens when you initialize the ITSHub balance of the original chain. Suppose there are chains A, B, and C, where A is the original chain, and B and C are related chains. Here are the ITSHub balances:

- A: None
- B: 100
- C: 50

Now, using the vulnerability, the balance on chain A has been initialized.

- A: 0
- B: 100
- C: 50

Until chain B or C send some tokens to chain A, no more tokens can be go out from chain A. Also, only a maximum of 150 tokens can move between bridges.

This is a PoC. You can run it by adding it to `interchain-token-service/test/InterchainTokenService.js`. It shows that chain A is the original chain, and after remotely deploying a token from A to B, it's possible to request remote deployment from B to A. On chain A, a token with the same `tokenId` is already deployed, so the GMP execution on the chain A will be failed, but the balance of chain A in ITSHub will be initialized to 0.

```
describe('PoC remote deploy to original chain', () => {
    const tokenName = 'Token Name';
    const tokenSymbol = 'TN';
    const tokenDecimals = 13;
    let sourceAddress;

    before(async () => {
        sourceAddress = service.address;
    });

    it('Can request to remote deploy for original chain', async () => {

        const salt = getRandomBytes32();
        const tokenId = await service.interchainTokenId(wallet.address, salt);
        const tokenManagerAddress = await service.tokenManagerAddress(tokenId);
        const minter = '0x';
        const operator = '0x';
        const tokenAddress = await service.interchainTokenAddress(tokenId);
        const params = defaultAbiCoder.encode(['bytes', 'address'], [operator, tokenAddress]);
        let payload = defaultAbiCoder.encode(
            ['uint256', 'bytes32', 'string', 'string', 'uint8', 'bytes', 'bytes'],
            [MESSAGE_TYPE_DEPLOY_INTERCHAIN_TOKEN, tokenId, tokenName, tokenSymbol, tokenDecimals, minter, operator],
        );
        const commandId = await approveContractCall(gateway, sourceChain, sourceAddress, service.address, payload);

        // 1. Receive remote deployment messages and deploy interchain tokens on the destination chain

        await expect(service.execute(commandId, sourceChain, sourceAddress, payload))
            .to.emit(service, 'InterchainTokenDeployed')
            .withArgs(tokenId, tokenAddress, AddressZero, tokenName, tokenSymbol, tokenDecimals)
            .and.to.emit(service, 'TokenManagerDeployed')
            .withArgs(tokenId, tokenManagerAddress, NATIVE_INTERCHAIN_TOKEN, params);
        const tokenManager = await getContractAt('TokenManager', tokenManagerAddress, wallet);
        expect(await tokenManager.tokenAddress()).to.equal(tokenAddress);
        expect(await tokenManager.hasRole(service.address, OPERATOR_ROLE)).to.be.true;

        // 2. Can request a remote deployment to the source(original) chain

        const destChain = sourceChain;

        payload = defaultAbiCoder.encode(
            ['uint256', 'bytes32', 'string', 'string', 'uint8', 'bytes'],
            [MESSAGE_TYPE_DEPLOY_INTERCHAIN_TOKEN, tokenId, tokenName, tokenSymbol, tokenDecimals, minter],
        );

        await expect(
            reportGas(
                service.deployInterchainToken(salt, destChain, tokenName, tokenSymbol, tokenDecimals, minter, gasValue, {
                    value: gasValue,
                }),
                'Send deployInterchainToken to remote chain',
            ),
        )
            .to.emit(service, 'InterchainTokenDeploymentStarted')
            .withArgs(tokenId, tokenName, tokenSymbol, tokenDecimals, minter, destChain)
            .and.to.emit(gasService, 'NativeGasPaidForContractCall')
            .withArgs(service.address, destChain, service.address, keccak256(payload), gasValue, wallet.address)
            .and.to.emit(gateway, 'ContractCall')
            .withArgs(service.address, destChain, service.address, keccak256(payload), payload);

        // 3. ITSHub will initialize the balance of the original chain
    });
});
```

### Recommended Mitigation Steps

1. In `InterchainTokenFactory.deployRemoteInterchainToken`, check if `destinationChain` is the same as `originalChainName` to prevent remote deployment requests to the original chain.

    ```diff
    function deployRemoteInterchainToken(
        string calldata originalChainName,
        bytes32 salt,
        address minter,
        string memory destinationChain,
        uint256 gasValue
    ) external payable returns (bytes32 tokenId) {
        string memory tokenName;
        string memory tokenSymbol;
        uint8 tokenDecimals;
        bytes memory minter_ = new bytes(0);

        {
            bytes32 chainNameHash_;
            if (bytes(originalChainName).length == 0) {
                chainNameHash_ = chainNameHash;
            } else {
                chainNameHash_ = keccak256(bytes(originalChainName));
            }
    +       require(chainNameHash_ != keccak256(bytes(destinationChain)), "Cannot remote deploy on original chain");

            address sender = msg.sender;
            salt = interchainTokenSalt(chainNameHash_, sender, salt);
            tokenId = interchainTokenService.interchainTokenId(TOKEN_FACTORY_DEPLOYER, salt);

            IInterchainToken token = IInterchainToken(interchainTokenService.interchainTokenAddress(tokenId));

            tokenName = token.name();
            tokenSymbol = token.symbol();
            tokenDecimals = token.decimals();

            if (minter != address(0)) {
                if (!token.isMinter(minter)) revert NotMinter(minter);

                minter_ = minter.toBytes();
            }
        }

        tokenId = _deployInterchainToken(salt, destinationChain, tokenName, tokenSymbol, tokenDecimals, minter_, gasValue);
    }
    ```

2. Check `originalChainName` and `destinationChain` in `InterchainTokenFactory.deployRemoteCanonicalInterchainToken` to ensure that a remote deploy cannot be requested to the original chain.

3. When deploying by `InterchainTokenService.deployInterchainToken` directly, it's not possible to check if the original chain and destination chain are the same. Store information about the original chain for each `tokenId` in ITSHub, and ensure that the balance of the original chain is not initialized.

### Assessed type

Invalid Validation

**[milapsheth (Axelar) confirmed and commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/8#issuecomment-2334767884):**
 > The report is valid. We consider this a Medium severity issue, however, since assets can't be stolen. It's a DOS issue that prevents canonical tokens from being transferred from the origin chain to other chains. ITS hub can be upgraded easily to handle the scenario where `source_chain == destination_chain` and fix the balance for the source chain, so the impact is low.

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/8#issuecomment-2362299816):**
 > The Warden has identified a mechanism via which bridging of canonical tokens can be permanently DoSd.
> 
> Upgrades of contract logic are not considered appropriate mitigations for vulnerabilities such as the one described (as we could effectively resolve any and all vulnerabilities identified via upgrades), so I believe that a high-risk severity rating is appropriate for this submission.

**[milapsheth (Axelar) commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/8#issuecomment-2388292342):**
 > @0xsomeone - In the case of a bug that allows stealing user funds, an upgrade could be too late if the issue is already exploited. Whereas if the DoS issue reported here is triggered, an upgrade can fix the issue without loss of funds. Furthermore, ITS Hub is designed to be upgradable (a non upgradable contract would have made this issue more severe). For this reason, we consider this to be a Medium severity issue since the impact is much lower than a compromise of user funds.

**[0xsomeone (judge) commented](https://github.com/code-423n4/2024-08-axelar-network-findings/issues/8#issuecomment-2396605748):**
 > @milapsheth - While from a practical perspective you might consider this to be a medium-severity issue, C4 guidelines are clear that contract upgrades cannot be utilized to mitigate the severity of a vulnerability (i.e. we consider this DoS irrecoverable). As such, this vulnerability's severity will remain from a C4 audit perspective.

***

---
