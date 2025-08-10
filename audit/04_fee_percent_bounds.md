# Unbounded Fee Percent Enables Division-by-Zero/Invalid Math

## Project: contracts/swap/src/queries.rs

## Severity: Medium

## Category: Arithmetic Validation

---

## 🔍 Description
Fee percent used in estimation is derived as:
- `fee_percent = taker_fee_rate * fee_multiplier * (1 - effective_discount)`
If upstream parameters cause `fee_percent >= 1`, then target-side computations divide by `(1 - fee_percent)` leading to division by zero or negative denominators.

## 📜 Affected Code
```rust
// contracts/swap/src/queries.rs
let fee_multiplier = querier.query_market_atomic_execution_fee_multiplier(market_id)?.multiplier;
let fee_percent = market.taker_fee_rate * fee_multiplier * (FPDecimal::ONE - get_effective_fee_discount_rate(&market, is_self_relayer));
```

```startLine:endLine:/workspace/contracts/swap/src/queries.rs
110:114:/workspace/contracts/swap/src/queries.rs
```

```rust
// contracts/swap/src/queries.rs
let required_swap_quantity_in_quote = target_quote_output_quantity / (FPDecimal::ONE - fee_percent);
```

```startLine:endLine:/workspace/contracts/swap/src/queries.rs
323:327:/workspace/contracts/swap/src/queries.rs
```

## 🧠 Root Cause
- No bound checks enforce `0 <= fee_percent < 1` before using it as a divisor complement.

## ⚠️ Exploitability
- Is this vulnerability exploitable? Yes (edge-case DoS / incorrect estimates)
- How: If chain configuration or a bug sets `fee_multiplier` or `taker_fee_rate` high enough, division by zero occurs during estimation, breaking swap flows relying on estimator.

## 💥 Impact
- Smart Contracts: Medium
  - Estimation failure, unusable swap path, potential panic if `FPDecimal` division panics on zero.

## ✅ Remediation Recommendations
- Introduce `MAX_FEE_PERCENT` (e.g., < 1), validate `fee_percent` with `ensure!(fee_percent < MAX_FEE_PERCENT)` and return an error if violated.
- Use safe checked division with explicit error on zero/negative.
- Add tests covering high `fee_multiplier` scenarios and boundary values.

## 🔁 Related Issues
- Rounding/precision consistency interacts with fee computations.

## 🧪 Test Cases
- Mock querier with large `fee_multiplier` causing `fee_percent >= 1`; assert graceful error.
- Boundary test: `fee_percent = 0.999999` validates and computes non-zero denominator.
