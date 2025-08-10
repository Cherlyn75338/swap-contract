# Unsafe Unwraps in Reply Parsing and Queries Lead to Panics

## Project: contracts/swap/src/swap.rs, contracts/swap/src/queries.rs, contracts/swap/src/helpers.rs

## Severity: Medium (Transaction-level DoS)

## Category: Error Handling

---

## 🔍 Description
The contract uses `unwrap()`/`expect()` at several points while parsing submessage replies and querying markets. In CosmWasm, a panic aborts execution and bubbles up as a failed transaction. Attackers cannot inject malformed reply protobufs easily, but chain-level anomalies or module changes can cause unexpected shapes. Similarly, empty `msg_responses` or missing market entries can occur under edge conditions, leading to panics.

## 📜 Affected Code
```rust
pub fn parse_market_order_response(msg: Reply) -> StdResult<MsgCreateSpotMarketOrderResponse> {
    let binding = msg.result.into_result().map_err(ContractError::SubMsgFailure).unwrap();

    let first_message = binding.msg_responses.first();
    let order_response = MsgCreateSpotMarketOrderResponse::decode(first_message.unwrap().value.as_slice())
        .map_err(|err| ContractError::ReplyParseFailure { id: msg.id, err: err.to_string(), })
        .unwrap();

    Ok(order_response)
}
```

```rust
// first market expect
let first_market = querier.query_spot_market(&first_market_id)?.market.expect("market should be available");
```

```rust
// next market expect
let next_market = querier.query_spot_market(&next_market_id)?.market.expect("market should be available");
```

```rust
// helpers: unwrap on response indexing
match &response.get(position).unwrap().msg { /* ... */ }
```

## 🧠 Root Cause
- Use of `unwrap()` and `expect()` on external data (reply messages, querier results) without graceful handling.

## ⚠️ Exploitability
- Is this vulnerability exploitable? Yes (Denial-of-Service)
- How: While direct crafting of malformed replies is constrained by the chain, abnormal module responses, empty `msg_responses`, or market being delisted can trigger these unwraps to panic and fail the transaction predictably.

## 💥 Impact
- Smart Contracts: High for availability if frequent; practically Medium for isolated tx DoS
  - Transaction failures; path unavailability when market data or replies do not meet expectations.

## ✅ Remediation Recommendations
- Replace `unwrap()`/`expect()` with error returns. Example:
  - Safely handle `msg.result.into_result()` and return `ContractError::SubMsgFailure`.
  - Validate `msg_responses.first()` and return a descriptive error if absent.
  - When `market` is `None`, return a clear error instead of panicking.
- Add tests for empty `msg_responses`, decode failures, and missing market responses.

## 🔁 Related Issues
- Over-refund path also uses `expect` for market presence, compounding risks.

## 🧪 Test Cases
- Simulate reply with empty `msg_responses` and assert `ReplyParseFailure` not panic.
- Simulate missing market in querier; assert handled error.
- Simulate decode error; assert graceful error.
