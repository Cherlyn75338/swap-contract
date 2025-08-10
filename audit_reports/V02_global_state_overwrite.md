# Global `SWAP_OPERATION_STATE` Overwrite Leads to Cross-User Hijacking

## Project: contracts/swap/src/state.rs & contracts/swap/src/swap.rs

## Severity: Critical

## Category: Access Control / Reentrancy

---

## 🔍 Description
`SWAP_OPERATION_STATE` is declared as a **singleton** storage item:

```rust
6:9:/workspace/contracts/swap/src/state.rs
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
```

Every time a new swap is initiated the singleton is overwritten **without** any check that a previous swap is still in flight:

```rust
99:103:/workspace/contracts/swap/src/swap.rs
    SWAP_RESULTS.save(deps.storage, &Vec::new())?;
    SWAP_OPERATION_STATE.save(deps.storage, &swap_operation)?;
```

Because swap execution is *asynchronous*—steps are carried out via sub-message replies—any user can race-call `start_swap_flow` and replace the global state **between two steps** of another user’s multi-hop swap. When the pending `handle_atomic_order_reply` fires, it will load the *attacker-controlled* state and continue execution, ultimately sending the victim’s funds to the attacker.

## 🧠 Root Cause
Lack of per-swap isolation. The design assumes only one swap can exist at a time but does not enforce this at the message level. Cosm-Wasm contracts are **re-entrant between messages**.

## ⚠️ Exploitability
**Exploitable:** YES

Attack scenario:
1. Victim starts a multi-hop swap (e.g. 3 markets). After step-0 the contract stores the state and waits for the `Reply`.
2. Attacker immediately calls `start_swap_flow`, supplying their own `sender_address` and route. `SWAP_OPERATION_STATE` is overwritten.
3. When the reply from the victim’s step-0 arrives, `handle_atomic_order_reply` loads the attacker’s state and proceeds, transferring the *victim’s* acquired tokens to `attacker.sender_address`.

No special permissions are required—only transaction ordering.

## 💥 Impact
• Direct theft of user funds across any asset supported by the swap contract.  
• Full loss is possible because the entire multi-hop amount can be redirected.  
Aligned with **Critical** impact class for smart contracts.

## ✅ Remediation Recommendations
1. **Per-User / Per-Swap Keys:** Index swap operation by `(sender, nonce)` or a deterministic UUID rather than using a global singleton.
2. **Re-entrancy Guard:** Reject `start_swap_flow` if another swap is still pending (simple lock) *or* maintain a queue of swaps.
3. Review all `STEP_STATE` and `SWAP_RESULTS` stores for the same assumption.

## 🔁 Related Issues
• V03 – Panic in reply handler will also unlock the re-entrancy window.  
• Architectural – Consider redesigning to make each swap stateless and atomic.

## 🧪 Test Cases
1. Integration test with two actors executing overlapping swaps; assert that funds can be hijacked before patch and are isolated afterwards.
2. Fuzz test varying hop counts and block ordering via mocked `App` scheduler.