# Cluster -1241

**Rank:** #336  
**Count:** 20  

## Label
Missing input validation leads to invalid state transitions, irreversible failures, and exploitable behavior due to unbounded or malformed parameters.

## Cluster Information
- **Total Findings:** 20

## Examples

### Example 1

**Auto Label:** Panic-induced denial-of-service from unhandled errors in input validation and result propagation.  

**Original Text Preview:**

## Context
- `rpc.rs#L133-L134`, `rpc.rs#L185`

## Description
In several cases, an address is parsed using `Address::from_str()` and immediately unwrapped:
- `rpc.rs#L133`
- `rpc.rs#L182`
- `rpc.rs#L241`
- `rpc.rs#L242`
- `rpc.rs#L185`

This can cause a panic if the string is not a valid address.

## Proof of Concept
```rust
Address::from_str("").unwrap();
// Panic:
// called `Result::unwrap()` on an `Err` value: Invalid input length

Address::from_str("xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx").unwrap();
// Panic:
// called `Result::unwrap()` on an `Err` value: Invalid character 'x' at position 0
```

## Recommendation
It is better to return an error instead of panicking.

## Patch
```diff
diff --git a/src/rpc.rs b/src/rpc.rs
index 730791d..0fc0358 100644
--- a/src/rpc.rs
+++ b/src/rpc.rs
@@ -130,7 +130,7 @@ pub async fn eth_withdraw(
 if !a["amount"].is_u64() {
     return Ok(invalid_input_error("amount is not a rational number"));
 }
- let address = Address::from_str(a["address"].as_str().unwrap()).unwrap();
+ let address = Address::from_str(a["address"].as_str().unwrap())?;
 let amount = parse_ether(a["amount"].as_u64().unwrap()).unwrap();
 info!(
 "Sending notification {:?} / {:?} to Worker",
@@ -179,10 +179,10 @@ pub async fn token_withdraw(
 if !a["amount"].is_u64() {
     return Ok(invalid_input_error("amount is not a rational number"));
 }
- let receiver = Address::from_str(a["address"].as_str().unwrap()).unwrap();
+ let receiver = Address::from_str(a["address"].as_str().unwrap())?;
// Native amount
 let token_amount = a["amount"].as_u64().unwrap();
- let token_address = Address::from_str(a["token_address"].as_str().unwrap()).unwrap();
+ let token_address = Address::from_str(a["token_address"].as_str().unwrap())?;
 info!(
 "Sending notification {:?} / {:?} of {:?} to Worker",
 receiver, token_amount, token_address
@@ -238,8 +238,8 @@ pub async fn set_evm_forwarding(
 if !a["amount"].is_u64() {
     return Ok(invalid_input_error("amount is not a rational number"));
 }
- let receiver = Address::from_str(a["receiver"].as_str().unwrap()).unwrap();
- let token_address = Address::from_str(a["token"].as_str().unwrap()).unwrap();
+ let receiver = Address::from_str(a["receiver"].as_str().unwrap())?;
+ let token_address = Address::from_str(a["token"].as_str().unwrap())?;
 let hash_str = a["hashlock"].as_str().unwrap();
 let hashlock = match hex::decode(hash_str) {
```

**SatsBridge:** Fixed in PR 1.  
**Cantina Managed:** Fixed.

---
### Example 2

**Auto Label:** Panic-induced denial-of-service from unhandled errors in input validation and result propagation.  

**Original Text Preview:**

## Context: auth.rs#L116

## Description
The `/action` HTTP endpoint is used by users to submit an action to be executed through a POST request. The payload from the POST request is decoded as an `ActionDecoded`. However, the `verify` function will panic when the payload does not correspond to a valid action. This will exit the nord process. Any user can crash the nord server through a single POST request.

## Proof of Concept
Build and start the nord server:

1. Build in release mode:
   ```bash
   cargo build --release
   ```

2. Start the nord server, which should listen on :3000:
   ```bash
   ./target/release/nord start
   ```

Execute a POST request with an incorrect payload:
```bash
curl -d ' This is not going to be decoded ' -X POST http://127.0.0.1:3000/action
```

This results in the following output for the nord server:
```
thread '<unnamed>' panicked at nord/src/auth.rs:116:57:
called `Result::unwrap()` on an `Err` value: DecodeFailure
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
[1] 639194 IOT instruction ./target/release/nord start
```

## Recommendation
The `Result` from `ActionDecoded::encode(payload)` must be handled in `verify` instead of panicking. The following code can be used to propagate the error:

```rust
let decoded_action = ActionDecoded::decode(payload)?;
```

## LayerN
Fixed in PR 762 (commit 84e63cc3).

## Cantina Managed
Fixed. The result is correctly handled. Any error is mapped into an `engine::Error::DecodeFailure` error.

---
### Example 3

**Auto Label:** Panic-induced denial-of-service from unhandled errors in input validation and result propagation.  

**Original Text Preview:**

Source: https://github.com/sherlock-audit/2024-06-allora-judging/issues/107 

## Found by 
LZ\_security
## Summary

## Vulnerability Detail
topics_handler will make an http request to call the http api in blockless,

```go
func makeApiCall(payload string) error {
	url := os.Getenv("BLOCKLESS_API_URL")
	method := "POST"

	client := &http.Client{}
	req, err := http.NewRequest(method, url, strings.NewReader(payload))
	if err != nil {
		return err
	}
	req.Header.Add("Accept", "application/json, text/plain, */*")
	req.Header.Add("Content-Type", "application/json;charset=UTF-8")

	res, err := client.Do(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()

	return nil
}
```

It is often dangerous to initiate an http request in a node on a blockchain chain.
However, because the operation here is performed in a PrepareProposalHandler, it may not result in direct state inconsistency between different nodes.

But the problem here is that the malicious node can not perform this operation, so blockless can not receive http requests.

Because other nodes have no way of knowing whether the http request can be successfully executed, the http request may fail due to network problems.

Therefore, the malicious node can choose not to execute the http request, so that he can save server-side resources, or in the purpose of attack.

## Impact
The api in blockless cannot be invoked because the malicious node does not execute the http request,causing the protocol to fail to work or affecting blockless working efficiency.

## Code Snippet
https://github.com/sherlock-audit/2024-06-allora/blob/main/allora-chain/app/api.go#L166-L185

## Tool used

Manual Review

## Recommendation
Let blockless query data from the chain instead of the node on the chain calling blockless.



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**0xmystery** commented:
> Malicious node could cause http request to not execute



**sherlock-admin2**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/allora-network/allora-chain/pull/458

---
