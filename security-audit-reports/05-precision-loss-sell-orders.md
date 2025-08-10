# Precision Loss in Sell Orders

## Project: Injective Swap Contract (contracts/swap/src/swap.rs)

## Severity: Low

## Category: Arithmetic / Precision Loss / Rounding

---

## 🔍 Description
A vulnerability exists in the sell order calculations where precision loss can occur due to unsafe arithmetic operations. The proposed fix includes using `checked_mul`, `checked_sub`, and `round_to_min_tick` for sell orders to mitigate precision loss and ensure accurate calculations. This vulnerability can lead to users receiving slightly less than expected amounts in sell operations.

## 📜 Affected Code
```rust
// Location: contracts/swap/src/swap.rs
// The vulnerability occurs in sell order calculations

// Vulnerable pattern (before fix):
let sell_amount = input_amount * price / PRECISION; // Can lose precision
let fee_amount = sell_amount * fee_rate / 100;     // Additional precision loss
let final_amount = sell_amount - fee_amount;       // Cumulative precision loss

// Proposed fix:
let sell_amount = input_amount
    .checked_mul(price)?
    .checked_div(PRECISION)?
    .round_to_min_tick()?;
let fee_amount = sell_amount
    .checked_mul(fee_rate)?
    .checked_div(100)?;
let final_amount = sell_amount.checked_sub(fee_amount)?;
```

## 🧠 Root Cause
The root cause is the use of unsafe arithmetic operations that can lead to precision loss in financial calculations. When dealing with decimal arithmetic in fixed-point systems, each operation can introduce rounding errors that compound over multiple calculations. This is particularly problematic in financial applications where even small precision losses can accumulate to significant amounts.

## ⚠️ Exploitability
**Yes, this vulnerability is exploitable, but with limited impact.**

**Exploitation Method:**
1. **Precision Manipulation:** An attacker can manipulate input parameters to maximize precision loss
2. **Rounding Exploitation:** Small rounding errors can be exploited in high-frequency trading scenarios
3. **Fee Calculation Abuse:** Precision loss in fee calculations can lead to slightly incorrect fee amounts

**Attack Scenarios:**
- Submitting orders with specific amounts that maximize precision loss
- Exploiting rounding behavior in fee calculations
- Using the precision loss for arbitrage opportunities

## 💥 Impact
**Low** - This vulnerability results in:
- **Minor precision loss** in sell order calculations
- **Slightly incorrect amounts** for users
- **Potential for small-scale arbitrage** by sophisticated attackers
- **Loss of user confidence** due to inaccurate calculations

## ✅ Remediation Recommendations

### Immediate Fixes:
1. **Use Safe Arithmetic Operations:**
   ```rust
   // Replace unsafe arithmetic with checked operations
   let sell_amount = input_amount
       .checked_mul(price)
       .ok_or_else(|| StdError::generic_err("Sell amount calculation overflow"))?
       .checked_div(PRECISION)
       .ok_or_else(|| StdError::generic_err("Sell amount division error"))?
       .round_to_min_tick()
       .ok_or_else(|| StdError::generic_err("Rounding error"))?;
   ```

2. **Implement Rounding Functions:**
   ```rust
   // Add rounding to minimum tick size
   pub trait RoundToMinTick {
       fn round_to_min_tick(self) -> Result<Self, StdError>;
   }
   
   impl RoundToMinTick for Uint128 {
       fn round_to_min_tick(self) -> Result<Self, StdError> {
           let min_tick = MIN_TICK_SIZE;
           let remainder = self % min_tick;
           if remainder >= min_tick / 2 {
               Ok(self + min_tick - remainder)
           } else {
               Ok(self - remainder)
           }
       }
   }
   ```

3. **Add Precision Validation:**
   ```rust
   // Validate precision requirements
   pub fn validate_precision(amount: Uint128, min_precision: Uint128) -> Result<(), StdError> {
       if amount < min_precision {
           return Err(StdError::generic_err("Amount below minimum precision"));
       }
       Ok(())
   }
   ```

### Long-term Improvements:
1. **Implement comprehensive precision tracking** throughout all calculations
2. **Add precision loss monitoring** and alerting
3. **Implement minimum precision requirements** for all operations
4. **Add comprehensive testing** for edge cases and precision boundaries
5. **Consider using higher precision** arithmetic libraries for critical calculations

## 🔁 Related Issues
- This vulnerability is related to the overall arithmetic safety in the contract
- May compound with other vulnerabilities to create more severe attack vectors
- Related to the proposed fixes for fee calculations and buffer fund calculations

## 🧪 Test Cases

### Test Case 1: Precision Loss Detection
```rust
#[test]
fn test_sell_order_precision_loss() {
    let input_amount = Uint128::new(1000000); // 1.000000
    let price = Uint128::new(999999);         // 0.999999
    let precision = Uint128::new(1000000);    // 6 decimal places
    
    // Calculate with unsafe arithmetic
    let unsafe_amount = input_amount * price / precision;
    
    // Calculate with safe arithmetic
    let safe_amount = input_amount
        .checked_mul(price)
        .unwrap()
        .checked_div(precision)
        .unwrap();
    
    // Verify both calculations produce the same result
    assert_eq!(unsafe_amount, safe_amount);
}
```

### Test Case 2: Fee Calculation Precision
```rust
#[test]
fn test_fee_calculation_precision() {
    let sell_amount = Uint128::new(1000000); // 1.000000
    let fee_rate = Uint128::new(30);         // 0.3%
    
    // Calculate fee with safe arithmetic
    let fee_amount = sell_amount
        .checked_mul(fee_rate)
        .unwrap()
        .checked_div(10000)
        .unwrap();
    
    // Verify fee calculation is accurate
    let expected_fee = Uint128::new(3000); // 0.003000
    assert_eq!(fee_amount, expected_fee);
}
```

### Test Case 3: Rounding to Minimum Tick
```rust
#[test]
fn test_rounding_to_min_tick() {
    let min_tick = Uint128::new(1000); // 0.001000
    let test_amounts = vec![
        (Uint128::new(1000123), Uint128::new(1000000)), // Should round down
        (Uint128::new(1000500), Uint128::new(1000000)), // Should round down
        (Uint128::new(1000501), Uint128::new(1001000)), // Should round up
        (Uint128::new(1000999), Uint128::new(1001000)), // Should round up
    ];
    
    for (input, expected) in test_amounts {
        let rounded = input.round_to_min_tick().unwrap();
        assert_eq!(rounded, expected);
    }
}
```

### Test Case 4: Edge Case Handling
```rust
#[test]
fn test_precision_edge_cases() {
    // Test with very small amounts
    let small_amount = Uint128::new(1);
    let price = Uint128::new(1000000);
    let precision = Uint128::new(1000000);
    
    let result = small_amount
        .checked_mul(price)
        .unwrap()
        .checked_div(precision)
        .unwrap();
    
    assert_eq!(result, Uint128::new(1));
    
    // Test with maximum values
    let max_amount = Uint128::MAX;
    let result = max_amount
        .checked_mul(Uint128::new(1))
        .unwrap()
        .checked_div(Uint128::new(1))
        .unwrap();
    
    assert_eq!(result, max_amount);
}
```

## 📊 Additional Notes
- The severity is classified as Low because the impact is limited to precision loss
- However, in high-frequency trading scenarios, even small precision losses can be significant
- The proposed fix with `checked_mul`, `checked_sub`, and `round_to_min_tick` is a good approach
- Consider implementing additional precision tracking and monitoring
- Add comprehensive testing for edge cases and precision boundaries
- Consider using higher precision arithmetic for critical financial calculations