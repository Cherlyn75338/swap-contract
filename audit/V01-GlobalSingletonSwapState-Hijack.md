# Global Singleton Swap State Hijack via Overwrite

## 📌 Project / File / Module
- Injective Swap Contract
- Files: `contracts/swap/src/state.rs`, `contracts/swap/src/swap.rs`, `contracts/swap/src/contract.rs`

## 🧭 Severity
- None (Not exploitable under current design)
- Based on Smart Contracts classification

## 📚 Category
- State Management, Reentrancy Analysis

---

## 🔍 Full Technical Description
The contract uses singleton storage items to thread ephemeral state across submessage replies within a single transaction:

```6:10:/workspace/contracts/swap/src/state.rs
use cw_storage_plus::{Bound, Item, Map};

pub const SWAP_ROUTES: Map<(String, String), SwapRoute> = Map::new("swap_routes");
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
pub const STEP_STATE: Item<CurrentSwapStep> = Item::new("current_step_cache");
pub const SWAP_RESULTS: Item<Vec<SwapResults>> = Item::new("swap_results");
```

The swap starts by saving the operation and producing an Injective atomic order submessage whose reply is handled synchronously:

```20:31:/workspace/contracts/swap/src/swap.rs
pub fn start_swap_flow(
    deps: DepsMut<InjectiveQueryWrapper>,
    env: Env,
    info: MessageInfo,
    target_denom: String,
    swap_quantity_mode: SwapQuantityMode,
) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    if info.funds.len() != 1 {
        return Err(ContractError::CustomError {
            val: "Only one denom can be passed in funds".to_string(),
        });
    }
```

```91:103:/workspace/contracts/swap/src/swap.rs
let swap_operation = CurrentSwapOperation { /* … */ };

SWAP_RESULTS.save(deps.storage, &Vec::new())?;
SWAP_OPERATION_STATE.save(deps.storage, &swap_operation)?;

execute_swap_step(deps, env, swap_operation, 0, current_balance).map_err(ContractError::Std)
```

```144:156:/workspace/contracts/swap/src/swap.rs
let order_message = SubMsg::reply_on_success(
    create_spot_market_order_msg(contract.to_owned(), order),
    ATOMIC_ORDER_REPLY_ID,
);

let current_step = CurrentSwapStep { /* … */ };
STEP_STATE.save(deps.storage, &current_step)?;

let response = Response::new().add_submessage(order_message);
Ok(response)
```

Replies are dispatched only to the atomic order handler:

```68:74:/workspace/contracts/swap/src/contract.rs
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn reply(deps: DepsMut<InjectiveQueryWrapper>, env: Env, msg: Reply) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    match msg.id {
        ATOMIC_ORDER_REPLY_ID => handle_atomic_order_reply(deps, env, msg),
        _ => Err(ContractError::UnrecognizedReply(msg.id)),
    }
}
```

Within the reply, the contract loads the previously stored singletons, computes the next step, and either enqueues another submessage (continuing within the same transaction) or finalizes and removes all state:

```175:186:/workspace/contracts/swap/src/swap.rs
let mut swap_results = SWAP_RESULTS.load(deps.storage)?;

let current_step = STEP_STATE.load(deps.storage).map_err(ContractError::Std)?;

let swap = SWAP_OPERATION_STATE.load(deps.storage)?;

let has_next_market = swap.swap_steps.len() > (current_step.step_idx + 1) as usize;
```

```213:231:/workspace/contracts/swap/src/swap.rs
if current_step.step_idx < (swap.swap_steps.len() - 1) as u16 {
    SWAP_RESULTS.save(deps.storage, &swap_results)?;
    return execute_swap_step(deps, env, swap, current_step.step_idx + 1, new_balance).map_err(ContractError::Std);
}

// last step, finalize and send back funds to a caller
let send_message = BankMsg::Send {
    to_address: swap.sender_address.to_string(),
    amount: vec![new_balance.clone().into()],
};
```

```243:247:/workspace/contracts/swap/src/swap.rs
SWAP_OPERATION_STATE.remove(deps.storage);
STEP_STATE.remove(deps.storage);
SWAP_RESULTS.remove(deps.storage);

let mut response = Response::new().add_message(send_message).add_event(swap_event);
```

CosmWasm’s execution model guarantees that `SubMsg::reply_on_success` callbacks execute synchronously within the same transaction that enqueued the submessage. No external transaction can interleave between a submessage and its `reply` handler. Therefore, although singleton state appears globally shared, it is effectively scoped to the single in-flight transaction and cannot be overwritten by another user between steps.

---

## 🛠️ Root Cause
No vulnerability in current architecture. The singletons are used as an internal transaction-scoped scratchpad across synchronous submessage replies. There is no cross-transaction or cross-user interleaving between `reply_on_success` and `reply`.

---

## 💥 Exploitability
- Is it exploitable: ❌ No
- Proof path: An attacker cannot inject execution between the victim’s `SubMsg::reply_on_success` and `handle_atomic_order_reply` due to CosmWasm’s synchronous, atomic transaction model. Any front-run or back-run is an entirely separate transaction that executes strictly before or after the victim’s transaction, not during it.
- Prerequisites: N/A

Conditions under which risk would emerge:
- If the swap were redesigned to rely on asynchronous, cross-transaction callbacks (e.g., IBC acknowledgements, asynchronous module results) or cross-contract callbacks scheduled in later transactions, singleton state would become unsafe and require keying by swap-id or sender.

---

## 🎯 Exploit Scenario
Not achievable under current design. Even with MEV reordering, another transaction cannot overwrite singletons during the in-flight reply chain of the victim’s transaction.

---

## 📉 Financial/System Impact
- None under the present synchronous `reply_on_success` pipeline with Injective atomic orders. No theft path exists.

---

## 🧰 Mitigations Present
- Synchronous `SubMsg::reply_on_success` with immediate module execution and reply.
- No external contract calls that could reenter this contract mid-flow.
- State cleanup at the end of the flow to avoid stale residues.

---

## 🧬 Remediation Recommendations
Non-blocking hardening (future-proofing if design evolves):
- Key state by `(sender, nonce)` or by a generated `swap_id` and encode the id into `reply.id` or maintain a `reply_id -> swap_id` map.
- Add a guard to prevent initiating a new swap if a user’s prior swap is in-flight, if parallel user swaps become supported.
- Defensive checks in the reply to confirm state consistency (e.g., expected step index, route length).

---

## 🧪 Suggested Tests
- Multi-user sequential swaps in a single block to confirm isolation and correct state cleanup.
- Regression test that attempts to simulate interleaving calls across different transactions and asserts no state bleed between them.
- Round-trip multi-hop route test verifying state is present during the reply chain and removed upon completion.

---

## 🔄 Related Issues
- See separate report on unsafe unwraps in `parse_market_order_response` that can cause abort-on-panic (reliability), though not exploitable for theft.