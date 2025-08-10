# Division by Zero in Fee Percentage Calculations

## Project: Injective Swap Contract  

## Severity: Medium

## Category: Arithmetic Error / Input Validation

---

## 🔍 Description

The swap contract's fee calculation logic contains potential division by zero vulnerabilities when handling extreme fee percentages. The code performs divisions using `fee_percent` in denominators without validating that the resulting expressions cannot equal zero. This can occur when `fee_percent` equals 1 (100%) in sell operations or approaches -1 in buy operations, leading to transaction failures or incorrect calculations.

## 📜 Affected Code

```rust
// contracts/swap/src/queries.rs line 146 - Buy operation
let available_swap_quote_funds = input_quote_quantity / (FPDecimal::ONE + fee_percent);
// If fee_percent = -1, denominator becomes 0

// contracts/swap/src/queries.rs line 325 - Sell operation  
let required_swap_quantity_in_quote = target_quote_output_quantity / (FPDecimal::ONE - fee_percent);
// If fee_percent = 1, denominator becomes 0

// contracts/swap/src/queries.rs line 112
let fee_percent = market.taker_fee_rate * fee_multiplier * (FPDecimal::ONE - get_effective_fee_discount_rate(&market, is_self_relayer));
// No validation that fee_percent is within safe bounds
```

## 🧠 Root Cause

The root cause stems from multiple factors:

1. **Missing Input Validation**: No bounds checking on `fee_percent` before use in division
2. **Assumption of Valid Fee Rates**: Code assumes market fees will always be reasonable
3. **Complex Fee Calculation**: Multiple multipliers can compound to extreme values
4. **No Safe Math Operations**: Direct division without checking for zero denominators

The vulnerability chain:
- `taker_fee_rate` from market configuration
- Multiplied by `fee_multiplier` from atomic execution
- Modified by discount rates
- Can theoretically reach or exceed 100% (FPDecimal::ONE)

## ⚠️ Exploitability

**Is this vulnerability exploitable?** **Conditionally - Requires specific market conditions**

### Exploitation Scenarios

#### Scenario 1: Malicious Market Configuration
If an attacker gains control over market parameters (through governance or admin access):
```rust
// Attack setup:
// 1. Set market.taker_fee_rate = 0.5 (50%)
// 2. Set fee_multiplier = 2.0
// 3. Result: fee_percent = 0.5 * 2.0 = 1.0 (100%)
// 4. Sell operation: 1 / (1 - 1.0) = 1 / 0 = PANIC
```

#### Scenario 2: Edge Case in Fee Discount
```rust
// If get_effective_fee_discount_rate returns > 1:
// fee_percent = 0.3 * 1.5 * (1 - 1.2) = 0.3 * 1.5 * (-0.2) = -0.09
// While not causing immediate division by zero, negative fees cause logic errors
```

#### Scenario 3: Governance Attack
```rust
// Through malicious governance proposal:
// 1. Propose "fee adjustment" with extreme multipliers
// 2. If passed, all swaps using affected markets fail
// 3. Funds locked in incomplete swap states
```

### Limitations on Exploitability

1. **Requires privileged access**: Market parameters typically controlled by governance
2. **Chain-wide impact**: Would affect all users, making it obvious
3. **Economic disincentive**: Attacker gains no direct profit from DoS
4. **Likely caught in testing**: Extreme fees would be noticed before production

## 💥 Impact

This vulnerability falls under **Medium** severity:

- **Temporary freezing of funds**: Failed swaps due to division by zero
- **DoS on specific markets**: Markets with extreme fees become unusable
- **No direct fund theft**: Errors cause transaction failure, not fund loss
- **Governance manipulation risk**: Could be weaponized through governance

The severity is Medium because:
1. Requires specific conditions or privileged access
2. Impact limited to DoS, not fund theft
3. Easily detectable and reversible through governance
4. Affects protocol availability, not security

## ✅ Remediation Recommendations

### Immediate Fix: Add Fee Percentage Validation

```rust
// Add constant for maximum allowed fee
const MAX_FEE_PERCENT: FPDecimal = FPDecimal::from_str("0.5").unwrap(); // 50% max
const MIN_FEE_PERCENT: FPDecimal = FPDecimal::from_str("-0.1").unwrap(); // -10% min (for discounts)

// Validation function
fn validate_fee_percent(fee_percent: FPDecimal) -> StdResult<FPDecimal> {
    if fee_percent >= FPDecimal::ONE {
        return Err(StdError::generic_err(
            format!("Fee percentage {} would cause division by zero in sell operations", fee_percent)
        ));
    }
    
    if fee_percent <= FPDecimal::NEG_ONE {
        return Err(StdError::generic_err(
            format!("Fee percentage {} would cause division by zero in buy operations", fee_percent)
        ));
    }
    
    if fee_percent > MAX_FEE_PERCENT {
        return Err(StdError::generic_err(
            format!("Fee percentage {} exceeds maximum allowed {}", fee_percent, MAX_FEE_PERCENT)
        ));
    }
    
    if fee_percent < MIN_FEE_PERCENT {
        return Err(StdError::generic_err(
            format!("Fee percentage {} below minimum allowed {}", fee_percent, MIN_FEE_PERCENT)
        ));
    }
    
    Ok(fee_percent)
}

// Updated estimate_single_swap_execution
pub fn estimate_single_swap_execution(
    deps: &Deps<InjectiveQueryWrapper>,
    env: &Env,
    market_id: &MarketId,
    swap_estimation_amount: SwapEstimationAmount,
    is_simulation: bool,
) -> StdResult<StepExecutionEstimate> {
    // ... existing code ...
    
    let raw_fee_percent = market.taker_fee_rate * fee_multiplier * 
        (FPDecimal::ONE - get_effective_fee_discount_rate(&market, is_self_relayer));
    
    // Validate fee percentage before use
    let fee_percent = validate_fee_percent(raw_fee_percent)?;
    
    // ... rest of function
}
```

