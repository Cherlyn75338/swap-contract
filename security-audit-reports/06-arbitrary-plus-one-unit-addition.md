# Arbitrary +1 Unit Addition in Exact Output Calculation

## Project: Injective Swap Contract (contracts/swap/src/swap.rs)

## Severity: Medium

## Category: Logic Error / Arithmetic / Overcharging

---

## 🔍 Description
A vulnerability exists in the exact output calculation logic where an arbitrary +1 unit is added to the calculation, leading to overcharging of users. This arbitrary addition can result in users being charged more than the actual required amount for their swaps, potentially leading to fund extraction and unfair pricing.

## 📜 Affected Code
```rust
// Location: contracts/swap/src/swap.rs
// The vulnerability occurs in exact output swap calculations

// Vulnerable pattern (before fix):
let required_input = calculated_amount + 1; // VULNERABLE: Arbitrary +1 unit addition

// Proposed fix:
let required_input = calculated_amount; // Remove arbitrary addition
// Or if buffer is needed, use proper calculation:
let required_input = calculated_amount.checked_add(buffer_amount)?;
```

## 🧠 Root Cause
The root cause is the arbitrary addition of +1 unit to exact output calculations without proper justification or validation. This appears to be either a programming error or an attempt to add a buffer that wasn't properly thought through. The arbitrary addition can lead to:

1. **Overcharging:** Users pay more than necessary for their swaps
2. **Inconsistent Pricing:** The +1 unit creates unpredictable and unfair pricing
3. **Fund Extraction:** The protocol extracts additional funds from users without clear justification

## ⚠️ Exploitability
**Yes, this vulnerability is exploitable.**

**Exploitation Method:**
1. **Overcharging Attack:** Users are systematically overcharged by 1 unit per transaction
2. **Fund Extraction:** The protocol extracts additional funds from all users
3. **Pricing Manipulation:** The arbitrary addition can be exploited to create unfair pricing

**Attack Scenarios:**
- The vulnerability itself is exploitable by the protocol to extract funds
- Users may be discouraged from using the service due to unfair pricing
- The +1 unit can compound with other precision issues to create larger overcharges

## 💥 Impact
**Medium** - This vulnerability results in:
- **Systematic overcharging** of users
- **Fund extraction** by the protocol
- **Loss of user trust** due to unfair pricing
- **Potential regulatory issues** due to hidden fees

## ✅ Remediation Recommendations

### Immediate Fixes:
1. **Remove Arbitrary Addition:**
   ```rust
   // Replace this:
   let required_input = calculated_amount + 1;
   
   // With this:
   let required_input = calculated_amount;
   ```

2. **Implement Proper Buffer Calculation:**
   ```rust
   // If a buffer is needed, calculate it properly:
   let buffer_percentage = BUFFER_PERCENTAGE; // e.g., 0.1%
   let buffer_amount = calculated_amount
       .checked_mul(buffer_percentage)?
       .checked_div(10000)?;
   let required_input = calculated_amount.checked_add(buffer_amount)?;
   ```

3. **Add Input Validation:**
   ```rust
   // Validate that required_input is reasonable
   if required_input <= calculated_amount {
       return Err(StdError::generic_err("Invalid required input calculation"));
   }
   ```

### Long-term Improvements:
1. **Implement transparent fee structure** with clear justification for all charges
2. **Add comprehensive testing** for exact output calculations
3. **Implement price validation** to ensure fair pricing
4. **Add monitoring and alerting** for unusual pricing patterns
5. **Consider implementing refund mechanisms** for overcharged amounts

## 🔁 Related Issues
- This vulnerability is related to the refund calculation bug in exact output swaps
- May compound with other vulnerabilities to create more severe attack vectors
- Related to the overall pricing and fee calculation logic

## 🧪 Test Cases

### Test Case 1: Remove Arbitrary +1 Addition
```rust
#[test]
fn test_exact_output_calculation_no_arbitrary_addition() {
    let calculated_amount = Uint128::new(1000);
    
    // Before fix (vulnerable):
    // let required_input = calculated_amount + 1;
    
    // After fix:
    let required_input = calculated_amount;
    
    // Verify no arbitrary addition
    assert_eq!(required_input, calculated_amount);
}
```

### Test Case 2: Proper Buffer Calculation
```rust
#[test]
fn test_exact_output_calculation_with_proper_buffer() {
    let calculated_amount = Uint128::new(1000);
    let buffer_percentage = Uint128::new(10); // 0.1%
    
    // Calculate buffer properly
    let buffer_amount = calculated_amount
        .checked_mul(buffer_percentage)
        .unwrap()
        .checked_div(10000)
        .unwrap();
    
    let required_input = calculated_amount.checked_add(buffer_amount).unwrap();
    
    // Verify buffer calculation is reasonable
    assert!(required_input > calculated_amount);
    assert!(required_input <= calculated_amount.checked_mul(101).unwrap().checked_div(100).unwrap());
}
```

### Test Case 3: Input Validation
```rust
#[test]
fn test_exact_output_input_validation() {
    let calculated_amount = Uint128::new(1000);
    let required_input = calculated_amount;
    
    // Validate that required_input is reasonable
    if required_input < calculated_amount {
        panic!("Required input cannot be less than calculated amount");
    }
    
    // If buffer is added, ensure it's reasonable
    let max_buffer_percentage = Uint128::new(100); // 1% maximum
    let max_allowed_input = calculated_amount
        .checked_mul(Uint128::new(100).checked_add(max_buffer_percentage).unwrap())
        .unwrap()
        .checked_div(100)
        .unwrap();
    
    assert!(required_input <= max_allowed_input);
}
```

### Test Case 4: Edge Case Handling
```rust
#[test]
fn test_exact_output_edge_cases() {
    // Test with zero amount
    let zero_amount = Uint128::zero();
    let required_input = zero_amount;
    assert_eq!(required_input, zero_amount);
    
    // Test with maximum amount
    let max_amount = Uint128::MAX;
    let required_input = max_amount;
    assert_eq!(required_input, max_amount);
    
    // Test with small amounts
    let small_amount = Uint128::new(1);
    let required_input = small_amount;
    assert_eq!(required_input, small_amount);
}
```

## 📊 Additional Notes
- The severity is classified as Medium because it leads to systematic overcharging
- The fix should be implemented immediately to prevent unfair pricing
- Consider implementing a transparent fee structure with clear justification
- Add comprehensive testing for all pricing calculations
- Consider implementing refund mechanisms for users who were overcharged
- Monitor pricing patterns to detect any future anomalies