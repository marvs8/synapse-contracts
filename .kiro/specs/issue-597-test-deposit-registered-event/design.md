# Design Document

## Overview

Add `test_register_deposit_emits_event` to `tests/contract_test.rs`. The test verifies that calling `register_deposit` emits exactly one `Event::DepositRegistered(tx_id, anchor_transaction_id)` event whose fields match the return value and the input argument respectively.

A secondary finding during design: `src/lib.rs` currently emits `Event::DepositRegistered(id.clone(), caller)` where `caller` is an `Address`, but the `Event::DepositRegistered` variant is typed `(SorobanString, SorobanString)` and the requirements specify the second field must be `anchor_transaction_id`. The emit call must be corrected to `Event::DepositRegistered(id.clone(), anchor_transaction_id.clone())` for the test to pass.

## Architecture

This is a pure test addition with one accompanying bug-fix in the contract emit call. No new modules, traits, or abstractions are introduced.

```
tests/contract_test.rs          ← new test function added here
src/lib.rs (register_deposit)   ← emit call corrected
```

## Components and Interfaces

### Existing: `setup` helper (`tests/contract_test.rs`)

```rust
fn setup(env: &Env) -> (Address, Address, SynapseContractClient<'_>)
```

Initialises the contract, creates an admin, and returns `(admin, contract_id, client)`. The new test uses this directly.

### Existing: `emit` (`src/events/mod.rs`)

```rust
pub fn emit(env: &Env, event: Event)
```

Publishes `(event, ledger_sequence)` under the `"synapse"` topic. Consumers decode with `TryFromVal::<_, (Event, u32)>`.

### Existing: `Event::DepositRegistered` (`src/types/mod.rs`)

```rust
DepositRegistered(SorobanString, SorobanString),
//                ^tx_id          ^anchor_transaction_id
```

### Fix required: `register_deposit` emit call (`src/lib.rs`)

Current (incorrect):
```rust
emit(&env, Event::DepositRegistered(id.clone(), caller));
```

Corrected:
```rust
emit(&env, Event::DepositRegistered(id.clone(), anchor_transaction_id.clone()));
```

## Data Models

No new data models. The relevant existing type:

```rust
// src/types/mod.rs
pub enum Event {
    DepositRegistered(SorobanString, SorobanString), // (tx_id, anchor_transaction_id)
    // ...
}
```

Events are published as `(Event, u32)` tuples where `u32` is the ledger sequence number.

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: DepositRegistered event payload correctness

*For any* valid `register_deposit` call (with a non-empty `anchor_transaction_id`, a registered relayer, and an allowed asset), exactly one `Event::DepositRegistered` event SHALL be present in the event log, its first field SHALL equal the `SorobanString` returned by `register_deposit`, and its second field SHALL equal the `anchor_transaction_id` argument passed by the caller.

**Validates: Requirements 1.1, 1.2, 1.3**

## Error Handling

| Scenario | Behaviour |
|---|---|
| `DepositRegistered` event absent from log | `assert!(found, "DepositRegistered event not found")` fails the test |
| Event payload fields do not match | `assert_eq!` on each field fails with a descriptive message |
| `TryFromVal` decode fails | `.unwrap()` panics, failing the test with a clear Rust panic message |

## Testing Strategy

### Unit / integration test (primary deliverable)

Add `test_register_deposit_emits_event` to `tests/contract_test.rs`:

```rust
#[test]
fn test_register_deposit_emits_event() {
    let env = Env::default();
    let (admin, contract_id, client) = setup(&env);
    let relayer = Address::generate(&env);
    client.grant_relayer(&admin, &relayer);
    client.add_asset(&admin, &SorobanString::from_str(&env, "USD"));

    let anchor_id = SorobanString::from_str(&env, "anchor-597");
    let tx_id = client.register_deposit(
        &relayer,
        &anchor_id,
        &Address::generate(&env),
        &100_000_000,
        &SorobanString::from_str(&env, "USD"),
        &None,
        &None,
        &None,
    );

    let all_events = env.events().all();
    let topics: soroban_sdk::Vec<Val> = (symbol_short!("synapse"),).into_val(&env);

    // Collect all DepositRegistered events emitted by this contract
    let deposit_events: Vec<_> = all_events
        .iter()
        .filter_map(|(contract, event_topics, raw)| {
            if contract != contract_id || event_topics != topics {
                return None;
            }
            match <(Event, u32)>::try_from_val(&env, &raw) {
                Ok((Event::DepositRegistered(id, anchor), _)) => Some((id, anchor)),
                _ => None,
            }
        })
        .collect();

    // Property 1: exactly one DepositRegistered event
    assert_eq!(
        deposit_events.len(),
        1,
        "expected exactly one DepositRegistered event, got {}",
        deposit_events.len()
    );

    let (emitted_tx_id, emitted_anchor_id) = &deposit_events[0];

    // Property 1: first field matches return value
    assert_eq!(
        *emitted_tx_id, tx_id,
        "DepositRegistered tx_id does not match register_deposit return value"
    );

    // Property 1: second field matches input anchor_transaction_id
    assert_eq!(
        *emitted_anchor_id, anchor_id,
        "DepositRegistered anchor_transaction_id does not match input"
    );
}
```

### Property-based testing

The property identified above (payload correctness across all valid inputs) is well-suited to property-based testing using [`proptest`](https://github.com/proptest-rs/proptest) or [`quickcheck`](https://github.com/BurntSushi/quickcheck). However, Soroban's `Env::default()` is not `Send + Sync` and does not integrate cleanly with proptest's runner. The recommended approach is to use a table-driven / parameterised test with multiple distinct `anchor_transaction_id` values to approximate coverage:

```rust
// Feature: issue-597-test-deposit-registered-event, Property 1: DepositRegistered event payload correctness
#[test]
fn test_register_deposit_emits_event_various_anchor_ids() {
    let cases = ["anchor-a", "anchor-b", "ANCHOR-999", "x"];
    for anchor_str in cases {
        let env = Env::default();
        let (admin, contract_id, client) = setup(&env);
        let relayer = Address::generate(&env);
        client.grant_relayer(&admin, &relayer);
        client.add_asset(&admin, &SorobanString::from_str(&env, "USD"));
        let anchor_id = SorobanString::from_str(&env, anchor_str);
        let tx_id = client.register_deposit(
            &relayer, &anchor_id, &Address::generate(&env),
            &100_000_000, &SorobanString::from_str(&env, "USD"),
            &None, &None, &None,
        );
        let all_events = env.events().all();
        let found = all_events.iter().any(|(_, _, raw)| {
            matches!(
                <(Event, u32)>::try_from_val(&env, &raw),
                Ok((Event::DepositRegistered(ref id, ref anchor), _))
                    if *id == tx_id && *anchor == anchor_id
            )
        });
        assert!(found, "DepositRegistered not found for anchor_id={anchor_str}");
    }
}
```

Each iteration covers a distinct input, approximating the "for any valid input" quantifier. Minimum coverage: 4 distinct anchor IDs exercising different lengths and character sets.

**Tag format:** `Feature: issue-597-test-deposit-registered-event, Property 1: DepositRegistered event payload correctness`
