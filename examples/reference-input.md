# Reference Input — CIP-31 Oracle Price Feed

CIP-31 reference inputs pattern with two validators working together:
1. `price_oracle` (mint) — operator-controlled oracle NFT minting/burning
2. `price_swap` (spend) — reads oracle price via reference inputs (CIP-31)

Reference inputs appear in `tx.reference_inputs` — they are READ but NOT CONSUMED.
Multiple transactions can reference the same oracle UTxO simultaneously without
contention.

## Types

```aiken
/// Oracle datum attached to the oracle NFT UTxO.
pub type OracleDatum {
  /// Price in lovelace per USD, e.g., 3_000_000 = 3 ADA per USD
  price_lovelace_per_usd: Int,
  /// POSIX timestamp of last update
  last_updated: Int,
}

/// Actions for the oracle minting policy.
pub type OracleAction {
  CreateOracle
  RemoveOracle
}

/// Datum locked at the swap script address.
pub type SwapDatum {
  /// Owner's verification key hash
  owner: ByteArray,
  /// Policy ID of the oracle NFT to look for in reference inputs
  oracle_policy: ByteArray,
  /// Token name of the oracle NFT
  oracle_token_name: ByteArray,
}

/// Actions for the swap validator.
pub type SwapRedeemer {
  Withdraw
}
```

## Validator

```aiken
// validators/reference_input.ak

use aiken/collection/list
use cardano/assets
use cardano/transaction.{InlineDatum, OutputReference, Transaction}

/// Minting policy for the oracle NFT. Only the oracle operator can mint or burn.
validator price_oracle(oracle_operator: ByteArray) {
  mint(redeemer: OracleAction, _policy_id: ByteArray, tx: Transaction) {
    let signed = list.has(tx.extra_signatories, oracle_operator)?
    when redeemer is {
      CreateOracle -> signed
      RemoveOracle -> signed
    }
  }
}

/// Swap validator that reads oracle price via a CIP-31 reference input.
/// Users lock ADA here; to withdraw they must include a reference input
/// containing the oracle NFT with a valid price datum.
validator price_swap {
  spend(
    datum: Option<SwapDatum>,
    _redeemer: SwapRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    // Owner must sign
    let owner_signed = list.has(tx.extra_signatories, d.owner)?

    // Find the oracle reference input by looking for one containing the oracle NFT
    let oracle_ref =
      list.find(
        tx.reference_inputs,
        fn(ref_input) {
          assets.quantity_of(
            ref_input.output.value,
            d.oracle_policy,
            d.oracle_token_name,
          ) > 0
        },
      )
    expect Some(ref) = oracle_ref
    expect InlineDatum(raw) = ref.output.datum
    expect oracle_data: OracleDatum = raw

    // Verify oracle has valid price (proves the oracle was consulted)
    and {
      owner_signed,
      (oracle_data.price_lovelace_per_usd > 0)?,
    }
  }
}
```

## Tests

