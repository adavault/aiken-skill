# Oracle Feed — Reference Input Authentication

Reference inputs are read-only inputs available to a transaction without being
spent (no validator execution required). This makes them ideal for oracle price
feeds, configuration UTxOs, and shared state.

**Security critical:** Reference inputs can be faked — anyone can supply any
UTxO as a reference input. Always authenticate with a known NFT before trusting
the data.

## Types

```aiken
pub type OracleDatum {
  price: Int,
  timestamp: Int,
}

pub type SwapRedeemer {
  oracle_index: Int,
}
```

## Validator

```aiken
// validators/oracle_feed.ak

use aiken/collection/list
use cardano/assets.{PolicyId}
use cardano/transaction.{InlineDatum, Input, Output, OutputReference, Transaction}

const oracle_policy =
  #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

const oracle_token = "ORACLE"

/// Find and authenticate the oracle reference input
fn get_oracle_price(ref_inputs: List<Input>, idx: Int) -> Int {
  expect Some(ref_input) = list.at(ref_inputs, idx)
  // CRITICAL: Verify oracle NFT is present — prevents fake reference inputs
  let has_nft =
    (assets.quantity_of(ref_input.output.value, oracle_policy, oracle_token) == 1)?
  expect True = has_nft
  expect InlineDatum(raw) = ref_input.output.datum
  expect oracle: OracleDatum = raw
  oracle.price
}

validator price_swap {
  spend(
    _datum: Option<Data>,
    redeemer: SwapRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    // Read price from authenticated oracle reference input
    let price = get_oracle_price(tx.reference_inputs, redeemer.oracle_index)
    // Validate price is positive and reasonable
    (price > 0)? && (price < 1_000_000_000)?
  }
}
```

## Tests

```aiken
// validators/oracle_feed.ak (continued)

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn oracle_ref_input(price: Int, timestamp: Int) -> Input {
  Input {
    output_reference: OutputReference {
      transaction_id: #"1111111111111111111111111111111111111111111111111111111111111111",
      output_index: 0,
    },
    output: Output {
      address: address.from_script(oracle_policy),
      value: assets.from_lovelace(2_000_000)
        |> assets.add(oracle_policy, oracle_token, 1),
      datum: InlineDatum(
        OracleDatum { price: price, timestamp: timestamp },
      ),
      reference_script: None,
    },
  }
}

fn fake_ref_input(price: Int) -> Input {
  Input {
    output_reference: OutputReference {
      transaction_id: #"2222222222222222222222222222222222222222222222222222222222222222",
      output_index: 0,
    },
    output: Output {
      address: address.from_verification_key(#"cc01"),
      value: assets.from_lovelace(2_000_000),
      datum: InlineDatum(OracleDatum { price: price, timestamp: 0 }),
      reference_script: None,
    },
  }
}

test oracle_read_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [oracle_ref_input(450_000, 1000)],
    }
  price_swap.spend(None, SwapRedeemer { oracle_index: 0 }, mock_oref(), tx)
}

test oracle_at_index_1_succeeds() {
  // Oracle is second reference input — redeemer index points to it
  let other_ref =
    Input {
      output_reference: OutputReference {
        transaction_id: #"3333333333333333333333333333333333333333333333333333333333333333",
        output_index: 0,
      },
      output: Output {
        address: address.from_verification_key(#"dd01"),
        value: assets.from_lovelace(5_000_000),
        datum: transaction.NoDatum,
        reference_script: None,
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [other_ref, oracle_ref_input(450_000, 1000)],
    }
  price_swap.spend(None, SwapRedeemer { oracle_index: 1 }, mock_oref(), tx)
}

test fake_oracle_without_nft_fails() fail {
  // Reference input without the oracle NFT — must be rejected
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [fake_ref_input(450_000)],
    }
  price_swap.spend(None, SwapRedeemer { oracle_index: 0 }, mock_oref(), tx)
}

test oracle_with_zero_price_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [oracle_ref_input(0, 1000)],
    }
  price_swap.spend(None, SwapRedeemer { oracle_index: 0 }, mock_oref(), tx)
}

test oracle_wrong_index_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [oracle_ref_input(450_000, 1000)],
    }
  // Index 1 doesn't exist — expect fails
  price_swap.spend(None, SwapRedeemer { oracle_index: 1 }, mock_oref(), tx)
}

// Property: any positive price under limit passes
test prop_valid_price_always_passes(
  price via fuzz.int_between(1, 999_999_999),
) {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [oracle_ref_input(price, 1000)],
    }
  price_swap.spend(None, SwapRedeemer { oracle_index: 0 }, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **`tx.reference_inputs`** — `List<Input>` of read-only inputs (no validator runs on them)
2. **NFT authentication** — `assets.quantity_of(ref_input.output.value, policy, name) == 1` before trusting data
3. **`assets.add(value, policy, name, qty)`** — add native token to a Value (pipe-friendly)
4. **`address.from_script(hash)`** — create script address for test outputs
5. **Redeemer-indexed lookup** — `list.at(tx.reference_inputs, idx)` for O(1) access
6. **InlineDatum on reference inputs** — same pattern as regular inputs: `expect InlineDatum(raw) = output.datum`
7. **Fake input detection** — test proves that reference inputs without the oracle NFT are rejected

## When to Use

- **Oracle feeds** — read price data without consuming the oracle UTxO
- **Configuration UTxOs** — shared parameters (fees, limits) read by multiple validators
- **Protocol state** — global state that multiple transactions reference simultaneously
- **Script references** — read a reference script's hash without executing it

## Security Checklist

- [ ] Always verify NFT/authentication token on reference inputs
- [ ] Never trust datum from unauthenticated reference inputs
- [ ] Consider what happens if the oracle UTxO is consumed between tx construction and submission
- [ ] Validate data bounds (price > 0, timestamp freshness, etc.)
