### Injective Swap Contract — Final Verified Exploit Analysis

Scope: `contracts/swap` (current repo)
Goal: 100% verification of exploitability under CosmWasm semantics (semantics, actor-model, transactions, execute/reply)
Output: Definitive answers, no speculation; code-cited, phase-structured

---

## Executive Summary (Final Verdict)

- The swap flow uses a single-transaction execute + reply_on_success pattern with immediate cleanup. There are no IBC entrypoints, no sudo entrypoint, and no WasmMsg external calls.
- Because the reply handler executes within the same transaction that wrote state, no other transaction can interleave between the write and the reply read. Failed submessages revert the entire transaction (no dirty state persistence).
- Therefore, the alleged paths (singleton overwrite → misdirect payout; submsg failure dirty state; MEV cross-tx overwrite affecting a prior tx’s reply) are NOT reachable in this codebase as written.
- Final: Not exploitable under current implementation; the behavior is intentional and consistent with CosmWasm atomicity.

---

## Phase 1: Line-by-Line Technical Dissection

### Storage layout
```startLine:6:endLine:12:/workspace/contracts/swap/src/state.rs
pub const SWAP_ROUTES: Map<(String, String), SwapRoute> = Map::new("swap_routes");
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
pub const STEP_STATE: Item<CurrentSwapStep> = Item::new("current_step_cache");
pub const SWAP_RESULTS: Item<Vec<SwapResults>> = Item::new("swap_results");
pub const CONFIG: Item<Config> = Item::new("config");
```
- Singletons exist, but are only used inside the atomic swap flow and are removed at the end.

### Entry points and control flow
```startLine:34:endLine:66:/workspace/contracts/swap/src/contract.rs
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(..., msg: ExecuteMsg) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    match msg {
        ExecuteMsg::SwapMinOutput { .. } => start_swap_flow(deps, env, info, target_denom, SwapQuantityMode::MinOutputQuantity(min_output_quantity)),
        ExecuteMsg::SwapExactOutput { .. } => start_swap_flow(deps, env, info, target_denom, SwapQuantityMode::ExactOutputQuantity(target_output_quantity)),
        // Admin functions elided
    }
}
```
```startLine:68:endLine:74:/workspace/contracts/swap/src/contract.rs
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn reply(deps: DepsMut<InjectiveQueryWrapper>, env: Env, msg: Reply) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    match msg.id {
        ATOMIC_ORDER_REPLY_ID => handle_atomic_order_reply(deps, env, msg),
        _ => Err(ContractError::UnrecognizedReply(msg.id)),
    }
}
```
- Not present anywhere: `ibc_*` entrypoints; `sudo` entrypoint; `WasmMsg::Execute` calls in the swap path.

### Swap state write → submessage → reply → cleanup, all in one tx
```startLine:91:endLine:105:/workspace/contracts/swap/src/swap.rs
let swap_operation = CurrentSwapOperation { /* sender, steps, mode, refund, input */ };
SWAP_RESULTS.save(deps.storage, &Vec::new())?;
SWAP_OPERATION_STATE.save(deps.storage, &swap_operation)?;
execute_swap_step(deps, env, swap_operation, 0, current_balance).map_err(ContractError::Std)
```
```startLine:144:endLine:156:/workspace/contracts/swap/src/swap.rs
let order_message = SubMsg::reply_on_success(create_spot_market_order_msg(contract.to_owned(), order), ATOMIC_ORDER_REPLY_ID);
let current_step = CurrentSwapStep { step_idx, current_balance, step_target_denom: estimation.result_denom, is_buy: estimation.is_buy_order };
STEP_STATE.save(deps.storage, &current_step)?;
let response = Response::new().add_submessage(order_message);
Ok(response)
```
```startLine:158:endLine:176:/workspace/contracts/swap/src/swap.rs
pub fn handle_atomic_order_reply(..., msg: Reply) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    let order_response = parse_market_order_response(msg)?;
    let mut swap_results = SWAP_RESULTS.load(deps.storage)?;
    let current_step = STEP_STATE.load(deps.storage).map_err(ContractError::Std)?;
    let swap = SWAP_OPERATION_STATE.load(deps.storage)?;
```
```startLine:227:endLine:245:/workspace/contracts/swap/src/swap.rs
let send_message = BankMsg::Send { to_address: swap.sender_address.to_string(), amount: vec![new_balance.clone().into()] };
...
SWAP_OPERATION_STATE.remove(deps.storage);
STEP_STATE.remove(deps.storage);
SWAP_RESULTS.remove(deps.storage);
```
- The reply is part of the same transaction; cleanup occurs before the tx ends.

