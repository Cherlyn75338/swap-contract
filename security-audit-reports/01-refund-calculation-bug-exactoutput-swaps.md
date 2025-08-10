# Refund Calculation Bug in ExactOutput Swaps

## Project: Injective Swap Contract (contracts/swap/src/swap.rs)

## Severity: Critical

## Category: Logic Error / Arithmetic

---

## 🔍 Description
A critical vulnerability exists in the refund calculation logic for ExactOutput swaps where the first hop uses quote as input. The refund calculation incorrectly uses `estimation.result_quantity` instead of `required_input`, which is the actual amount deducted from the user. This discrepancy allows users to steal funds (reported as 1 USDT per transaction) by receiving refunds based on estimates rather than actual amounts spent.

## 📜 Affected Code
```rust
// Location: contracts/swap/src/swap.rs:86
// The vulnerability occurs in the refund calculation logic
// where estimation.result_quantity is used instead of required_input

// Pseudo-code representation of the vulnerable logic:
let refund_amount = estimation.result_quantity; // VULNERABLE: Should be required_input
// This leads to incorrect refund calculations
```

## 🧠 Root Cause
The root cause is a fundamental logic error in the refund calculation algorithm. The system calculates refunds based on estimated quantities (`estimation.result_quantity`) rather than the actual amount deducted from the user's account (`required_input`). This creates a discrepancy that can be exploited to extract more funds than should be refunded.

## ⚠️ Exploitability
**Yes, this vulnerability is exploitable.**

**Exploitation Method:**
1. User initiates an ExactOutput swap with quote as input for the first hop
2. The system calculates required_input (actual amount needed)
3. User's account is debited the required_input amount
4. During refund calculation, the system incorrectly uses estimation.result_quantity
5. If estimation.result_quantity < required_input, user receives a refund
6. If estimation.result_quantity > required_input, user is overcharged
7. In cases where estimation.result_quantity < required_input, user can extract the difference

**Real-world Impact:** Users can steal approximately 1 USDT per transaction as reported in the audit.

## 💥 Impact
**Critical** - This vulnerability directly results in:
- **Direct theft of user funds** (in-motion)
- **Financial loss** for the protocol
- **Loss of user trust** and potential protocol abandonment
- **Regulatory and legal implications** due to fund misappropriation

## ✅ Remediation Recommendations

### Immediate Fixes:
1. **Correct Refund Calculation:**
   ```rust
   // Replace this:
   let refund_amount = estimation.result_quantity;
   
   // With this:
   let refund_amount = required_input;
   ```

2. **Add Validation:**
   ```rust
   // Ensure refund never exceeds what was actually charged
   let refund_amount = std::cmp::min(estimation.result_quantity, required_input);
   ```

### Long-term Improvements:
1. **Implement comprehensive testing** for edge cases including zero amounts and extreme values
2. **Add invariant checks** to ensure refund amounts never exceed input amounts
3. **Implement proper error handling** for cases where estimation fails
4. **Add logging and monitoring** for refund operations to detect anomalies

## 🔁 Related Issues
- This vulnerability is related to the "arbitrary +1 unit addition" issue that also affects exact output calculations
- The precision loss issues in sell orders may compound this vulnerability

## 🧪 Test Cases

### Test Case 1: ExactOutput Swap with Quote Input
```rust
#[test]
fn test_exactoutput_swap_refund_calculation() {
    // Setup: Create ExactOutput swap with quote as first hop input
    let required_input = Uint128::new(1000);
    let estimation_result = Uint128::new(950);
    
    // Execute swap
    let swap_result = execute_exactoutput_swap(required_input, estimation_result);
    
    // Verify refund calculation uses required_input, not estimation.result_quantity
    assert_eq!(swap_result.refund_amount, required_input);
}
```

### Test Case 2: Edge Case - Zero Amounts
```rust
#[test]
fn test_zero_amount_refund_calculation() {
    // Test with zero amounts to ensure no division by zero or unexpected behavior
    let required_input = Uint128::zero();
    let estimation_result = Uint128::zero();
    
    let swap_result = execute_exactoutput_swap(required_input, estimation_result);
    assert_eq!(swap_result.refund_amount, Uint128::zero());
}
```

### Test Case 3: Extreme Value Handling
```rust
#[test]
fn test_extreme_value_refund_calculation() {
    // Test with maximum possible values
    let required_input = Uint128::MAX;
    let estimation_result = Uint128::MAX;
    
    let swap_result = execute_exactoutput_swap(required_input, estimation_result);
    assert_eq!(swap_result.refund_amount, required_input);
}
```

## 📊 Additional Notes
- This vulnerability affects the core swap functionality and impacts all users performing ExactOutput swaps
- The fix should be deployed immediately as it represents a direct financial risk
- Consider implementing a circuit breaker mechanism to temporarily disable affected swap types until the fix is deployed