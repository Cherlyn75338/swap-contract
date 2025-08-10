# Panic-Based Denial of Service via Unwrap Operations

## Project: Injective Swap Contract

## Severity: Low

## Category: Error Handling / Denial of Service

---

## 🔍 Description

The swap contract contains multiple instances of `.unwrap()` calls in critical execution paths, particularly in the `handle_atomic_order_reply` and `parse_market_order_response` functions. These unwrap operations can cause the contract to panic when encountering unexpected data, leading to transaction failure and potential denial of service conditions. While not directly exploitable for fund theft, this can disrupt normal contract operations and potentially be used to grief users.

## 📜 Affected Code

```rust
// contracts/swap/src/swap.rs lines 261-270
pub fn parse_market_order_response(msg: Reply) -> StdResult<MsgCreateSpotMarketOrderResponse> {
    let binding = msg.result.into_result()
        .map_err(ContractError::SubMsgFailure)
        .unwrap();  // PANIC POINT 1: Can panic if map_err fails

    let first_message = binding.msg_responses.first();
    let order_response = MsgCreateSpotMarketOrderResponse::decode(
        first_message.unwrap().value.as_slice()  // PANIC POINT 2: Can panic if no messages
    )
    .map_err(|err| ContractError::ReplyParseFailure {
        id: msg.id,
        err: err.to_string(),
    })
    .unwrap();  // PANIC POINT 3: Can panic if decode fails

    Ok(order_response)
}

// Additional unwrap in handle_atomic_order_reply
// contracts/swap/src/swap.rs line 164
let trade_data = match order_response.results {
    Some(trade_data) => Ok(trade_data),
    None => Err(ContractError::CustomError {
        val: "No trade data in order response".to_string(),
    }),
}?;  // This is properly handled, but parse_market_order_response is not
```

## 🧠 Root Cause

The root cause is improper error handling patterns:

1. **Mixing Error Handling Styles**: The code inconsistently uses both `Result` types and `.unwrap()` calls
2. **Assumption of Success**: The code assumes SubMsg replies will always be well-formed
3. **Missing Validation**: No validation of reply message structure before accessing fields
4. **Error Propagation Failure**: Using `.unwrap()` prevents proper error propagation up the call stack

The vulnerability is particularly problematic because:
- SubMsg replies come from external contract calls
- Message format could change in protocol upgrades
- Malformed messages could be injected in certain scenarios

## ⚠️ Exploitability

**Is this vulnerability exploitable?** **Partially - DoS only**

### Attack Scenario 1: Griefing via Malformed Replies

While an attacker cannot directly control SubMsg replies in normal operation, there are edge cases:

1. **Protocol Upgrade Incompatibility**: If the Injective exchange module changes its response format
2. **Chain State Corruption**: Validator misbehavior could produce malformed replies
3. **Indirect Griefing**: Triggering edge cases that cause empty responses

### Attack Scenario 2: Cascading Failures

```rust
// Scenario: Market order fails but produces unexpected response format
// 1. User initiates large swap
// 2. Market order partially executes but encounters an error
// 3. Exchange module returns error response in unexpected format
// 4. parse_market_order_response() panics on .unwrap()
// 5. User's transaction fails, funds locked in contract state
// 6. Contract state becomes corrupted (state not cleaned up)
```

### Limitations on Exploitability

- Attacker cannot directly inject malicious replies
- Requires specific chain conditions or module bugs
- Impact limited to DoS, not fund theft
- Self-limiting as it affects all users equally

## 💥 Impact

This vulnerability falls under **Low** severity for smart contracts:

- **Temporary freezing of operations**: Panic causes transaction rollback
- **User experience degradation**: Failed swaps due to panics
- **Potential state corruption**: If cleanup doesn't occur after panic
- **No direct fund loss**: Funds remain in user control or contract

The severity is Low because:
1. No direct fund theft possible
2. Requires unusual conditions to trigger
3. Impact is temporary (per-transaction)
4. Contract can recover with proper state cleanup

## ✅ Remediation Recommendations

### Immediate Fix: Replace Unwraps with Proper Error Handling