### Safe Division Operations

```rust
// Safe division helper
fn safe_divide_with_fee(amount: FPDecimal, fee_percent: FPDecimal, is_buy: bool) 
    -> StdResult<FPDecimal> {
    let denominator = if is_buy {
        FPDecimal::ONE + fee_percent
    } else {
        FPDecimal::ONE - fee_percent
    };
    
    // Extra safety check
    if denominator.is_zero() || denominator.abs() < FPDecimal::from_str("0.0001").unwrap() {
        return Err(StdError::generic_err(
            "Division would result in overflow or undefined behavior"
        ));
    }
    
    Ok(amount / denominator)
}

// Updated buy calculation
let available_swap_quote_funds = safe_divide_with_fee(
    input_quote_quantity, 
    fee_percent, 
    true
)?;

// Updated sell calculation  
let required_swap_quantity_in_quote = safe_divide_with_fee(
    target_quote_output_quantity,
    fee_percent,
    false
)?;
```

### Additional Safeguards

1. **Circuit Breaker for Extreme Fees**:
```rust
pub const FEE_CIRCUIT_BREAKER: Item<bool> = Item::new("fee_circuit_breaker");

// Auto-disable swaps if extreme fees detected
if fee_percent > FPDecimal::from_str("0.3").unwrap() { // 30% threshold
    FEE_CIRCUIT_BREAKER.save(deps.storage, &true)?;
    return Err(ContractError::CircuitBreakerTriggered);
}
```

2. **Governance Timelock**:
```rust
// Add delay for fee changes to prevent sudden attacks
pub struct PendingFeeChange {
    pub new_fee: FPDecimal,
    pub effective_after: Timestamp,
}
```

3. **Fee Sanity Checks in Market Creation**:
```rust
fn validate_market_params(market: &SpotMarket) -> StdResult<()> {
    if market.taker_fee_rate > FPDecimal::from_str("0.1").unwrap() {
        return Err(StdError::generic_err("Market taker fee exceeds 10%"));
    }
    // Additional validations...
    Ok(())
}
```

## 🔁 Related Issues

- **Integer Overflow**: Extreme multiplications could cause overflow in other calculations
- **Precision Loss**: Edge cases in fee calculations may lose precision
- **Negative Fees**: Current code doesn't properly handle negative fee scenarios

## 🧪 Test Cases

```rust
#[test]
fn test_division_by_zero_sell_100_percent_fee() {
    let deps = mock_dependencies();
    let env = mock_env();
    
    // Setup market with 100% fee (should fail)
    let mut market = mock_spot_market();
    market.taker_fee_rate = FPDecimal::ONE;
    
    let result = estimate_single_swap_execution(
        &deps.as_ref(),
        &env,
        &market.market_id,
        SwapEstimationAmount::InputQuantity(FPCoin {
            amount: FPDecimal::from(100u128),
            denom: "USDT".to_string(),
        }),
        true,
    );
    
    assert!(result.is_err());
    assert!(result.unwrap_err().to_string().contains("division by zero"));
}

#[test]
fn test_extreme_fee_validation() {
    // Test fee validation catches extreme values
    assert!(validate_fee_percent(FPDecimal::from_str("0.99").unwrap()).is_err());
    assert!(validate_fee_percent(FPDecimal::from_str("-0.99").unwrap()).is_err());
    assert!(validate_fee_percent(FPDecimal::from_str("0.3").unwrap()).is_ok());
}

#[test]
fn test_safe_division_helper() {
    // Test safe division prevents panics
    let result = safe_divide_with_fee(
        FPDecimal::from(100u128),
        FPDecimal::ONE, // 100% fee
        false, // sell operation
    );
    assert!(result.is_err());
}

#[test]
fn test_fee_circuit_breaker() {
    // Test circuit breaker triggers on extreme fees
    let mut deps = mock_dependencies();
    let env = mock_env();
    
    // Simulate extreme fee scenario
    let fee_percent = FPDecimal::from_str("0.35").unwrap(); // 35%
    
    // Should trigger circuit breaker
    let result = process_with_fee_check(deps.as_mut(), env, fee_percent);
    assert!(matches!(result, Err(ContractError::CircuitBreakerTriggered)));
}
```