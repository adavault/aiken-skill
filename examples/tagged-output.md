# Tagged Output — Double Satisfaction Prevention

Tags each output with a hash derived from its corresponding input's output
reference. Since each UTxO has a unique output reference, the tags are unique,
preventing one output from satisfying multiple input validations.

This pattern uses `crypto.blake2b_256` and `cbor.serialise` to create
deterministic, collision-resistant tags.

## Types

```aiken
pub type TaggedDatum {
  owner: ByteArray,
  amount: Int,
  /// blake2b_256 hash of the input's OutputReference
  input_tag: ByteArray,
}

pub type TagRedeemer {
  output_index: Int,
}
```

## Validator

```aiken
// validators/tagged_output.ak

use aiken/cbor
use aiken/collection/list
use aiken/crypto
use cardano/assets
use cardano/transaction.{InlineDatum, Output, OutputReference, Transaction}

/// Derive a unique tag from an output reference
fn tag_from_oref(oref: OutputReference) -> ByteArray {
  crypto.blake2b_256(cbor.serialise(oref))
}

validator tagged_validator {
  spend(
    datum: Option<TaggedDatum>,
    redeemer: TagRedeemer,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum
    let signed = list.has(tx.extra_signatories, d.owner)?

    // Compute expected tag for THIS input
    let expected_tag = tag_from_oref(oref)

    // Look up output at specified index
    expect Some(output) = list.at(tx.outputs, redeemer.output_index)

    // Output must carry our unique tag
    expect InlineDatum(raw) = output.datum
    expect out_datum: TaggedDatum = raw
    let correct_tag = (out_datum.input_tag == expected_tag)?

    signed && correct_tag
  }
}
```

## Tests

```aiken
// validators/tagged_output.ak (continued)

test tagged_spend_succeeds() {
  let oref = mock_oref(0)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [tagged_input(oref)],
      outputs: [tagged_output(oref)],
      extra_signatories: [mock_owner],
    }
  tagged_validator.spend(
    Some(TaggedDatum {
      owner: mock_owner,
      amount: 5_000_000,
      input_tag: tag_from_oref(oref),
    }),
    TagRedeemer { output_index: 0 },
    oref,
    tx,
  )
}

test wrong_tag_fails() fail {
  let oref = mock_oref(0)
  let wrong_oref = mock_oref(99)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [tagged_input(oref)],
      outputs: [tagged_output(wrong_oref)],  // tagged with wrong oref
      extra_signatories: [mock_owner],
    }
  tagged_validator.spend(
    Some(TaggedDatum {
      owner: mock_owner,
      amount: 5_000_000,
      input_tag: tag_from_oref(oref),
    }),
    TagRedeemer { output_index: 0 },
    oref,
    tx,
  )
}

test two_inputs_two_outputs_succeeds() {
  // Each input has its own tagged output — no double satisfaction possible
  let oref_a = mock_oref(0)
  let oref_b = mock_oref(1)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [tagged_input(oref_a), tagged_input(oref_b)],
      outputs: [tagged_output(oref_a), tagged_output(oref_b)],
      extra_signatories: [mock_owner],
    }
  tagged_validator.spend(
    Some(TaggedDatum {
      owner: mock_owner,
      amount: 5_000_000,
      input_tag: tag_from_oref(oref_a),
    }),
    TagRedeemer { output_index: 0 },
    oref_a,
    tx,
  )
}

// Property: different OutputReferences always produce different tags
test prop_different_orefs_different_tags(
  params via fuzz.both(fuzz.bytearray_fixed(32), fuzz.bytearray_fixed(32)),
) {
  let (txid_a, txid_b) = params
  let oref_a = OutputReference { transaction_id: txid_a, output_index: 0 }
  let oref_b = OutputReference { transaction_id: txid_b, output_index: 0 }
  if txid_a == txid_b {
    tag_from_oref(oref_a) == tag_from_oref(oref_b)
  } else {
    tag_from_oref(oref_a) != tag_from_oref(oref_b)
  }
}
```

## Key Concepts Demonstrated

1. **`crypto.blake2b_256`** — on-chain hashing for unique identifiers
2. **`cbor.serialise`** — serialize any Aiken value to ByteArray (requires `use aiken/cbor`)
3. **`use aiken/cbor`** — must be explicitly imported (not available by default)
4. **Deterministic tagging** — same input always produces same tag
5. **Collision resistance** — different inputs produce different tags (cryptographic guarantee)
6. **Double satisfaction prevention** — each output is uniquely tied to one input

## How It Prevents Double Satisfaction

```
Without tagging:
  Input A → check "output has >= 10 ADA" → Output X (10 ADA) ✓
  Input B → check "output has >= 10 ADA" → Output X (10 ADA) ✓  ← DOUBLE SATISFIED!
  Attacker consumes 20 ADA of inputs, provides only 10 ADA output

With tagging:
  Input A → check "output has tag(A)" → Output X has tag(A) ✓
  Input B → check "output has tag(B)" → Output X has tag(A) ✗  ← REJECTED
  Each input must have its own uniquely tagged output
```