```rust
// FIXED VERSION of parse_market_order_response
pub fn parse_market_order_response(msg: Reply) -> Result<MsgCreateSpotMarketOrderResponse, ContractError> {
    // Handle Result properly without unwrap
    let binding = msg.result.into_result()
        .map_err(|e| ContractError::SubMsgFailure(e))?;

    // Validate message exists before accessing
    let first_message = binding.msg_responses.first()
        .ok_or_else(|| ContractError::CustomError {
            val: "No message responses in reply".to_string(),
        })?;

    // Decode with proper error handling
    let order_response = MsgCreateSpotMarketOrderResponse::decode(first_message.value.as_slice())
        .map_err(|err| ContractError::ReplyParseFailure {
            id: msg.id,
            err: err.to_string(),
        })?;

    Ok(order_response)
}

// Updated handle_atomic_order_reply to handle errors gracefully
pub fn handle_atomic_order_reply(
    deps: DepsMut<InjectiveQueryWrapper>,
    env: Env,
    msg: Reply,
) -> Result<Response<InjectiveMsgWrapper>, ContractError> {
    let dec_scale_factor = dec_scale_factor();

    // Use ? operator instead of unwrap
    let order_response = parse_market_order_response(msg)?;

    let trade_data = order_response.results
        .ok_or_else(|| ContractError::CustomError {
            val: "No trade data in order response".to_string(),
        })?;

    // Continue with safe error handling...
    let average_price = FPDecimal::from_str(&trade_data.price)
        .map_err(|e| ContractError::CustomError {
            val: format!("Failed to parse price: {}", e),
        })?;
    
    let quantity = FPDecimal::from_str(&trade_data.quantity)
        .map_err(|e| ContractError::CustomError {
            val: format!("Failed to parse quantity: {}", e),
        })?;
    
    let fee = FPDecimal::from_str(&trade_data.fee)
        .map_err(|e| ContractError::CustomError {
            val: format!("Failed to parse fee: {}", e),
        })?;

    // Rest of function with proper error handling...
}
```

### Additional Safety Measures

1. **Add Reply Validation**:
```rust
fn validate_reply(msg: &Reply) -> Result<(), ContractError> {
    // Check reply ID is expected
    if msg.id != ATOMIC_ORDER_REPLY_ID {
        return Err(ContractError::UnrecognizedReply(msg.id));
    }
    
    // Validate result is success
    if msg.result.is_err() {
        return Err(ContractError::SubMsgFailure(
            "Reply indicates failure".to_string()
        ));
    }
    
    Ok(())
}
```

2. **Implement Recovery Mechanism**:
```rust
// Add cleanup on any error
fn cleanup_failed_swap(deps: DepsMut, user: Addr) -> StdResult<()> {
    SWAP_OPERATION_STATE.remove(deps.storage, user.clone());
    STEP_STATE.remove(deps.storage, user.clone());
    SWAP_RESULTS.remove(deps.storage, user);
    Ok(())
}
```

3. **Add Circuit Breaker**:
```rust
// Track consecutive failures
pub const FAILURE_COUNT: Item<u32> = Item::new("failure_count");
const MAX_FAILURES: u32 = 10;

// Check before processing
let failures = FAILURE_COUNT.load(deps.storage).unwrap_or(0);
if failures > MAX_FAILURES {
    return Err(ContractError::CircuitBreakerTriggered);
}
```

## 🔁 Related Issues

- **State Cleanup**: Panics may prevent proper state cleanup
- **Error Message Quality**: Generic unwrap panics provide no debugging information
- **Missing Logging**: No event emission on failures for monitoring

## 🧪 Test Cases

```rust
#[test]
fn test_handle_empty_reply_messages() {
    let mut deps = mock_dependencies();
    let env = mock_env();
    
    // Create reply with empty message responses
    let reply = Reply {
        id: ATOMIC_ORDER_REPLY_ID,
        result: SubMsgResult::Ok(SubMsgResponse {
            events: vec![],
            data: None,
            msg_responses: vec![], // Empty!
        }),
    };
    
    // Should return error, not panic
    let result = handle_atomic_order_reply(deps.as_mut(), env, reply);
    assert!(result.is_err());
    match result.unwrap_err() {
        ContractError::CustomError { val } => {
            assert!(val.contains("No message responses"));
        }
        _ => panic!("Wrong error type"),
    }
}

#[test]
fn test_handle_malformed_protobuf() {
    let mut deps = mock_dependencies();
    let env = mock_env();
    
    // Create reply with invalid protobuf data
    let reply = Reply {
        id: ATOMIC_ORDER_REPLY_ID,
        result: SubMsgResult::Ok(SubMsgResponse {
            events: vec![],
            data: None,
            msg_responses: vec![MsgResponse {
                type_url: "test".to_string(),
                value: Binary::from(vec![0xFF, 0xFF, 0xFF]), // Invalid protobuf
            }],
        }),
    };
    
    // Should return error, not panic
    let result = handle_atomic_order_reply(deps.as_mut(), env, reply);
    assert!(result.is_err());
}

#[test]
fn test_state_cleanup_on_error() {
    // Verify state is properly cleaned up even when errors occur
}
```