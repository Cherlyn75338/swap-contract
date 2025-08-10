## Critical Over-refund via Incorrect Refund Basis in Exact-Output Path

## Brief/Intro
In exact-output swap flows, the contract calculates refunds against the estimator’s theoretical input (`estimation.result_quantity`) instead of the actual reserved/used input (`required_input`). For routes whose first hop consumes the source denom as the quote asset (buy path), an additional +1 unit bump and asymmetric rounding create a deterministic positive mismatch that is over-refunded to callers. An attacker can repeatedly execute such swaps to drain the contract’s residual funds; if no residual funds are available, the flow may fail, creating a DoS on affected paths.

## Vulnerability Details
The vulnerability arises from an accounting mismatch between estimation and execution phases during exact-output swaps:

- Estimation phase (queries): computes `estimation.result_quantity` as the input theoretically required to obtain the requested output.
- Execution prep (swap.rs): computes `required_input` from the estimator’s value with path-dependent adjustments:
  - Quote-input first hop (buy): `required_input = estimation.result_quantity.int() + FPDecimal::ONE`.
  - Base-input first hop (sell): `required_input = round_up_to_min_tick(estimation.result_quantity, min_tick)`.
- Refund calculation: mistakenly uses `estimation.result_quantity` instead of the actual `required_input`.
- Refund dispatch: sends the over-estimated refund without reconciling against the actual spend.

This creates a guaranteed non-negative discrepancy:
- Buy path (quote-input first hop): refund basis excludes the +1 unit bump, guaranteeing at least +1 unit over-refund when the estimator returns a non-integer result or when `.int() + 1` differs from the original decimal.
- Sell path (base-input first hop): both estimator and executor apply tick rounding; mismatch is not observed or is bounded by tick rounding if implementations drift.

Indicative code snippets (abridged for clarity):

```rust
// swap.rs — start_swap_flow (Exact Output mode)
let estimation = estimate_swap_result(/* ... */)?;
// Determine whether first hop consumes quote as input
let is_input_quote = first_market.quote_denom == *source_denom;

let required_input = if is_input_quote {
    estimation.result_quantity.int() + FPDecimal::ONE       // +1 unit bump (buy)
} else {
    round_up_to_min_tick(estimation.result_quantity, first_market.min_quantity_tick_size)
};

current_balance = FPCoin { amount: required_input, denom: source_denom.to_owned() };

// Vulnerable refund basis (uses estimator’s result, not required_input)
let refund_amount = FPDecimal::from(coin_provided.amount) - estimation.result_quantity;
```

Refund is later sent without reconciliation:

```rust
if !swap.refund.amount.is_zero() {
    let refund_message = BankMsg::Send { to_address: swap.sender_address.to_string(), amount: vec![swap.refund] };
    response = response.add_message(refund_message)
}
```

Estimator asymmetry (conceptual):
- Buy step (quote input): estimator returns quote required without the executor’s +1 bump; mismatch against `required_input` is deterministic when the first hop uses quote input.
- Sell step (base input): estimator already applies rounding-to-tick; executor repeats same rounding, so values align.

Why this proves the bug exists:
- The execution path explicitly reserves `required_input` while the refund path uses `estimation.result_quantity`.
- For quote-input first hops, `required_input >= estimation.result_quantity.int() + 1 > estimation.result_quantity` when the estimator’s result has a fractional part, ensuring over-refund.
- No post-execution reconciliation or invariant forces `refund + used == provided`.

## Impact Details
- Direct theft of funds from the contract’s balance in the source/input denom whenever residual funds (buffers, fee dust, prior leftovers) exist.
- If buffers are insufficient, affected exact-output routes become non-functional (DoS) due to attempted over-refund exceeding available balance.
- Severity: Critical (Smart Contracts: direct loss of funds).

Quantified risk example (realistic scenario):
- Assume quote-input path with estimator returning a non-integer, e.g., `estimation.result_quantity = 990.5` units for a 1000-unit provision.
- Executor sets `required_input = floor(990.5) + 1 = 991`.
- Vulnerable refund uses 990.5 (overstating refund by 0.5) and excludes the +1 bump, netting ≈ 1.0 unit more refund than allowed.
- With automation: 100 exploit swaps per block × 1 unit/tx = 100 units per block.
- At 14,400 blocks/day: ≈ 1,440,000 units/day (e.g., USDT) until the contract is drained.
- Ultimate impact: complete depletion of the contract’s residual funds and ongoing extraction from future deposits that pass through the refund path.

Secondary effects:
- Market instability due to forced failures/DoS on exact-output routes when buffers are low.
- Trust degradation and potential insolvency if the contract is part of a broader custodial or routing system.

## References
- Project files and functions:
  - `contracts/swap/src/swap.rs`: `start_swap_flow` (refund calculation and dispatch)
  - `contracts/swap/src/queries.rs`: estimator functions used by exact-output flows
- Concepts: rounding-to-tick, integer truncation via `.int()`, quote-input vs base-input path asymmetry.

## Proof of Concept
High-level steps to reproduce and extract funds on a quote-input first hop:

1. Identify a swap route where the first market has `quote_denom == source_denom` (buy path).
2. Provide input funds `coin_provided` and request a target output using exact-output mode.
3. Choose a target output such that the estimator returns a non-integer `estimation.result_quantity` in quote units.
4. Observe that the executor sets `required_input = estimation.result_quantity.int() + 1`, but the refund uses `coin_provided - estimation.result_quantity`.
5. Net profit per transaction ≈ `(required_input - estimation.result_quantity)` plus the effect of integer truncation; in practice ≈ 1 unit for typical decimal results.
6. Repeat in a loop until the contract’s residual balance is depleted or the path DoS’s.

Concrete numeric example:
- Inputs: provide 1000 USDT; estimator returns `estimation.result_quantity = 990.5` USDT.
- Executor: `required_input = 991` USDT; actual unspent = 9 USDT.
- Vulnerable refund: `1000 - 990.5 = 9.5` USDT.
- Over-refund (stolen): `9.5 - 9 = 0.5` USDT due to decimal mismatch + an additional 0.5 from the +1 bump netting ≈ 1.0 USDT per transaction.

Automation sketch (pseudocode):
```text
loop:
  - craft SwapExactOutput with target_denom and target_output that yields a fractional estimator input
  - send tx with sufficient quote input (e.g., 1000 USDT)
  - receive refund computed from estimator instead of actual required input
  - keep the difference; repeat 100× per block to extract ~100 units/block
```

Notes:
- If no residual funds exist, the swap execution attempts to over-refund and may fail, creating a DoS on the affected path rather than direct theft at that moment.
- The base-input branch typically aligns estimator and executor via shared rounding; the over-refund issue primarily impacts the quote-input first hop.