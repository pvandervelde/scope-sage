# Testing Strategy

## Test Categories

1. **Unit Tests**: Test individual functions and types in isolation
2. **Integration Tests**: Test interactions between components
3. **Contract Tests**: Verify trait implementations meet contracts
4. **Property Tests**: Use `proptest` for property-based testing

## Test Coverage Expectations

- **Core Business Logic**: 100% coverage
- **Error Paths**: All error variants tested
- **Edge Cases**: Boundary conditions covered
- **Security-Critical Code**: Exhaustive testing (e.g., token handling, validation)

## Test File Organization

Tests should be organized in separate test files adjacent to the code they test, following this pattern:

**For a source file**: `src/module/file.rs`
**Create test file**: `src/module/file_tests.rs`

**For a module file**: `src/module/mod.rs`
**Create test file**: `src/module/mod_tests.rs`

## Test Module Declaration

Reference the external test file from the source file using:

```rust
#[cfg(test)]
#[path = "<TEST_FILE_NAME_WITH_EXTENSION>"]
mod tests;
```

## Test Organization Patterns

```rust
//! Tests for authentication module.

use super::*;

// Group related tests using module organization.
// Test function names follow: test_{function_or_component}_{scenario}_{expected_outcome}
mod token_tests {
    use super::*;

    #[test]
    fn test_token_creation_valid_input_returns_token() { }

    #[test]
    fn test_token_expiry_expired_timestamp_returns_error() { }
}

mod validation_tests {
    use super::*;

    #[test]
    fn test_validate_credentials_correct_password_returns_ok() { }

    #[test]
    fn test_validate_credentials_wrong_password_returns_invalid_credentials() { }
}
```

### Test Naming Convention

All test function names **must** follow the three-part pattern: `test_{function_or_component}_{scenario}_{expected_outcome}`

- `{function_or_component}`: the function, method, or component under test
- `{scenario}`: the input condition or state being exercised
- `{expected_outcome}`: what correct behaviour looks like

Examples from this codebase:

- `test_event_router_design_label_on_issue_returns_qualifying_event`
- `test_document_signer_empty_body_produces_valid_hash`
- `test_webhook_source_invalid_hmac_returns_401`

Avoid short names like `test_valid_input` — they do not communicate the scenario or expected outcome and make failures harder to diagnose.

## Integration Testing

- Place integration tests in `tests/` directory at crate root
- Use test fixtures and mock patterns for external dependencies
- Test async code with `#[tokio::test]` or appropriate runtime
- Verify error paths and edge cases explicitly
- Use `assert_eq!`, `assert!`, and `matches!` appropriately
