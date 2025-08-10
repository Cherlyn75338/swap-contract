# Rounding Inconsistencies and Manipulation Risk

## Project: contracts/swap/src/swap.rs, contracts/swap/src/queries.rs, contracts/swap/src/helpers.rs

## Severity: Low to Medium

## Category: Rounding/Precision

---

## 🔍 Description
The code mixes `round_up_to_min_tick` and `round_to_min_tick` in different contexts and adds an extra `+ FPDecimal::ONE` in the exact-output quote-input path. This can lead to biased pricing/quantity decisions:
- Buys: uses `round_to_min_tick` or estimator-driven rounding.
- Sells: similarly but varies across estimation/execution and next-hop preparation.
- First-hop exact-output uses `+1` bump for quote input, inconsistent with tick-based rounding.
These inconsistencies enable small but systematic deviations that can harm users or the contract during edge quantities.

## 📜 Affected Code
```rust
// contracts/swap/src/helpers.rs
pub fn round_up_to_min_tick(num: FPDecimal, min_tick: FPDecimal) -> FPDecimal { /* ... */ }
```

```startLine:endLine:/workspace/contracts/swap/src/helpers.rs
21:33:/workspace/contracts/swap/src/helpers.rs
```

```rust
// contracts/swap/src/swap.rs (exact-output refund precomputation)
let required_input = if is_input_quote {
    estimation.result_quantity.int() + FPDecimal::ONE
} else {
    round_up_to_min_tick(estimation.result_quantity, first_market.min_quantity_tick_size)
};
```

```startLine:endLine:/workspace/contracts/swap/src/swap.rs
67:74:/workspace/contracts/swap/src/swap.rs
```

```rust
// contracts/swap/src/swap.rs (preparing next hop)
if is_next_swap_sell {
    round_to_min_tick(new_quantity, next_market.min_quantity_tick_size)
} else { new_quantity }
```

```startLine:endLine:/workspace/contracts/swap/src/swap.rs
185:196:/workspace/contracts/swap/src/swap.rs
```

## 🧠 Root Cause
- Lack of a unified rounding policy across estimator, execution sizing, and refund calculations.
- Ad-hoc `+1` unit bump rather than tick-aware rounding for quote-input path.

## ⚠️ Exploitability
- Is this vulnerability exploitable? Yes, but limited
- How: Adversaries can craft quantities around tick boundaries to bias refunds or execution sizing, extracting minor value or causing avoidable failures.

## 💥 Impact
- Smart Contracts: Low to Medium
  - Value leakage due to biased rounding; potential user overpayment or under-delivery near ticks.

## ✅ Remediation Recommendations
- Define and document a consistent rounding strategy:
  - For buy orders: round average price up; round base quantity down to min tick when consuming book; size funds using worst price.
  - For sell orders: round average price down; round resulting quote down to tick.
  - Remove arbitrary `+1`; always use tick functions consistent with market’s tick sizes.
- Ensure refund and required input/output computations use the same rounding path as order creation.

## 🔁 Related Issues
- Over-refund in exact-output path is exacerbated by inconsistent rounding.

## 🧪 Test Cases
- Fuzz around tick boundaries for both directions; assert monotonicity and absence of arbitrageable gaps between estimate and execution.
- Specific regression for the removed `+1` path.
