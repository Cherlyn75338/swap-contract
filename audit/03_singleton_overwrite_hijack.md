# Alleged Global Singleton Overwrite Enabling Swap Hijack

## Project: contracts/swap/src/state.rs, contracts/swap/src/swap.rs

## Severity: None (Not exploitable) – Misinterpretation of CosmWasm execution model

## Category: State Management, Reentrancy Analysis

---

## 🔍 Description
A concern was raised that the `SWAP_OPERATION_STATE` singleton allows cross-user overwrite, enabling an attacker to hijack another user's multi-step swap by overwriting the state between submessage replies. However, in CosmWasm, submessage replies are executed within the same transaction, synchronously and atomically, and there is no external reentrancy into the contract between `SubMsg::reply_on_success` and its `reply` handler.

## 📜 Affected Code
```rust
// contracts/swap/src/state.rs
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
```

```startLine:endLine:/workspace/contracts/swap/src/state.rs
6:10:/workspace/contracts/swap/src/state.rs
```

```rust
// contracts/swap/src/swap.rs
SWAP_OPERATION_STATE.save(deps.storage, &swap_operation)?;
// ... later, within reply handler
let swap = SWAP_OPERATION_STATE.load(deps.storage)?;
```

```startLine:endLine:/workspace/contracts/swap/src/swap.rs
98:105:/workspace/contracts/swap/src/swap.rs
```

```startLine:endLine:/workspace/contracts/swap/src/swap.rs
181:186:/workspace/contracts/swap/src/swap.rs
```

## 🧠 Root Cause
- Singleton usage appears dangerous in general multi-user contexts, but in this pattern it is only used for the current in-flight swap within the same transaction. There is no yielded control to another external transaction between steps.

## ⚠️ Exploitability
- Is this vulnerability exploitable? No
- Why: CosmWasm contracts execute to completion per transaction. Submessage reply callbacks are part of the same atomic execution context. Another user cannot interleave execution to overwrite the singleton before `handle_atomic_order_reply` completes. There is no async persistence gap exploitable by a different sender.
- Caveat: If the design evolves to use async, cross-tx callbacks or IBC, user-scoped state keys would be required.

## 💥 Impact
- None under current design. No cross-user hijack is achievable in practice.

## ✅ Remediation Recommendations
- Optional: Make state keys user-scoped (e.g., by sender address) to future-proof against changes and to support parallel in-flight swaps if needed.
- Document that multi-step swaps are single-tx atomic and not concurrent across users.

## 🔁 Related Issues
- Error handling and refund bugs are unrelated but occur in nearby code.

## 🧪 Test Cases
- Attempt to run two swaps in the same block/tx context is not possible from two senders; simulate sequential swaps to confirm state is removed at the end and cannot be read by another tx.
