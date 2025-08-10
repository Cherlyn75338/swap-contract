# Incorrect Refund Calculation in Exact-Output Swaps

## Project: contracts/swap/src/swap.rs

## Severity: Critical

## Category: Logic Error / Arithmetic

---

## 🔍 Description
For `ExactOutput` swaps the contract must *deduct* the **exact** number of input coins that will be consumed by the swap and return the *remainder* (if any) back to the caller.  

Inside `start_swap_flow` the helper `estimate_swap_result` is invoked to predict the amount of input coins required.  When the first hop of the route takes **quote** as the input denomination the function adds **+1** tick to the estimate:

```rust
69:73:/workspace/contracts/swap/src/swap.rs
        let required_input = if is_input_quote {
            estimation.result_quantity.int() + FPDecimal::ONE
        } else {
            round_up_to_min_tick(estimation.result_quantity, first_market.min_quantity_tick_size)
        };
```

Unfortunately, the refund is later calculated **against the un-adjusted estimate** instead of the `required_input` actually stored in `current_balance`:

```rust
75:86:/workspace/contracts/swap/src/swap.rs
        // …snip…
        current_balance = FPCoin { amount: required_input, denom: source_denom.to_owned() };

        // BUG: uses `estimation.result_quantity` not `required_input`
        FPDecimal::from(coin_provided.amount) - estimation.result_quantity
```

This 1-unit mismatch creates an *accounting hole*—the contract schedules a refund it cannot honour without dipping into its own buffer.

## 🧠 Root Cause
`refund_amount` is derived from `estimation.result_quantity` instead of `required_input`, ignoring the rounding (+1) that was *already* applied to reach tick precision.  The error is a classic off-by-one logic bug.

## ⚠️ Exploitability
**Exploitable:** YES

1. Attacker chooses a market where `is_input_quote == true` and crafts an `ExactOutput` swap.
2. They fund the transaction with *exactly* `required_input` coins.
3. The contract instantly earmarks `refund = 1` coin even though every coin is actually needed for the swap route.
4. During finalisation it attempts to `BankMsg::Send` the refund. Two outcomes are possible:
   * If the contract holds a buffer of the refund denom it will pay from its own funds ⇒ **direct theft**.
   * Otherwise the transfer fails, reverting the entire reply handler ⇒ **permanent DoS** for all swaps until fixed.

Either branch is classified as *Critical* because it enables loss or indefinite locking of funds.

## 💥 Impact
• Direct theft of contract-owned funds (if buffer present).  
• Permanent freezing of swap functionality via failed refund (DoS).

## ✅ Remediation Recommendations
1. Compute the refund using the same `required_input` variable:
```rust
let refund_amount = FPDecimal::from(coin_provided.amount) - required_input;
```
2. Add unit tests covering edge cases where `required_input != estimation.result_quantity`.
3. Consider removing the arbitrary `+FPDecimal::ONE` and replacing with deterministic rounding helpers.

## 🔁 Related Issues
• V06 – Rounding Manipulation (same rounding pathway).  
• V05 – Unchecked arithmetic may hide similar off-by-one bugs.

## 🧪 Test Cases
1. **Happy path (no rounding):** provide `coin_provided = required_input`; assert zero refund.
2. **Edge path (quote input):** craft values where `required_input = estimation + 1`; assert that refund is **zero** and swap executes.
3. **Property-based:** randomised amounts around tick boundaries, ensure `refund ≤ coin_provided - required_input` always holds.