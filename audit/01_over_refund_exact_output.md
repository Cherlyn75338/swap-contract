# Over-refund in Exact-Output Path When First Hop Uses Quote Input

## Project: contracts/swap/src/swap.rs

## Severity: High (potential DoS) – can reach Critical if contract pre-holds external buffer funds

## Category: Logic Error, Accounting

---

## 🔍 Description
When executing exact-output swaps, the contract pre-computes a refund before placing the first order. In the quote-input case for the first hop, it derives `required_input` from an estimation and separately computes `refund` as `input_funds - estimation.result_quantity`. This mixes two different quantities:
- `required_input`: quote funds actually needed for the first buy
- `estimation.result_quantity`: estimator output for target path, not necessarily the same as required input

Moreover, the code adds an arbitrary `+ FPDecimal::ONE` to `required_input` in the quote-input path, creating a mismatch between deducted funds and the refund basis. This can over-refund or trigger validation errors, potentially causing DoS or enabling drains if the contract holds residual funds.

## 📜 Affected Code
```rust
let refund_amount = if matches!(swap_quantity_mode, SwapQuantityMode::ExactOutputQuantity(..)) {
    let target_output_quantity = quantity;

    let estimation = estimate_swap_result(
        deps.as_ref(),
        &env,
        source_denom.to_owned(),
        target_denom,
        SwapQuantity::OutputQuantity(target_output_quantity),
    )?;

    let querier = InjectiveQuerier::new(&deps.querier);
    let first_market_id = steps[0].to_owned();
    let first_market = querier.query_spot_market(&first_market_id)?.market.expect("market should be available");

    let is_input_quote = first_market.quote_denom == *source_denom;

    let required_input = if is_input_quote {
        estimation.result_quantity.int() + FPDecimal::ONE
    } else {
        round_up_to_min_tick(estimation.result_quantity, first_market.min_quantity_tick_size)
    };

    let fp_coins: FPDecimal = coin_provided.amount.into();

    if required_input > fp_coins {
        return Err(ContractError::InsufficientFundsProvided(fp_coins, required_input));
    }

    current_balance = FPCoin {
        amount: required_input,
        denom: source_denom.to_owned(),
    };

    FPDecimal::from(coin_provided.amount) - estimation.result_quantity
} else {
    FPDecimal::ZERO
};
```

## 🧠 Root Cause
- Refund is computed against `estimation.result_quantity` rather than the actual `required_input` that will be spent.
- Arbitrary `+ FPDecimal::ONE` inflates `required_input` in the quote-input case, creating a mismatch with refund calculation.
- Mismatch leads to either over-refund (if contract has extra funds) or insufficient funds/validation errors that can DoS the transaction path.

## ⚠️ Exploitability
- Is this vulnerability exploitable? Yes
- How:
  - If the contract holds buffer funds (from fees, previous operations, or prefunded), a user can send a minimal amount, trigger exact-output with quote-as-input first hop, and receive a refund computed from `estimation.result_quantity`, which can exceed the actual spend `required_input` due to rounding and the `+1` behavior. Repeating can gradually drain the buffer.
  - Even without buffer, it can cause deterministic DoS by producing inconsistent accounting leading to `InsufficientFundsProvided` or mismatched post-conditions.

Assumptions: Contract occasionally holds funds beyond the user-provided input (possible in fee-share/self-relayer scenarios), or simply induces consistent failures, blocking certain paths.

## 💥 Impact
- Smart Contracts: High to Critical
  - Potential over-refund resulting in incremental theft from contract-held funds (Critical: direct loss of funds)
  - Transaction failure/DoS for exact-output swaps (High/Medium depending on frequency)

## ✅ Remediation Recommendations
- Compute refund strictly as `input_funds - required_input` where `required_input` is the exact amount charged in the first hop for exact-output.
- Remove the `+ FPDecimal::ONE` bump. Replace with precise tick-aware rounding and slippage-buffered checks.
- Ensure consistent rounding: when deriving `required_input`, apply the same rounding rules used to place the order.
- Add unit tests for quote-input first hop, zero/near-zero amounts, extreme values, and cross-denom paths.

## 🔁 Related Issues
- Rounding manipulation/precision discrepancies
- Unbounded fee percent leading to pathological required input

## 🧪 Test Cases
- Exact-output request with first market input denom equal to quote denom; assert `refund = input_funds - required_input` and no over-refund.
- Edge: `target_output_quantity` small; ensure no `+1` bump; verify min tick rounding.
- Edge: high `fee_percent` near 1; ensure guarded and correct refund.
- Property test comparing estimator vs executed amounts for consistency across random markets.