---

## Phase 2: Exploit Path Confirmation

We test each claimed vector against the actual code and CosmWasm guarantees (reply executes in the same tx; failures revert all writes; no external async handlers implemented).

- Global singleton overwrite → misdirect prior tx’s funds: No
  - Requirement: TX2 must overwrite `SWAP_OPERATION_STATE` in between TX1’s execute and TX1’s reply. This is impossible because the reply runs inside TX1. No other transaction can interleave.
  - Payout address is taken from `swap.sender_address` loaded in the same tx that stored it; then state is removed.

- SubMsg failure dirty state persistence: No
  - With `reply_on_success`, a failing submessage prevents `reply` and causes the entire TX to revert, rolling back prior writes in that TX. No dirty state persists.

- Race/MEV across transactions affecting a prior tx’s reply: No
  - MEV can reorder different transactions, but cannot interleave inside a single transaction. The critical read of singleton state happens inside the same TX that wrote it.

- IBC async callbacks (ibc_packet_ack): Not applicable
  - No IBC entrypoints exist in the repo.

- Sudo callbacks (exchange fills): Not applicable
  - No `sudo` entrypoint; exchange interaction is via custom message + reply in the same tx.

- WasmMsg reentrancy before cleanup: Not applicable
  - No `WasmMsg::Execute` to untrusted contracts in the swap path.

---

## Phase 3: Mitigation & Countermeasure Analysis

Existing protections (by design):
- Single-transaction execute + reply_on_success flow; reply runs before the tx completes.
- State removed immediately after final payout.
- No async IBC or sudo callbacks; no external Wasm calls in swap path.
- Failed submessage ⇒ full rollback of all writes in the transaction.

Effectiveness: These collectively prevent the alleged exploit paths in this implementation.

Recommendations (only if future async/external surfaces are added):
- Replace singletons with keyed `Map<K,V>` per operation/user and correlate async callbacks by ID; verify ownership.
- Add reentrancy guard if introducing external Wasm calls.

---

## Definitive Q&A

- Is this a real vulnerability, or intended behavior?
  - Intended in this codebase: singletons are used only within one atomic transaction and cleared; not a vulnerability here.

- Is it 100% exploitable on-chain?
  - No. No interleaving between execute and reply; failures revert; no async/external surfaces present.

- Can any external actor trigger it (permissions, timing, states)?
  - Not applicable; no reachable exploit path given current code.

- Financial impact?
  - None under current implementation.

- Conditions under which an exploit would work/fail?
  - Would require adding IBC/sudo async handlers or external Wasm calls that read global singletons without correlation; otherwise fails.

- Existing mitigations and their effectiveness?
  - Effective by design: atomic flow, immediate cleanup, no async/sudo/WasmMsg, and automatic rollback on failure.

- Achievable under actual protocol conditions or only contrived?
  - Only contrived storage demonstrations; not achievable given this implementation’s transaction model.

---

## Closing Statement

Given the actual code and CosmWasm semantics, the alleged singleton-overwrite exploit path (leading to fund misdirection) is not possible in this repository. The swap operation writes, subcalls, replies, and cleans up within a single transaction, eliminating cross-transaction windows and preventing interleaving by other users’ transactions. Any different conclusion would require async handlers or external reentrant calls that this contract simply does not implement.