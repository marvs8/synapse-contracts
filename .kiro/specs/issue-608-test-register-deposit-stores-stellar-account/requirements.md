# Requirements Document

## Introduction

Add a test asserting that the `stellar_account` field on the stored `Transaction` matches the address passed to `register_deposit`. The existing test suite covers `amount`, `memo`, `memo_type`, and `relayer` fields but has no dedicated assertion for `stellar_account` storage correctness.

## Glossary

- **Test_Suite**: The integration test file at `tests/contract_test.rs`
- **SynapseContract**: The Soroban smart contract under test
- **register_deposit**: The contract function that creates and stores a `Transaction`
- **Transaction**: The contract type stored by `register_deposit`, containing a `stellar_account` field of type `Address`
- **stellar_account**: The `Address` argument passed to `register_deposit` that identifies the depositor's Stellar account

## Requirements

### Requirement 1: Test stellar_account field storage

**User Story:** As a developer, I want a test that asserts `tx.stellar_account` equals the address passed to `register_deposit`, so that regressions in stellar_account storage are caught automatically.

#### Acceptance Criteria

1. WHEN `register_deposit` is called with a given `stellar_account` address, THE Test_Suite SHALL retrieve the stored `Transaction` via `get_transaction` and assert that `tx.stellar_account` equals the address that was passed in.
2. THE Test_Suite SHALL use a freshly generated `Address` as the `stellar_account` argument so the assertion is non-trivial.
3. THE Test_Suite SHALL follow the naming convention `test_register_deposit_stores_stellar_account` for the new test function.
4. THE Test_Suite SHALL compile and pass without modifying any contract source files.
