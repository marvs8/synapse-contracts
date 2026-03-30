# Requirements Document

## Introduction

Add a test that verifies `set_max_deposit` takes effect immediately for subsequent deposit enforcement. The test sets a max deposit, successfully deposits at that limit, lowers the limit, then asserts a deposit above the new (lower) limit is rejected. This closes the gap where no test verified that updating `max_deposit` is reflected in subsequent deposit calls.

## Glossary

- **Contract**: The `SynapseContract` Soroban smart contract under test.
- **Admin**: The privileged address that can call `set_max_deposit`.
- **Relayer**: The address authorized to call `register_deposit`.
- **max_deposit**: The upper bound on the `amount` field accepted by `register_deposit`, configurable by the admin.
- **Test**: The Rust integration test added to `tests/contract_test.rs`.

## Requirements

### Requirement 1: max_deposit update is enforced immediately

**User Story:** As a developer, I want a test that confirms lowering `max_deposit` immediately rejects deposits above the new limit, so that I can be confident the enforcement logic reads the current value on every call.

#### Acceptance Criteria

1. WHEN `set_max_deposit` is called with an initial limit and a deposit equal to that limit is registered, THE Contract SHALL accept the deposit without panicking.
2. WHEN `set_max_deposit` is subsequently called with a lower limit and a deposit above the new limit is attempted, THE Contract SHALL panic with `"amount exceeds max deposit"`.
3. THE Test SHALL use unique `anchor_transaction_id` values for each `register_deposit` call to avoid duplicate-anchor panics.
4. THE Test SHALL be named `test_max_deposit_update_enforced_immediately` and placed in `tests/contract_test.rs`.
5. FOR ALL valid sequences of (set high limit → deposit at limit → set lower limit → deposit above lower limit), THE Contract SHALL reject the second deposit after the limit is lowered (round-trip enforcement property).
