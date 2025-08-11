### Injective Swap Contract – Exploitability Verification Report

- Project: `contracts/swap`
- Commit scope: current workspace snapshot
- Objective: Verify, with certainty, whether the reported vulnerabilities are exploitable in this codebase

---

## Executive Verdict

- **Is this a real vulnerability, or intended behavior?**
  - Using singleton items (`Item<T>`) exists, but in this codebase it is used only within a single, atomic execute flow that completes synchronously via `SubMsg::reply_on_success` to the Injective exchange module. **Given current code, this is intended behavior and not a vulnerability.**

- **Is it 100% exploitable on-chain?**
  - **No.** The alleged exploit paths require IBC/sudo/async callbacks or WasmMsg-based cross-contract reentrancy, none of which exist here. The swap process is implemented as a single transaction flow; the submessage reply is synchronous and atomic.

- **External actor prerequisites?**
  - Not applicable, as the exploit paths do not exist in this codebase.

- **Financial impact (theoretical and real-world)?**
  - Under current implementation, **none**. Funds are sent to the original caller captured and persisted within the same transaction and cleared on completion.

- **Conditions under which exploit works/fails?**
  - Would only become viable if the contract later introduced async flows (IBC `ack`, `sudo` fills) or untrusted Wasm contract calls before state cleanup while still relying on global singletons without correlation/ownership checks.

- **Existing mitigations and effectiveness?**
  - The design inherently mitigates the reported class via a fully atomic, single-transaction flow using `reply_on_success`. No IBC, no `sudo`, no external Wasm calls.

- **Achievable under actual protocol conditions vs contrived edge cases?**
  - The tested “overwrites” in isolated storage mocks do not model transaction atomicity. **Exploit is not achievable** in the actual protocol conditions given this repo’s implementation.

---

## What Actually Exists in This Repository

- Storage singletons are present:
```startLine:63:/workspace/contracts/swap/src/state.rs
use cw_storage_plus::{Bound, Item, Map};

pub const SWAP_ROUTES: Map<(String, String), SwapRoute> = Map::new("swap_routes");
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
pub const STEP_STATE: Item<CurrentSwapStep> = Item::new("current_step_cache");
pub const SWAP_RESULTS: Item<Vec<SwapResults>> = Item::new("swap_results");
pub const CONFIG: Item<Config> = Item::new("config");
```

- Swap flow saves singleton state, then immediately issues a submessage with `reply_on_success` and processes the reply in the same transaction. Cleanup happens at the end of the flow:
```startLine:91:endLine:105:/workspace/contracts/swap/src/swap.rs
let swap_operation = CurrentSwapOperation {
    sender_address,
    swap_steps: steps,
    swap_quantity_mode,
    refund: Coin::new(refund_amount, source_denom.to_owned()),
    input_funds: coin_provided.to_owned(),
};

SWAP_RESULTS.save(deps.storage, &Vec::new())?;
SWAP_OPERATION_STATE.save(deps.storage, &swap_operation)?;

execute_swap_step(deps, env, swap_operation, 0, current_balance).map_err(ContractError::Std)
```

```startLine:144:endLine:156:/workspace/contracts/swap/src/swap.rs
let order_message = SubMsg::reply_on_success(
    create_spot_market_order_msg(contract.to_owned(), order),
    ATOMIC_ORDER_REPLY_ID,
);

let current_step = CurrentSwapStep { ... };
STEP_STATE.save(deps.storage, &current_step)?;

let response = Response::new().add_submessage(order_message);
Ok(response)
```

```startLine:158:endLine:173:/workspace/contracts/swap/src/swap.rs
pub fn handle_atomic_order_reply(..., msg: Reply) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    let order_response = parse_market_order_response(msg)?;
    // ... compute results ...
    let mut swap_results = SWAP_RESULTS.load(deps.storage)?;
    let current_step = STEP_STATE.load(deps.storage).map_err(ContractError::Std)?;
    let swap = SWAP_OPERATION_STATE.load(deps.storage)?;
    // ... continue flow ...
```

```startLine:227:endLine:255:/workspace/contracts/swap/src/swap.rs
// last step, finalize and send back funds to a caller
let send_message = BankMsg::Send {
    to_address: swap.sender_address.to_string(),
    amount: vec![new_balance.clone().into()],
};
// ...
SWAP_OPERATION_STATE.remove(deps.storage);
STEP_STATE.remove(deps.storage);
SWAP_RESULTS.remove(deps.storage);

let mut response = Response::new().add_message(send_message).add_event(swap_event);
if !swap.refund.amount.is_zero() {
    let refund_message = BankMsg::Send { to_address: swap.sender_address.to_string(), amount: vec![swap.refund] };
    response = response.add_message(refund_message)
}
Ok(response)
```

