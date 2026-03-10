# Testing Guide

## Overview

Aiken tests run on the same CEK machine as production validators. Test results
include exact CPU and memory execution units — what you measure in tests is what
you pay on-chain.

Run tests: `aiken check`
Run specific: `aiken check -m "test_name"`
Verbose: `aiken check --trace-level verbose`

## Unit Tests

```aiken
use cardano/transaction.{Transaction}

// Basic test — must evaluate to True
test simple_addition() {
  1 + 1 == 2
}

// Testing a validator
test owner_can_unlock() {
  let owner = #"aabbccdd"
  let datum = Datum { owner: owner }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [owner],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref(), tx)
}

// Expected failure — test passes if validator fails
test stranger_cannot_unlock() fail {
  let owner = #"aabbccdd"
  let stranger = #"11223344"
  let datum = Datum { owner: owner }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [stranger],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref(), tx)
}

// Expected failure with specific trace message
test fails_with_message() fail @"missing signature" {
  // ... test that produces "missing signature" trace
  my_validator.spend(Some(datum), Unlock, mock_oref(), transaction.placeholder)
}
```

## Building Test Transactions

Use `transaction.placeholder` as a base and override fields:

```aiken
use cardano/transaction.{
  Transaction, Input, Output, OutputReference, InlineDatum,
  NoDatum, placeholder,
}
use cardano/address.{from_verification_key}
use cardano/assets

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_input(owner: VerificationKeyHash, lovelace: Int) -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: from_verification_key(owner),
      value: assets.from_lovelace(lovelace),
      datum: NoDatum,
      reference_script: None,
    },
  }
}

fn build_test_tx(signers: List<VerificationKeyHash>) -> Transaction {
  Transaction {
    ..placeholder,
    extra_signatories: signers,
    inputs: [mock_input(#"aabb", 5_000_000)],
  }
}
```

## Property-Based Testing

Property tests generate random inputs and verify invariants. Uses the
`aiken-lang/fuzz` library.

### Setup

Add to `aiken.toml`:
```toml
[[test_dependencies]]
name = "aiken-lang/fuzz"
version = "v2.2.0"
source = "github"
```

### Writing Property Tests

```aiken
use aiken/fuzz

// Basic property — any valid signer should work
test prop_any_owner_can_unlock(owner via fuzz.bytearray_fixed(28)) {
  let datum = Datum { owner: owner }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [owner],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref(), tx)
}

// Property with multiple fuzzed arguments
test prop_amount_preserved(
  amount via fuzz.int_between(2_000_000, 100_000_000),
  owner via fuzz.bytearray_fixed(28),
) {
  let input_value = assets.from_lovelace(amount)
  let output_value = assets.from_lovelace(amount)
  assets.lovelace_of(input_value) == assets.lovelace_of(output_value)
}

// Negative property — should fail for ANY wrong signer
test prop_wrong_signer_fails(
  owner via fuzz.bytearray_fixed(28),
  stranger via fuzz.bytearray_fixed(28),
) fail {
  // Only fails when owner != stranger (which is overwhelmingly likely
  // for random 28-byte values)
  let datum = Datum { owner: owner }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [stranger],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref(), tx)
}
```

### Available Fuzzers

**Primitives:**
- `fuzz.bool()` — random Bool
- `fuzz.byte()` — 0..255
- `fuzz.int()` — arbitrary Int
- `fuzz.int_between(min, max)` — bounded Int
- `fuzz.int_at_least(min)` / `fuzz.int_at_most(max)`
- `fuzz.bytearray()` — random length ByteArray
- `fuzz.bytearray_fixed(len)` — exact length (use 28 for key hashes, 32 for tx ids)
- `fuzz.bytearray_between(min, max)` — bounded length

**Collections:**
- `fuzz.list(fuzzer)` — random list
- `fuzz.list_between(f, min, max)` — bounded list
- `fuzz.set(fuzzer)` — unique elements
- `fuzz.pick(items)` — choose from list
- `fuzz.sublist(items)` — random sublist
- `fuzz.option(fuzzer)` — Some or None

**Combinators:**
- `fuzz.map(f, fn)` — transform fuzzed value
- `fuzz.and_then(f, fn)` — chain fuzzers
- `fuzz.both(a, b)` — pair of fuzzed values
- `fuzz.either(a, b)` — one or the other
- `fuzz.such_that(f, predicate)` — filter values
- `fuzz.constant(value)` — always returns value

**Cardano-specific (from stdlib cardano/fuzz):**
- `fuzz.address()`, `fuzz.credential()`
- `fuzz.verification_key_hash()`, `fuzz.script_hash()`
- `fuzz.policy_id()`, `fuzz.asset_name()`, `fuzz.value()`
- `fuzz.transaction_id()`, `fuzz.output_reference()`
- `fuzz.input()`, `fuzz.output()`, `fuzz.datum()`

### Distribution Labels

Track input distribution to ensure fuzz coverage:

```aiken
test prop_handles_all_amounts(amount via fuzz.int_between(0, 100_000_000)) {
  let label =
    if amount < 2_000_000 {
      @"below min ADA"
    } else if amount < 50_000_000 {
      @"normal range"
    } else {
      @"large amount"
    }
  fuzz.label(label)
  // ... test logic
  True
}
```

## Scenario Testing

For multi-step state machine validators. Tests sequences of transactions.

```aiken
use aiken/fuzz/scenario.{Done, Step, Scenario}

type VaultState {
  Empty
  Locked { amount: Int, owner: VerificationKeyHash }
  Unlocked
}

fn vault_scenario(state: VaultState) -> Scenario<VaultState> {
  when state is {
    Empty -> {
      let owner = #"aabbccdd"
      let amount = 10_000_000
      let tx = build_lock_tx(owner, amount)
      Step(
        [@"lock"],
        Locked { amount, owner },
        tx,
      )
    }
    Locked { amount, owner } -> {
      let tx = build_unlock_tx(owner, amount)
      Step(
        [@"unlock"],
        Unlocked,
        tx,
      )
    }
    Unlocked -> Done
  }
}

test vault_lifecycle() {
  scenario.run(vault_scenario, Empty, [
    scenario.into_spend_handler(vault.spend),
  ])
}
```

## Test Organisation

```
validators/
  vault.ak              # Validator + inline tests
lib/
  vault/helpers.ak      # Shared functions + their tests
  vault/types.ak        # Types (no tests needed)
```

Tests can live in the same file as the code they test, or in separate files
under `lib/`. All files are scanned by `aiken check`.

## Debugging Failed Tests

1. Add `trace` statements to narrow the failure point
2. Use `cbor.diagnostic(value)` to inspect data structures
3. Run with `--trace-level verbose` for full trace output
4. Check execution units — unexpectedly high values suggest infinite recursion
5. Use `aiken check -m "failing_test"` to isolate
