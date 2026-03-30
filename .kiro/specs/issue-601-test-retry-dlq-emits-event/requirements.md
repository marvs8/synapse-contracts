# Requirements Document

## Introduction

Add a test in `tests/contract_test.rs` that asserts `retry_dlq` emits an `Event::StatusUpdated(tx_id, TransactionStatus::Pending)` event as its last emitted event. The existing test `retry_dlq_resets_transaction_status_to_pending` verifies the stored state but does not assert the emitted event, leaving a gap in event-emission coverage for the DLQ retry path.

## Glossary

- **SynapseContract**: The Soroban smart contract under test, defined in `src/lib.rs`.
- **retry_dlq**: The contract method that retries a failed transaction from the dead-letter queue, restoring its status to `Pending`.
- **StatusUpdated**: The `Event` variant emitted when a transaction's status changes, carrying `(tx_id: SorobanString, new_status: TransactionStatus)`.
- **TransactionStatus**: The `#[contracttype]` enum in `src/types/mod.rs` representing transaction lifecycle states (`Pending`, `Processing`, `Completed`, `Failed`, `Cancelled`).
- **DLQ**: Dead-letter queue — the storage structure holding failed transactions awaiting retry.
- **Event**: The `#[contracttype]` enum in `src/types/mod.rs` representing all contract events.
- **TestEnv**: The Soroban `Env::default()` test environment used in integration tests.

## Requirements

### Requirement 1: Emit StatusUpdated(Pending) Event on retry_dlq

**User Story:** As a contract integrator, I want `retry_dlq` to emit a `StatusUpdated` event with `TransactionStatus::Pending` as the new status, so that off-chain observers can reliably detect when a failed transaction has been re-queued for processing.

#### Acceptance Criteria

1. WHEN `retry_dlq` is called on a transaction in the DLQ, THE `SynapseContract` SHALL emit an `Event::StatusUpdated(tx_id, TransactionStatus::Pending)` event.
2. WHEN `retry_dlq` completes successfully, THE last event emitted by `SynapseContract` SHALL be `Event::StatusUpdated(tx_id, TransactionStatus::Pending)`.
3. THE `test_retry_dlq_emits_status_updated` test SHALL be added to `tests/contract_test.rs` and SHALL pass under `cargo test`.

### Requirement 2: Test Isolation and Setup

**User Story:** As a developer maintaining the test suite, I want the new test to use the existing `setup` helper and follow the conventions in `contract_test.rs`, so that the test is consistent and easy to maintain.

#### Acceptance Criteria

1. THE new test SHALL use the `setup` helper defined in `tests/contract_test.rs` to initialise the contract, admin, and client.
2. THE new test SHALL grant a relayer, add an asset, call `register_deposit`, call `mark_failed`, and then call `retry_dlq` before asserting on events.
3. WHEN the test inspects emitted events, THE test SHALL use `env.events().all()` and `TryFromVal` to decode the `(Event, u32)` tuple, consistent with existing event tests in the file.
4. IF the last event is not `Event::StatusUpdated(tx_id, TransactionStatus::Pending)`, THEN the test SHALL fail with a descriptive `assert_eq!` or `assert!` message.
