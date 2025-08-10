# Division by Zero via Unbounded `fee_percent`

## Project: contracts/swap/src/queries.rs

## Severity: Medium

## Category: Arithmetic / Validation

---

## 🔍 Description
`fee_percent` is computed dynamically based on market parameters:

```rust
110:112:/workspace/contracts/swap/src/queries.rs
    let fee_multiplier = querier.query_market_atomic_execution_fee_multiplier(market_id)?.multiplier;
    let fee_percent = market.taker_fee_rate * fee_multiplier * (FPDecimal::ONE - get_effective_fee_discount_rate(&market, is_self_relayer));
```

Later, the code divides by `1 ± fee_percent`:

```rust
146:147:/workspace/contracts/swap/src/queries.rs
    let available_swap_quote_funds = input_quote_quantity / (FPDecimal::ONE + fee_percent);
…
325:325:/workspace/contracts/swap/src/queries.rs
    let required_swap_quantity_in_quote = target_quote_output_quantity / (FPDecimal::ONE - fee_percent);
```

If `fee_percent ≥ 1` (100 %) the denominator of at least one path becomes **zero** or negative, causing a **panic in `FPDecimal` division** or incorrect results.

Although extreme, fee multipliers and discounts are external parameters set by the Injective chain governance; the contract must therefore defensively cap them.

## 🧠 Root Cause
Missing upper-bound validation for `fee_percent` before using it as a divisor.

## ⚠️ Exploitability
**Exploitable:** YES (Denial-of-Service)

A malicious (or faulty) on-chain upgrade could set `taker_fee_rate = 1` or `fee_multiplier = 2`. Subsequent calls to the estimator will hit a division-by-zero panic, trapping the contract.

## 💥 Impact
Temporary or permanent DoS of both **query** and **execute** paths that rely on `estimate_*` helpers. The contract becomes unusable, blocking swaps.

## ✅ Remediation Recommendations
1. Introduce a compile-time constant, e.g. `const MAX_FEE_PERCENT: FPDecimal = FPDecimal::from_str("0.3").unwrap(); // 30%`.
2. After computing `fee_percent` clamp it: `let fee_percent = fee_percent.min(MAX_FEE_PERCENT);`.
3. Alternatively, pre-compute `(1 + fee_percent)` and assert it is `> 0` before division.

## 🔁 Related Issues
• V05 — unchecked arithmetic may mask similar edge cases.

## 🧪 Test Cases
1. Unit test with `fee_percent == 1` expecting error not panic.
2. Governance simulation test applying fee update and ensuring contract continues to operate.