- Entry points include `reply`, wired to the same reply ID. No `sudo`, no IBC handlers:
```startLine:68:endLine:74:/workspace/contracts/swap/src/contract.rs
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn reply(deps: DepsMut<InjectiveQueryWrapper>, env: Env, msg: Reply) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    match msg.id {
        ATOMIC_ORDER_REPLY_ID => handle_atomic_order_reply(deps, env, msg),
        _ => Err(ContractError::UnrecognizedReply(msg.id)),
    }
}
```

- There is no `ibc_packet_ack`, `ibc_*` files, or `sudo` entry point anywhere in the repo.

---

## Exhaustive Check Against Alleged Exploit Paths

- Allegation: “IBC async callback state manipulation.”
  - Repo reality: No IBC module, no `ibc_packet_ack` or any IBC entrypoints. Therefore, **path does not exist**.

- Allegation: “Sudo callback state hijacking (Injective exchange fills).”
  - Repo reality: No `sudo` entrypoint implemented. Exchange interaction is via a submessage whose reply is handled synchronously within the same tx (`reply_on_success`). **Path does not exist**.

- Allegation: “WasmMsg reentrancy before cleanup.”
  - Repo reality: No `WasmMsg::Execute` to external Wasm contracts in the swap flow. The only submessage is a custom Injective exchange message, not a Wasm contract call. **Path does not exist**.

- Allegation: “Multi-transaction stepper state hijacking.”
  - Repo reality: Multi-step swaps are orchestrated by chaining submessages and replies inside the same transaction. The `reply` path computes next step or finalizes and cleans up state; there is no separate user transaction between steps. **No cross-transaction window**.

- Allegation: “reply_on_success leaves dirty state on failure.”
  - Repo reality: If the submessage fails, the entire execute fails and all writes in this transaction are rolled back by CosmWasm. Cleanup in reply is not needed to prevent persistence; the state changes won’t persist on failure. **No dirty state remains after a failed submessage.**

---

## Formal Answers to Required Questions

- **Is this a real vulnerability, or intended behavior?**
  - In this repository, the singleton usage supports a single, atomic execute flow. Given synchronous `reply_on_success` and no async callbacks, this is **intended behavior** and not a vulnerability.

- **Is it 100% exploitable in a realistic on-chain scenario?**
  - **No.** The prerequisite async or reentrant vectors are absent. All critical reads from singletons happen inside the same transaction that wrote them.

- **Actor prerequisites (permissions, timing, states, economic thresholds)?**
  - Not applicable; there is no reachable exploit path as implemented.

- **Theoretical and real-world financial impact?**
  - **None** under current implementation. Payouts are sent to the original sender stored earlier in the same atomic flow and then state is cleaned.

- **Conditions for exploit to work or fail?**
  - Would require adding: IBC or `sudo` async callbacks that read from global singletons without correlation, or adding `WasmMsg::Execute` to untrusted contracts before state cleanup. Under current code, **exploit fails**.

- **Mitigations/protections present and effectiveness?**
  - Effective protections by design: No IBC/sudo handlers, no Wasm reentrancy surface, `reply_on_success` synchronous flow, state removal at end of flow, and full transaction rollback on failures.

- **Achievable under protocol conditions or only contrived?**
  - Only contrived mock storage overwrites demonstrate that `Item<T>` is a singleton. They do not represent on-chain atomicity and do not yield an exploit in this codebase.

---

## Additional Notes and Recommendations

- While not currently exploitable, singleton usage is brittle if the contract is later extended with async flows (IBC/sudo) or external Wasm calls. If that happens, switch to correlated `Map<K,V>` state keyed by operation IDs or sender, verify ownership in callbacks, and consider reentrancy guards when introducing Wasm calls.

- Current code correctly confines swap execution within a single transaction using Injective’s atomic spot order submessages and reply handling, then clears state. This aligns with CosmWasm’s atomicity guarantees.

---

## Non-Existence of Claimed Files/Paths

- Referenced in prior analysis but absent here:
  - `/contracts/swap/src/ibc.rs` — not present
  - Any `ibc_*` entrypoints — not present
  - Any `sudo` entrypoint — not present
  - Any `WasmMsg::Execute` path in swap flow — not present

These missing components are the necessary preconditions for the claimed exploits; without them, the exploit paths are not reachable.
