# Potential Integer / Fixed-Point Overflow in Arithmetic Routines

## Project: Multiple (swap.rs, queries.rs)

## Severity: Low (Theoretical but credible with extreme inputs)

## Category: Arithmetic / Overflow

---

## 🔍 Description
Throughout the codebase large‐value multiplications are performed on `FPDecimal` without `checked_mul`, `checked_add`, or saturation. Examples include:

```rust
161:166:/workspace/contracts/swap/src/queries.rs
    let expected_base_quantity = available_swap_quote_funds / average_price;
    let result_quantity = round_to_min_tick(expected_base_quantity, market.min_quantity_tick_size);
    // …
    let required_funds = worst_price * expected_base_quantity * (FPDecimal::ONE + fee_percent);
```

If `worst_price` and `expected_base_quantity` are both near `u128::MAX ≈ 10^38`, their product exceeds the backing integer range of `FPDecimal`, triggering *wrap-around* or panic depending on the feature flags of the `injective_math` crate.

## 🧠 Root Cause
No upper-bound or `checked_*` arithmetic guards when multiplying two externally controlled decimals (price & quantity).

## ⚠️ Exploitability
**Exploitable:** *Unlikely but feasible*.

The Injective orderbook APIs cap order sizes, but an attacker could craft an extremely illiquid market with astronomic tick sizes to inflate `worst_price`. Combining that with maximum quantity could overflow the multiplication, crashing the contract during estimation.

## 💥 Impact
• Contract panic ⇒ temporary DoS.  
• No direct fund loss (calculation occurs prior to order placement).

## ✅ Remediation Recommendations
1. Replace chained multiplications with `checked_mul` + graceful error handling.
2. Sanity-cap input values (price ≤ 10^30, quantity ≤ 10^30 etc.).
3. Add unit tests with boundary values (`u128::MAX / 2` etc.).

## 🔁 Related Issues
• V04 — similar unchecked arithmetic caused division-by-zero risk.

## 🧪 Test Cases
1. Fuzz test with generated `worst_price` and `expected_base_quantity` near maximum decimal representable.
2. Assert that computations return `ContractError::Overflow` and do not panic.