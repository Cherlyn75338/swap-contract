### Additional Notes and Best Practices

- Derive and error tracing
  - Ensure error enums include sufficiently descriptive variants; prefer avoiding bare `String` in `SubMsgFailure` to keep context structured.
  - Consider adding `#[derive(Debug)]` where appropriate for richer logging in tests.

- Checked arithmetic
  - Prefer `checked_*` or explicit guards when combining fee multipliers and tick math.

- User-scoped state (future-proofing)
  - If design ever allows cross-tx flows (e.g., IBC or async), scope in-flight state by `(sender, nonce)` to avoid collisions and support concurrency.

- Slippage and guardrails
  - Ensure min-output checks cover estimator/execution rounding gaps; expose optional user-provided slippage.

- Test coverage
  - Add fuzz/property tests around tick edges and fee extremes.
  - Add integration tests for exact-output across quote/base first-hop permutations.