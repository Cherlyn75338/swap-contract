# Consolidated Security Findings

This document represents **Step 1** of the requested audit workflow: every raw message has been parsed, deduplicated, and normalized. The resulting unique issues are catalogued below and grouped by category.

| ID  | Vulnerability                                                             | Category                           | Initial Severity | Main Affected Component(s)                              | Source Reports |
|----|---------------------------------------------------------------------------|------------------------------------|------------------|----------------------------------------------------------|----------------|
| V01 | Incorrect refund calculation in exact-output swaps                       | Logic / Arithmetic                 | Critical         | `contracts/swap/src/swap.rs::start_swap_flow`           | 1, 2, 4        |
| V02 | Global `SWAP_OPERATION_STATE` can be overwritten across users            | Access Control / Re-entrancy      | Critical         | `contracts/swap/src/state.rs`, `swap.rs`                | 3              |
| V03 | `unwrap()` panics in sub-message reply parsing                           | Robustness / Denial-of-Service     | Medium           | `contracts/swap/src/swap.rs::parse_market_order_response` | 1              |
| V04 | Fee percent not bounded ⇒ division-by-zero & precision errors            | Arithmetic / Validation            | Medium           | `contracts/swap/src/queries.rs`                         | Proposed Fixes |
| V05 | Potential integer overflow in unchecked math operations                  | Arithmetic / Overflow              | Low              | Multiple (`swap.rs`, `queries.rs`)                      | 2              |
| V06 | Rounding manipulation via inconsistent tick-size handling                | Logic / Precision                  | Medium           | `swap.rs`, `queries.rs`                                 | 2, proposed    |

Duplicates (e.g., multiple refund-calculation messages) have been merged under a single ID. Each issue now has an individual deep-dive analysis located in `audit_reports/V0X.md`.