### Consolidated & Normalized Findings

- **Project**: `contracts/swap` (Injective swap contract)

- **Unique Vulnerabilities**
  1. **Over-refund in Exact-Output path (first hop uses quote input)**
     - Category: Logic Error, Accounting
     - Severity: High (DoS) to Critical (potential fund drain if contract holds extra funds)
     - Root: Refund uses `estimation.result_quantity` instead of `required_input`; also adds arbitrary `+ FPDecimal::ONE` for quote-input case, inflating required input and creating a mismatch.
  2. **Unsafe unwrap/expect in reply parsing and market queries**
     - Category: Error Handling / DoS
     - Severity: Medium (transaction-level panic/DoS)
     - Root: Multiple `unwrap()`/`expect()` in `parse_market_order_response` and spot market queries can panic.
  3. **Global singleton `SWAP_OPERATION_STATE` cross-user overwrite/hijack**
     - Category: State Management
     - Severity: None (Not exploitable under CosmWasm submessage execution model)
     - Root: Singleton state is not user-scoped, but reply execution is atomic within a single tx; external calls cannot interleave. No reentrancy path to overwrite state mid-execution.
  4. **Unbounded fee_percent may cause division-by-zero/invalid math**
     - Category: Arithmetic
     - Severity: Medium
     - Root: In target-side sell estimation, division by `(1 - fee_percent)` with no bound check; if `fee_percent >= 1`, division-by-zero/negative occurs.
  5. **Rounding manipulation/precision discrepancies**
     - Category: Rounding/Precision
     - Severity: Low to Medium (value deviation, potential user overpayment)
     - Root: Mixed use of `round_up_to_min_tick` vs `round_to_min_tick` and ad-hoc `+1` can bias results; consistency needed.

- **Duplicates/Overlaps Resolved**
  - Multiple reports citing “refund uses `estimation.result_quantity` instead of `required_input`” and “+1 unit added” are consolidated under item 1.
  - “Rounding inconsistency” and “rounding manipulation” are consolidated under item 5.
  - The “singleton overwrite reentrancy” is analyzed and determined non-exploitable in this model.

- **Proposed Fixes Mapped**
  - Add max fee bound and checks: maps to item 4.
  - Use checked math and consistent rounding for sell orders: maps to item 5.
  - Remove arbitrary `+1` in exact-output: maps to item 1.
  - Consistent buffer fund computation: maps to items 1 and 5.
  - Input fund validation and slippage protection: general hardening for items 1/5.