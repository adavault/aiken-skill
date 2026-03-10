# State Machine — Continuing Output Pattern

A counter validator that maintains state across transactions. The validator
requires its output to go back to the same script address with updated datum
(continuing output pattern).

This demonstrates the most common pattern for stateful contracts in eUTxO.

## Types

```aiken
pub type CounterState {
  count: Int,
  owner: ByteArray,
}

pub type CounterAction {
  Increment
  Reset
}
```

## Validator

```aiken
// validators/state_machine.ak

use aiken/collection/list
use cardano/address
use cardano/assets
use cardano/transaction.{InlineDatum, Input, Output, OutputReference, Transaction}

validator counter {
  spend(
    datum: Option<CounterState>,
    redeemer: CounterAction,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(state) = datum
    // Find our own input to determine the script address
    expect Some(own_input) = transaction.find_input(tx.inputs, oref)
    let script_cred = own_input.output.address.payment_credential

    when redeemer is {
      Increment -> {
        // Find the continuing output to the same script
        let script_outputs =
          list.filter(tx.outputs, fn(o) {
            o.address.payment_credential == script_cred
          })
        // Exactly one continuing output
        expect [continuing_output] = script_outputs
        // Extract and type-check the new datum
        expect InlineDatum(raw) = continuing_output.datum
        expect new_state: CounterState = raw
        // Validate state transition
        (new_state.count == state.count + 1)? &&
        (new_state.owner == state.owner)?
      }
      Reset -> list.has(tx.extra_signatories, state.owner)?
    }
  }
}
```

## Tests

```aiken
// validators/state_machine.ak (continued)

const mock_owner = #"aabb01"

const mock_script_hash =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn script_addr() -> address.Address {
  address.from_script(mock_script_hash)
}

// Helper: create input at the script address with counter state
fn mock_counter_input(count: Int) -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: script_addr(),
      value: assets.from_lovelace(5_000_000),
      // InlineDatum auto-coerces custom types to Data
      datum: InlineDatum(CounterState { count, owner: mock_owner }),
      reference_script: None,
    },
  }
}

// Helper: create output at the script address with counter state
fn mock_counter_output(count: Int) -> Output {
  Output {
    address: script_addr(),
    value: assets.from_lovelace(5_000_000),
    datum: InlineDatum(CounterState { count, owner: mock_owner }),
    reference_script: None,
  }
}

// --- Increment Tests ---

test increment_from_0_to_1() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(0)],
      outputs: [mock_counter_output(1)],
    }
  counter.spend(
    Some(CounterState { count: 0, owner: mock_owner }),
    Increment,
    mock_oref(),
    tx,
  )
}

test increment_from_5_to_6() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(5)],
      outputs: [mock_counter_output(6)],
    }
  counter.spend(
    Some(CounterState { count: 5, owner: mock_owner }),
    Increment,
    mock_oref(),
    tx,
  )
}

test increment_wrong_count_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(0)],
      outputs: [mock_counter_output(3)],  // should be 1
    }
  counter.spend(
    Some(CounterState { count: 0, owner: mock_owner }),
    Increment,
    mock_oref(),
    tx,
  )
}

test increment_changed_owner_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(0)],
      outputs: [
        Output {
          address: script_addr(),
          value: assets.from_lovelace(5_000_000),
          datum: InlineDatum(CounterState { count: 1, owner: #"ff99" }),
          reference_script: None,
        },
      ],
    }
  counter.spend(
    Some(CounterState { count: 0, owner: mock_owner }),
    Increment,
    mock_oref(),
    tx,
  )
}

test increment_no_continuing_output_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(0)],
      outputs: [],
    }
  counter.spend(
    Some(CounterState { count: 0, owner: mock_owner }),
    Increment,
    mock_oref(),
    tx,
  )
}

// --- Reset Tests ---

test reset_by_owner_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(5)],
      extra_signatories: [mock_owner],
    }
  counter.spend(
    Some(CounterState { count: 5, owner: mock_owner }),
    Reset,
    mock_oref(),
    tx,
  )
}

test reset_by_non_owner_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(5)],
      extra_signatories: [#"ff99"],
    }
  counter.spend(
    Some(CounterState { count: 5, owner: mock_owner }),
    Reset,
    mock_oref(),
    tx,
  )
}

// --- Property-Based Tests ---

use aiken/fuzz

test prop_increment_always_valid(
  count via fuzz.int_between(0, 1_000_000),
) {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_counter_input(count)],
      outputs: [mock_counter_output(count + 1)],
    }
  counter.spend(
    Some(CounterState { count, owner: mock_owner }),
    Increment,
    mock_oref(),
    tx,
  )
}

test prop_reset_always_needs_owner(
  params via fuzz.both(
    fuzz.int_between(0, 1_000_000),
    fuzz.bytearray_fixed(28),
  ),
) {
  let (count, owner) = params
  let input =
    Input {
      output_reference: mock_oref(),
      output: Output {
        address: script_addr(),
        value: assets.from_lovelace(5_000_000),
        datum: InlineDatum(CounterState { count, owner }),
        reference_script: None,
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [input],
      extra_signatories: [owner],
    }
  counter.spend(Some(CounterState { count, owner }), Reset, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Continuing output pattern** — output must go back to same script with updated state
2. **`transaction.find_input`** — locates the input being validated by output reference
3. **`InlineDatum` construction** — Aiken auto-coerces custom types to `Data` in `InlineDatum(...)`
4. **`expect` type casting** — `expect new_state: CounterState = raw` casts `Data` back to typed value
5. **Script address matching** — `address.payment_credential` to find outputs to same script
6. **State transition validation** — check both the new count AND preserved owner (prevents state manipulation)
7. **`expect [single] = list`** — pattern match requiring exactly one element (fails otherwise)
8. **`address.from_script(hash)`** — create script addresses in tests

## Security Notes

- **Owner preserved:** The validator checks `new_state.owner == state.owner` to prevent
  an attacker from changing the owner during an increment.
- **Single continuing output:** `expect [continuing_output] = script_outputs` ensures
  exactly one output to the script, preventing output splitting attacks.
- **No double satisfaction risk:** Each input validates its own continuing output by
  matching the script credential from its own address.
