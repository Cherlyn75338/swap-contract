# `unwrap()` Panics in `parse_market_order_response`

## Project: contracts/swap/src/swap.rs

## Severity: Medium

## Category: Robustness / Denial-of-Service

---

## 🔍 Description
The helper `parse_market_order_response` performs two unchecked `unwrap()` operations that can panic the whole contract instance:

```rust
260:265:/workspace/contracts/swap/src/swap.rs
    let binding = msg.result.into_result().map_err(ContractError::SubMsgFailure).unwrap();

    let first_message = binding.msg_responses.first();
    let order_response = MsgCreateSpotMarketOrderResponse::decode(first_message.unwrap().value.as_slice())
        .map_err(|err| ContractError::ReplyParseFailure { id: msg.id, err: err.to_string() })
        .unwrap();
```

Both unwraps assume that:
1. `msg.result` is `Ok`, i.e. the sub-message succeeded.
2. The protobuf‐encoded `MsgCreateSpotMarketOrderResponse` is present at index 0 of the reply and decodes successfully.

Any deviation—failed order, empty response vector, decoding error—will cause a **runtime panic**, triggering `Wasmer`’s “contract trapped” error and *aborting* the entire transaction.

## 🧠 Root Cause
Unsafe use of `unwrap()` inside library code that should propagate errors via `StdResult`/`ContractError`.

## ⚠️ Exploitability
**Exploitable:** YES (Denial-of-Service)

Although no funds are directly stolen, *anyone* can intentionally craft a swap that triggers an execution failure inside the exchange module (e.g., zero-liquidity order). The subsequent reply will contain an error, causing `unwrap()` to panic and prevent state cleanup, potentially bricking the contract until manual intervention.

## 💥 Impact
Temporary or permanent freezing of the swap contract, blocking all users from executing swaps (High impact for DEX front-ends relying on the contract).

## ✅ Remediation Recommendations
1. Replace `unwrap()` with proper `?` propagation:
```rust
let binding = msg.result.into_result().map_err(ContractError::SubMsgFailure)?;
let first = binding.msg_responses.first().ok_or(ContractError::CustomError { val: "empty reply".into() })?;
let order_response = MsgCreateSpotMarketOrderResponse::decode(first.value.as_slice())
    .map_err(|err| ContractError::ReplyParseFailure { id: msg.id, err: err.to_string() })?;
```
2. Add unit tests for error cases (failed order, empty reply).

## 🔁 Related Issues
• V02 – Hijacked state may combine with panic to lock funds.

## 🧪 Test Cases
1. Simulate a sub-message failure (`msg.result` = `Err`) and assert that the contract returns `ContractError::SubMsgFailure` not a panic.
2. Simulate empty `msg_responses` vector.