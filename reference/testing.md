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

Use `transaction.placeholder` as a base and override fields.

**InlineDatum in tests:** Aiken auto-coerces custom types to `Data`:
```aiken
// This works — no explicit Data conversion needed
datum: InlineDatum(MyState { count: 0, owner: #"aabb01" })
```

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

Add via CLI (preferred):
```bash
aiken add aiken-lang/fuzz --version v2.2.0
```

This adds to `aiken.toml` under `[[dependencies]]`:
```toml
[[dependencies]]
name = "aiken-lang/fuzz"
version = "v2.2.0"
source = "github"
```

**Note:** Aiken uses `[[dependencies]]` for all packages including test-only
ones. There is no separate `test_dependencies` section.

### Writing Property Tests

**Important:** Tests can only have 0 or 1 fuzzed argument. For multiple fuzzed
values, use `fuzz.both()` to combine into a tuple.

```aiken
use aiken/fuzz

// Basic property — single fuzzed argument
test prop_any_owner_can_unlock(owner via fuzz.bytearray_fixed(28)) {
  let datum = Datum { owner: owner }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [owner],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref(), tx)
}

// Multiple fuzzed values — use fuzz.both() to combine into a tuple
test prop_amount_preserved(
  params via fuzz.both(
    fuzz.int_between(2_000_000, 100_000_000),
    fuzz.bytearray_fixed(28),
  ),
) {
  let (amount, owner) = params
  let input_value = assets.from_lovelace(amount)
  let output_value = assets.from_lovelace(amount)
  assets.lovelace_of(input_value) == assets.lovelace_of(output_value)
}

// Negative property — should fail for ANY wrong signer
test prop_wrong_signer_fails(
  stranger via fuzz.bytearray_fixed(28),
) fail {
  let datum = Datum { owner: mock_owner }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [stranger],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref(), tx)
}

// No arithmetic in fuzzer arguments — use constants
const min_time = 1_700_000_001
// WRONG: fuzz.int_at_least(lock_time + 1)  -- parser error
// RIGHT: fuzz.int_at_least(min_time)

// No complex expressions in `via` clause — fuzz simple types, construct in body
// WRONG: test prop(ref via fuzz.map(fuzz.bytearray_fixed(32), fn(b) { OutputReference { ... } }))
// RIGHT: test prop(fake_txid via fuzz.bytearray_fixed(32)) {
//          let fake_ref = OutputReference { transaction_id: fake_txid, output_index: 0 }
//          ...
//        }
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
