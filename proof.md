## Brief/Intro
A claim was made that the Injective Swap Contract allows theft of user funds via global singleton state (`Item<T>`) being overwritten across users and leaving dirty state on submessage failure. After a line‑by‑line review and mapping to CosmWasm transaction and submessage semantics, this exact attack is not exploitable. All swap steps, including submessage execution and the reply handler, execute atomically inside a single transaction; errors with `reply_on_success` revert the entire transaction and roll back all state writes. There is no cross‑transaction window where another user can overwrite in‑flight state, so funds cannot be redirected by a second user.

## Vulnerability Details
The reported issue centers on three singletons used to maintain the in‑progress swap:

```7:9:/workspace/contracts/swap/src/state.rs
pub const SWAP_OPERATION_STATE: Item<CurrentSwapOperation> = Item::new("current_swap_cache");
pub const STEP_STATE: Item<CurrentSwapStep> = Item::new("current_step_cache");
pub const SWAP_RESULTS: Item<Vec<SwapResults>> = Item::new("swap_results");
```

These are written when a swap begins and read/updated across steps:

```99:104:/workspace/contracts/swap/src/swap.rs
SWAP_RESULTS.save(deps.storage, &Vec::new())?;
SWAP_OPERATION_STATE.save(deps.storage, &swap_operation)?;

execute_swap_step(deps, env, swap_operation, 0, current_balance).map_err(ContractError::Std)
```

A submessage is dispatched with `reply_on_success`:

```144:152:/workspace/contracts/swap/src/swap.rs
let order_message = SubMsg::reply_on_success(
    create_spot_market_order_msg(contract.to_owned(), order),
    ATOMIC_ORDER_REPLY_ID,
);

let current_step = CurrentSwapStep {
    step_idx,
    current_balance,
    step_target_denom: estimation.result_denom,
    is_buy: estimation.is_buy_order,
};
STEP_STATE.save(deps.storage, &current_step)?;
```

The reply loads the same singletons to continue the swap and, on final step, sends funds to the original sender and cleans state:

```175:186:/workspace/contracts/swap/src/swap.rs
let mut swap_results = SWAP_RESULTS.load(deps.storage)?;

let current_step = STEP_STATE.load(deps.storage).map_err(ContractError::Std)?;
// ...
let swap = SWAP_OPERATION_STATE.load(deps.storage)?;
```

```227:245:/workspace/contracts/swap/src/swap.rs
// last step, finalize and send back funds to a caller
let send_message = BankMsg::Send {
    to_address: swap.sender_address.to_string(),
    amount: vec![new_balance.clone().into()],
};
// ...
SWAP_OPERATION_STATE.remove(deps.storage);
STEP_STATE.remove(deps.storage);
SWAP_RESULTS.remove(deps.storage);
```

Why this is not exploitable under CosmWasm semantics:
- Submessages execute and their replies are handled synchronously within the same outer transaction (depth‑first). There is no “later block” callback window. See CosmWasm’s Actor Model and Transactions semantics.
- With `reply_on_success`, if the submessage errors, the entire transaction fails and all state updates in the calling contract (including writes to `Item<T>`) are rolled back. No dirty state persists.
- Since one contract instance processes one transaction at a time, another user’s transaction cannot interleave before the first transaction’s reply completes. Therefore, a second user cannot overwrite in‑flight state used by an active reply.

Thus, the described fund‑redirection via global singleton overwrite cannot occur. The PoC that assumes persisted state on failure, or cross‑tx overwrites impacting a pending reply, does not model actual CosmWasm execution.

## Impact Details
- Direct fund theft via overwriting `SWAP_OPERATION_STATE` between users is not feasible because there is no cross‑transaction concurrency window in which a reply would read attacker‑controlled state from a previous or next transaction.
- Submessage failures with `reply_on_success` revert the outer transaction entirely, preventing any partial state persistence or mis‑direction of funds.
- Multi‑step swaps are handled via sequential submessages and replies within the same transaction; the singleton usage is safe for this intra‑transaction workflow.

Residual risks and robustness notes (non‑critical):
- Panics in `parse_market_order_response` on unexpected reply formats could cause DoS for a failing module response (not theft):

```260:269:/workspace/contracts/swap/src/swap.rs
let binding = msg.result.into_result().map_err(ContractError::SubMsgFailure).unwrap();
let first_message = binding.msg_responses.first();
let order_response = MsgCreateSpotMarketOrderResponse::decode(first_message.unwrap().value.as_slice())
    .map_err(|err| ContractError::ReplyParseFailure { id: msg.id, err: err.to_string(), })
    .unwrap();
```
- If in the future `reply_on_success` were changed to `reply_always` while still using the same parsing/cleanup logic, additional care would be needed to handle error replies safely (still within the same transaction, but avoiding panics and ensuring cleanup on error paths).

## References
- CosmWasm Contract Semantics: Transactions and submessages execute in a single transaction; errors with `ReplyOn::Success` revert the outer tx. See CosmWasm docs on Contract Semantics, Actor Model, Transactions, and Dispatching Submessages.
  - [CosmWasm Contract Semantics — Transactions, Submessages, Rollback](https://docs.cosmwasm.com/docs/1.0/architecture/semantics)
  - [Actor Model: depth‑first submessage execution and no reentrancy](https://docs.cosmwasm.com/docs/1.0/architecture/actor)
- Code cites: `state.rs`, `swap.rs` in this repository (lines embedded above).

## Proof of Concept
The provided PoCs do not simulate CosmWasm’s transaction and submessage semantics and thus are not valid demonstrations of exploitability:
- They directly call `Item::save` and `Item::load` across artificial “transactions” without modeling that a submessage reply occurs before any other user’s transaction can execute.
- They assume `reply_on_success` leaves state in storage on submessage failure. In reality, with `ReplyOn::Success`, a submessage error reverts the entire transaction and all the caller’s prior writes are rolled back.

A realistic model would need to demonstrate a reply reading attacker‑controlled state written by a different user before the reply is handled, which CosmWasm’s single‑transaction, depth‑first submessage handling prevents. Therefore, with 100% confidence given the current code and CosmWasm semantics, the claimed global state overwrite theft path is not exploitable.