```aiken
// validators/reference_input.ak (continued)

const mock_operator =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_owner =
  #"112233445566778899aabbccddeeff00112233445566778899aabbccddeeff00"

const mock_non_operator =
  #"ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00"

const mock_oracle_policy =
  #"eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"

const mock_oracle_token = "ORACLE_NFT"

const mock_script_hash =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_swap_datum() -> SwapDatum {
  SwapDatum {
    owner: mock_owner,
    oracle_policy: mock_oracle_policy,
    oracle_token_name: mock_oracle_token,
  }
}

fn oracle_ref_input(price: Int, timestamp: Int) -> Input {
  Input {
    output_reference: OutputReference {
      transaction_id: #"1111111111111111111111111111111111111111111111111111111111111111",
      output_index: 0,
    },
    output: Output {
      address: address.from_script(mock_oracle_policy),
      value: assets.from_lovelace(2_000_000)
        |> assets.add(mock_oracle_policy, mock_oracle_token, 1),
      datum: InlineDatum(
        OracleDatum { price_lovelace_per_usd: price, last_updated: timestamp },
      ),
      reference_script: None,
    },
  }
}

fn swap_script_input() -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: address.from_script(mock_script_hash),
      value: assets.from_lovelace(50_000_000),
      datum: InlineDatum(mock_swap_datum()),
      reference_script: None,
    },
  }
}

// -- Mint Tests --

test mint_oracle_with_operator_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_operator],
    }
  price_oracle.mint(mock_operator, CreateOracle, mock_oracle_policy, tx)
}

test mint_oracle_without_operator_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_non_operator],
    }
  price_oracle.mint(mock_operator, CreateOracle, mock_oracle_policy, tx)
}

test burn_oracle_with_operator_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_operator],
    }
  price_oracle.mint(mock_operator, RemoveOracle, mock_oracle_policy, tx)
}

test burn_oracle_without_operator_fails() fail {
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [] }
  price_oracle.mint(mock_operator, RemoveOracle, mock_oracle_policy, tx)
}

// -- Spend / Withdraw Tests --

test withdraw_with_valid_oracle_reference_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [oracle_ref_input(3_000_000, 1_700_000_000)],
      extra_signatories: [mock_owner],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}

test withdraw_without_oracle_reference_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [],
      extra_signatories: [mock_owner],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}

test withdraw_with_zero_price_oracle_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [oracle_ref_input(0, 1_700_000_000)],
      extra_signatories: [mock_owner],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}

test withdraw_without_owner_signature_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [oracle_ref_input(3_000_000, 1_700_000_000)],
      extra_signatories: [mock_non_operator],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}

test withdraw_with_fake_oracle_no_nft_fails() fail {
  // Reference input has the right datum shape but lacks the oracle NFT
  let fake_ref =
    Input {
      output_reference: OutputReference {
        transaction_id: #"2222222222222222222222222222222222222222222222222222222222222222",
        output_index: 0,
      },
      output: Output {
        address: address.from_verification_key(mock_non_operator),
        value: assets.from_lovelace(2_000_000),
        datum: InlineDatum(
          OracleDatum {
            price_lovelace_per_usd: 3_000_000,
            last_updated: 1_700_000_000,
          },
        ),
        reference_script: None,
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [fake_ref],
      extra_signatories: [mock_owner],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}

test withdraw_finds_oracle_among_multiple_refs() {
  // Oracle reference input is not the first in the list
  let unrelated_ref =
    Input {
      output_reference: OutputReference {
        transaction_id: #"3333333333333333333333333333333333333333333333333333333333333333",
        output_index: 0,
      },
      output: Output {
        address: address.from_verification_key(mock_non_operator),
        value: assets.from_lovelace(5_000_000),
        datum: NoDatum,
        reference_script: None,
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [
        unrelated_ref,
        oracle_ref_input(3_000_000, 1_700_000_000),
      ],
      extra_signatories: [mock_owner],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}

// -- Property-Based Tests --

test prop_any_valid_price_passes(
  price via fuzz.int_at_least(1),
) {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [swap_script_input()],
      reference_inputs: [oracle_ref_input(price, 1_700_000_000)],
      extra_signatories: [mock_owner],
    }
  price_swap.spend(Some(mock_swap_datum()), Withdraw, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **CIP-31 reference inputs** — `tx.reference_inputs` reads UTxOs without consuming them, eliminating contention
2. **NFT-authenticated oracle** — `assets.quantity_of` on the reference input ensures the oracle datum is legitimate
3. **`list.find` over reference inputs** — locate the oracle among potentially many reference inputs
4. **`InlineDatum` extraction** — `expect InlineDatum(raw) = ref.output.datum` then cast to typed datum
5. **Two-validator pattern** — separate minting policy (oracle lifecycle) and spend validator (oracle consumer)
6. **Parameterized minting policy** — `price_oracle(oracle_operator)` bakes in the authorized operator at compile time
