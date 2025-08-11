### Injective Swap Contract – Definitive Exploitability Analysis

- Scope: `contracts/swap` (current repository state)
- Goal: 100% verification of exploitability under CosmWasm semantics (transactions, reply, atomicity)
- Outcome: Definitive yes/no answers; no speculation

---

## Executive Verdict (Concise)

- Global singleton items exist. However, the swap is executed as a single atomic transaction with a synchronous submessage reply to the Injective exchange module. There are no IBC handlers, no sudo entrypoints, and no external Wasm calls.
- Under these exact conditions, the alleged exploit paths are not reachable. The contract writes and consumes singleton state within the same transaction and removes it before the transaction completes.
- Therefore, the reported “singleton overwrite -> fund theft” exploit is not achievable in this codebase as written.

---

## Control/Data Flow and Entry Points (Citations)

- Entry points implemented: `instantiate`, `execute`, `reply`, `query`, `migrate`.
```startLine:21:endLine:31:/workspace/contracts/swap/src/contract.rs
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut<InjectiveQueryWrapper>,
    env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response<InjectiveMsgWrapper>, ContractError> { ... }
```
```startLine:34:endLine:66:/workspace/contracts/swap/src/contract.rs
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut<InjectiveQueryWrapper>,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response<InjectiveMsgWrapper>, ContractError> { ... }
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

- Storage definitions (singletons exist):
```startLine:6:endLine:12:/workspace/contracts/swap/src/state.rs
pub const SWAP_ROUTES: Map<(String, String), SwapRoute> = Map::new("swap_routes");
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
pub const STEP_STATE: Item<CurrentSwapStep> = Item::new("current_step_cache");
pub const SWAP_RESULTS: Item<Vec<SwapResults>> = Item::new("swap_results");
pub const CONFIG: Item<Config> = Item::new("config");
```

- Swap start writes state and immediately subcalls with reply_on_success:
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
```startLine:144:endLine:155:/workspace/contracts/swap/src/swap.rs
let order_message = SubMsg::reply_on_success(
    create_spot_market_order_msg(contract.to_owned(), order),
    ATOMIC_ORDER_REPLY_ID,
);
...
let response = Response::new().add_submessage(order_message);
Ok(response)
```

- Reply handler runs in the same transaction; reads then clears state and sends funds:
```startLine:181:endLine:189:/workspace/contracts/swap/src/swap.rs
let swap = SWAP_OPERATION_STATE.load(deps.storage)?;
let has_next_market = swap.swap_steps.len() > (current_step.step_idx + 1) as usize;
...
let next_market = querier.query_spot_market(&next_market_id)?.market.expect("market should be available");
```
```startLine:227:endLine:245:/workspace/contracts/swap/src/swap.rs
let send_message = BankMsg::Send {
    to_address: swap.sender_address.to_string(),
    amount: vec![new_balance.clone().into()],
};
...
SWAP_OPERATION_STATE.remove(deps.storage);
STEP_STATE.remove(deps.storage);
SWAP_RESULTS.remove(deps.storage);
```

---

## Verification of Claimed Attack Vectors (Yes/No)

- IBC async callbacks (ibc_packet_ack) state hijack: No
  - Reason: No IBC entrypoints exist in the repo; no IBC messages or handlers.

- Sudo callbacks (exchange order fills) state hijack: No
  - Reason: No `sudo` entrypoint; exchange interaction is via a submessage whose reply is handled within the same transaction.

- WasmMsg reentrancy before cleanup: No
  - Reason: No `WasmMsg::Execute` to external contracts in the swap flow; only a custom Injective message is used.

- Multi-transaction stepper hijacking: No
  - Reason: Steps are coordinated via reply within the same transaction; no user-facing second transaction to continue the flow.

- SubMsg failure leaving dirty state: No
  - Reason: With `reply_on_success`, if the submessage fails, the execute reverts and storage writes from this transaction are not committed. Dirty state cannot persist from a failed submessage.

---

## Definitive Q&A

- Is this a real vulnerability, or intended behavior?
  - Intended behavior here. Singletons are used only within a single atomic execute+reply flow and cleared at the end.

- Is it 100% exploitable in a realistic on-chain scenario?
  - No. The required async or reentrancy surfaces are absent; writes/reads occur within one transaction.

- Can any external actor trigger it (permissions/timing/states/economics)?
  - Not applicable; no reachable exploit path given current entrypoints and flow.

- Theoretical and real-world financial impact?
  - None under current code. Final payout uses the sender stored during the same transaction; state is removed before commit completes.

- Conditions where it works or fails?
  - Would require adding IBC/sudo async handling or untrusted Wasm calls that read global singletons without correlation; under current code it fails.

- Existing mitigations and their effectiveness?
  - Effective by design: synchronous submessage + reply path, no IBC/sudo/WasmMsg, end-of-flow cleanup, and transaction rollback on failure.

- Achievable under protocol conditions or only contrived?
  - Only contrived storage overwrite tests (that ignore transaction atomicity) show singleton behavior. They do not translate to an exploit here.

---

## CosmWasm Semantics Summary (relevant to this repo)

- Atomicity: State writes and replies within a single transaction are atomic; failures revert all writes.
- Reply (reply_on_success): Runs before the tx completes; no other transaction can interleave between execute and its reply.
- No IBC/sudo/Wasm reentrancy surfaces present here. Therefore the classic singleton-overwrite async exploits are inapplicable.

---

## Final Conclusion

- The “global state overwrite -> hijack payout” exploit is not achievable in this codebase as written because the write, subcall, reply, and cleanup occur atomically within one transaction and there are no async callbacks or external Wasm calls.
- If future changes add IBC/sudo callbacks or external Wasm calls, singleton usage would become unsafe and must be replaced with correlated `Map<K,V>` keyed by operation ID or actor, plus ownership checks and (if needed) reentrancy guards.