# UTxO Indexing — O(1) Input-Output Linking

Uses redeemer-carried indices to link specific inputs to specific outputs
without iterating. Prevents double satisfaction by proving which output
belongs to which input.

## Types

```aiken
pub type TokenDatum {
  owner: ByteArray,
  token_count: Int,
}

pub type IndexedRedeemer {
  input_index: Int,
  output_index: Int,
}
```

## Validator

```aiken
// validators/utxo_indexer.ak

use aiken/collection/list
use cardano/assets
use cardano/transaction.{InlineDatum, Input, Output, OutputReference, Transaction}

validator utxo_indexer {
  spend(
    datum: Option<TokenDatum>,
    redeemer: IndexedRedeemer,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    // O(1) lookup via redeemer indices — no iteration needed
    expect Some(input) = list.at(tx.inputs, redeemer.input_index)
    expect Some(output) = list.at(tx.outputs, redeemer.output_index)

    // CRITICAL: Verify the index points to OUR input (prevents index spoofing)
    let correct_input = (input.output_reference == oref)?

    // Validate the transition
    expect InlineDatum(raw) = output.datum
    expect new_datum: TokenDatum = raw
    let preserves_owner = (new_datum.owner == d.owner)?
    let preserves_value =
      (assets.lovelace_of(output.value) >= assets.lovelace_of(input.output.value))?

    correct_input && preserves_owner && preserves_value
  }
}
```

## Tests

```aiken
// validators/utxo_indexer.ak (continued)

test index_lookup_succeeds() {
  let oref = mock_oref(0)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [make_input(oref, 5, 10_000_000)],
      outputs: [make_output(10, 10_000_000)],
    }
  utxo_indexer.spend(
    Some(TokenDatum { owner: mock_owner, token_count: 5 }),
    IndexedRedeemer { input_index: 0, output_index: 0 },
    oref,
    tx,
  )
}

test index_lookup_with_multiple_utxos() {
  // Our input is at index 1, our output is at index 2
  let our_oref = mock_oref(1)
  let other_oref = mock_oref(0)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [
        make_input(other_oref, 99, 2_000_000),
        make_input(our_oref, 5, 10_000_000),
      ],
      outputs: [
        make_output(99, 2_000_000),
        make_output(99, 3_000_000),
        make_output(10, 10_000_000),
      ],
    }
  utxo_indexer.spend(
    Some(TokenDatum { owner: mock_owner, token_count: 5 }),
    IndexedRedeemer { input_index: 1, output_index: 2 },
    our_oref,
    tx,
  )
}

test wrong_input_index_fails() fail {
  // Index 0 points to a different input — should fail
  let our_oref = mock_oref(1)
  let other_oref = mock_oref(0)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [
        make_input(other_oref, 99, 2_000_000),
        make_input(our_oref, 5, 10_000_000),
      ],
      outputs: [make_output(10, 10_000_000)],
    }
  utxo_indexer.spend(
    Some(TokenDatum { owner: mock_owner, token_count: 5 }),
    IndexedRedeemer { input_index: 0, output_index: 0 },
    our_oref,
    tx,
  )
}

test out_of_bounds_index_fails() fail {
  let oref = mock_oref(0)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [make_input(oref, 5, 10_000_000)],
      outputs: [make_output(10, 10_000_000)],
    }
  utxo_indexer.spend(
    Some(TokenDatum { owner: mock_owner, token_count: 5 }),
    IndexedRedeemer { input_index: 5, output_index: 0 },
    oref,
    tx,
  )
}
```

## Key Concepts Demonstrated

1. **`list.at(xs, index)`** — O(1) lookup by index (returns `Option`)
2. **Redeemer-directed computation** — off-chain code provides indices, on-chain verifies
3. **Index verification** — `input.output_reference == oref` prevents spoofing
4. **Double satisfaction prevention** — each input points to a specific output, not "any" output
5. **`expect Some(x) = list.at(...)`** — crashes on out-of-bounds (desired behavior)

## Security: Why Index Verification Matters

Without `input.output_reference == oref`:
```
Attacker submits redeemer { input_index: 0, output_index: 0 }
for Input B, but index 0 points to Input A.
Validator validates Input A's transition, not Input B's.
Input B passes validation without proper checks.
```

The `oref` check ensures the index actually points to the input being validated.
