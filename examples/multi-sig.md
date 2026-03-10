# Multi-Signature — M-of-N Threshold

A spend validator that requires M signatures from a list of N authorized signers.
Demonstrates threshold-based authorization using list operations.

## Types

```aiken
pub type MultiSigDatum {
  signers: List<ByteArray>,
  threshold: Int,
}

pub type MultiSigRedeemer {
  Approve
}
```

## Validator

```aiken
// validators/multi_sig.ak

use aiken/collection/list
use cardano/transaction.{OutputReference, Transaction}

validator multi_sig {
  spend(
    datum: Option<MultiSigDatum>,
    redeemer: MultiSigRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum
    // Single-variant redeemers use `let` instead of `when` (compiler warning otherwise)
    let Approve = redeemer
    let valid_sigs =
      d.signers
        |> list.filter(fn(s) { list.has(tx.extra_signatories, s) })
        |> list.length
    (valid_sigs >= d.threshold)?
  }
}
```

## Tests

```aiken
// validators/multi_sig.ak (continued)

const signer_a = #"aa01"
const signer_b = #"bb02"
const signer_c = #"cc03"

fn mock_datum_2_of_3() -> MultiSigDatum {
  MultiSigDatum {
    signers: [signer_a, signer_b, signer_c],
    threshold: 2,
  }
}

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

// --- Threshold Tests ---

test approve_with_2_of_3_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [signer_a, signer_b],
    }
  multi_sig.spend(Some(mock_datum_2_of_3()), Approve, mock_oref(), tx)
}

test approve_with_all_3_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [signer_a, signer_b, signer_c],
    }
  multi_sig.spend(Some(mock_datum_2_of_3()), Approve, mock_oref(), tx)
}

test approve_with_1_of_3_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [signer_a],
    }
  multi_sig.spend(Some(mock_datum_2_of_3()), Approve, mock_oref(), tx)
}

test approve_with_no_sigs_fails() fail {
  multi_sig.spend(
    Some(mock_datum_2_of_3()),
    Approve,
    mock_oref(),
    transaction.placeholder,
  )
}

test approve_with_wrong_signers_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [#"dd04", #"ee05"],
    }
  multi_sig.spend(Some(mock_datum_2_of_3()), Approve, mock_oref(), tx)
}

// --- Property-Based Tests ---

use aiken/fuzz

test prop_all_signers_always_approve(
  params via fuzz.both(fuzz.bytearray_fixed(28), fuzz.bytearray_fixed(28)),
) {
  let (s1, s2) = params
  let datum = MultiSigDatum { signers: [s1, s2], threshold: 2 }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [s1, s2],
    }
  multi_sig.spend(Some(datum), Approve, mock_oref(), tx)
}

test prop_threshold_1_any_signer_works(
  signer via fuzz.bytearray_fixed(28),
) {
  let datum = MultiSigDatum { signers: [signer, #"ff"], threshold: 1 }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [signer],
    }
  multi_sig.spend(Some(datum), Approve, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Threshold authorization** — M-of-N signature checking
2. **List operations** — `list.filter` + `list.length` pipeline for counting
3. **Single-variant redeemer** — use `let Approve = redeemer` not `when` (avoids compiler warning)
4. **Pipe operator** — chaining `|> list.filter(...) |> list.length`
5. **Flexible key lengths** — test constants can be any byte length; Aiken doesn't enforce 28-byte VKH at the type level
