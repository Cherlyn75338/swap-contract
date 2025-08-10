# Rounding Manipulation via Inconsistent Tick Handling

## Project: contracts/swap/src/swap.rs & queries.rs

## Severity: Medium

## Category: Logic / Precision

---

## 🔍 Description
The contract uses two different helpers for tick rounding:
* `round_up_to_min_tick` (custom helper in `helpers.rs`)
* `round_to_min_tick` (from `injective_math`)

Depending on whether the path is BUY or SELL and whether the function is called during *estimation* or *execution*, different rounding behaviours are applied. Attackers can front-run around these inconsistencies to extract value or force small refunds.

Example of **upwards** rounding during estimation (BUY):

```rust
158:163:/workspace/contracts/swap/src/queries.rs
    let expected_base_quantity = available_swap_quote_funds / average_price;
    let result_quantity = round_to_min_tick(expected_base_quantity, market.min_quantity_tick_size);
```

Whereas **downwards** rounding is applied in execution path for SELL:

```rust
69:73:/workspace/contracts/swap/src/swap.rs
    round_up_to_min_tick(estimation.result_quantity, first_market.min_quantity_tick_size)
```

Such asymmetry lets adversaries craft swaps that *slip* by one tick, causing predictable micro-profits or forcing the contract into the refund branch (see V01).

## 🧠 Root Cause
Lack of a single authoritative rounding policy results in divergence between estimate and execution phases.

## ⚠️ Exploitability
**Exploitable:** YES

While per-transaction gains are small (1 tick), automated bots can repeatedly exploit the slippage to accumulate funds or congest the contract via repeated tiny refunds.

## 💥 Impact
• Repeated micro-theft or dust accumulation inside the contract.  
• Amplifies V01 by making off-by-one refunds deterministic.

## ✅ Remediation Recommendations
1. Establish a **single rounding function** (e.g., always `round_up_to_min_tick`).
2. Ensure both estimator and executor call the *same* helper.
3. Add integration tests comparing estimator output with actual execution results for random inputs.

## 🔁 Related Issues
• V01 Incorrect Refund Calculation

## 🧪 Test Cases
1. Randomised swap inputs verifying that `(required_input - estimation.result_quantity) < min_tick` invariant holds after fix.
2. Sandwich test: attacker executes back-to-back swaps before and after victim’s to capture predictable tick rounding.