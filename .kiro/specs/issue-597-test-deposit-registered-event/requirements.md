# Requirements Document

## Introduction

Add a test in `tests/contract_test.rs` that asserts `register_deposit` emits an `Event::DepositRegistered(tx_id, anchor_transaction_id)` event containing the correct IDs. Currently no test in `contract_test.rs` verifies the payload of the `DepositRegistered` event, leaving a gap in coverage for this critical registration signal.

## Glossary

- **SynapseContract**: The Soroban smart contract under test, defined in `src/lib.rs`.
- **register_deposit**: The contract method that registers a new deposit and returns a `tx_id`.
- **DepositRegistered**: The `Event` variant emitted by `register_deposit`, carrying `(tx_id: SorobanString, anchor_transaction_id: SorobanString)`.
- **tx_id**: The unique transaction identifier returned by `register_deposit` and generated internally via `next_id`.
- **anchor_transaction_id**: The caller-supplied external anchor transaction identifier passed to `register_deposit`.
- **Event**: The `#[contracttype]` enum in `src/types/mod.rs` representing all contract events.
- **TestEnv**: The Soroban `Env::default()` test environment used in integration tests.

## Requirements

### Requirement 1: Emit DepositRegistered Event with Correct Payload

**User Story:** As a contract integrator, I want `register_deposit` to emit a `DepositRegistered` event containing the returned `tx_id` and the supplied `anchor_transaction_id`, so that I can reliably index and audit deposit registrations off-chain.

#### Acceptance Criteria

1. WHEN `register_deposit` is called with a valid `anchor_transaction_id`, THE `SynapseContract` SHALL emit exactly one `Event::DepositRegistered` event.
2. WHEN `register_deposit` emits `Event::DepositRegistered(tx_id, anchor_id)`, THE first field SHALL equal the `SorobanString` returned by `register_deposit`.
3. WHEN `register_deposit` emits `Event::DepositRegistered(tx_id, anchor_id)`, THE second field SHALL equal the `anchor_transaction_id` argument supplied by the caller.
4. THE `test_register_deposit_emits_event` test SHALL be added to `tests/contract_test.rs` and SHALL pass under `cargo test`.

### Requirement 2: Test Isolation and Setup

**User Story:** As a developer maintaining the test suite, I want the new test to use the existing `setup` helper and follow the conventions in `contract_test.rs`, so that the test is consistent and easy to maintain.

#### Acceptance Criteria

1. THE new test SHALL use the `setup` helper defined in `tests/contract_test.rs` to initialise the contract, admin, and client.
2. THE new test SHALL grant a relayer, add an asset, and call `register_deposit` with a unique `anchor_transaction_id` string before asserting on events.
3. WHEN the test inspects emitted events, THE test SHALL use `env.events().all()` and `TryFromVal` to decode the `(Event, u32)` tuple, consistent with existing event tests in the file.
4. IF the `DepositRegistered` event is not found in the event log, THEN the test SHALL fail with a descriptive `assert!` or `assert_eq!` message.
