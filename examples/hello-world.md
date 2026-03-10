# Hello World — Simple Spend Validator

The simplest possible validator: locks ADA that can only be unlocked by the
owner who provides the correct message.

## Datum and Redeemer

```aiken
// lib/hello_world/types.ak

/// The datum stored with the locked UTxO
type Datum {
  /// The owner who can unlock the funds
  owner: VerificationKeyHash,
}

/// The redeemer provided when spending
type Redeemer {
  /// The message that must match
  msg: ByteArray,
}
```

## Validator

```aiken
// validators/hello_world.ak

use aiken/collection/list
use cardano/transaction.{Transaction}
use hello_world/types.{Datum, Redeemer}

validator hello_world {
  spend(
    datum: Option<Datum>,
    redeemer: Redeemer,
    _output_reference: OutputReference,
    transaction: Transaction,
  ) {
    // Extract datum — fails if missing
    expect Some(d) = datum

    // Owner must sign the transaction
    let must_be_signed =
      list.has(transaction.extra_signatories, d.owner)

    // Message must be "Hello, World!"
    let must_say_hello =
      redeemer.msg == "Hello, World!"

    must_be_signed? && must_say_hello?
  }
}
```

## Tests

```aiken
// validators/hello_world.ak (continued)

// Test helpers
const mock_owner = #"aabbccddee112233445566778899001122334455667788990011223344"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

// --- Unit Tests ---

test owner_can_unlock_with_correct_message() {
  let datum = Datum { owner: mock_owner }
  let redeemer = Redeemer { msg: "Hello, World!" }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_owner],
  }
  hello_world.spend(Some(datum), redeemer, mock_oref(), tx)
}

test wrong_message_fails() fail {
  let datum = Datum { owner: mock_owner }
  let redeemer = Redeemer { msg: "Goodbye!" }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_owner],
  }
  hello_world.spend(Some(datum), redeemer, mock_oref(), tx)
}

test missing_signature_fails() fail {
  let datum = Datum { owner: mock_owner }
  let redeemer = Redeemer { msg: "Hello, World!" }
  // No signatories
  hello_world.spend(Some(datum), redeemer, mock_oref(), transaction.placeholder)
}

test missing_datum_fails() fail {
  let redeemer = Redeemer { msg: "Hello, World!" }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_owner],
  }
  hello_world.spend(None, redeemer, mock_oref(), tx)
}

// --- Property-Based Tests ---

use aiken/fuzz

test prop_any_owner_can_unlock(owner via fuzz.bytearray_fixed(28)) {
  let datum = Datum { owner: owner }
  let redeemer = Redeemer { msg: "Hello, World!" }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [owner],
  }
  hello_world.spend(Some(datum), redeemer, mock_oref(), tx)
}

test prop_wrong_message_always_fails(msg via fuzz.bytearray()) fail {
  // This will fail for any message that isn't "Hello, World!"
  // Very rarely the fuzzer might generate exactly "Hello, World!"
  // but probability is negligible for random bytes
  let datum = Datum { owner: mock_owner }
  let redeemer = Redeemer { msg: msg }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_owner],
  }
  hello_world.spend(Some(datum), redeemer, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **expect Some(d) = datum** — safely unwrap the optional datum
2. **list.has()** — check transaction signatories
3. **? operator** — trace on failure for debugging
4. **transaction.placeholder** — base transaction for testing
5. **Record update syntax** — `Transaction { ..placeholder, field: value }`
6. **Property-based tests** — verify invariants across random inputs
