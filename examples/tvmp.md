# Transaction-Level Validator Minting Policy (TVMP)

Spend validators run once per input. When you need transaction-level validation
(global invariants across all inputs/outputs), use a coupled mint+spend validator
where the mint handler validates the entire transaction and the spend handler
just checks that minting occurred.

**Critical learning:** `assets.from_asset(policy, name, 0)` normalises away —
the policy does NOT appear in `assets.policies()`. You must mint an actual token
(quantity >= 1) for the policy to be detectable. This is why TVMP uses "receipt"
tokens rather than zero-quantity minting.

## Validator

```aiken
// validators/tvmp.ak

use aiken/collection/dict
use aiken/collection/list
use aiken/fuzz
use cardano/assets.{PolicyId}
use cardano/transaction.{Input, Output, OutputReference, Transaction}

const receipt_name = "RECEIPT"

validator tvmp_vault {
  /// Mint handler — runs ONCE, validates the entire transaction.
  /// Mints exactly 1 receipt token to signal validation passed.
  mint(_redeemer: Data, policy_id: PolicyId, tx: Transaction) {
    let minted = assets.tokens(tx.mint, policy_id)
    // Must mint exactly 1 receipt token
    let mints_receipt =
      (dict.to_pairs(minted) == [Pair(receipt_name, 1)])?

    // Transaction-level validation: outputs <= inputs (no value creation)
    let input_total =
      list.foldl(tx.inputs, 0, fn(input, acc) {
        acc + assets.lovelace_of(input.output.value)
      })
    let output_total =
      list.foldl(tx.outputs, 0, fn(output, acc) {
        acc + assets.lovelace_of(output.value)
      })

    mints_receipt && (output_total <= input_total)?
  }

  /// Spend handler — just checks that our policy minted in this transaction.
  /// This proves the mint handler ran and validated the whole tx.
  spend(
    _datum: Option<Data>,
    _redeemer: Data,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    let own_policy =
      #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
    list.any(assets.policies(tx.mint), fn(p) { p == own_policy })?
  }
}
```

## Tests

```aiken
// validators/tvmp.ak (continued)

const mock_policy =
  #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_input(lovelace: Int) -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: address.from_verification_key(#"cc01"),
      value: assets.from_lovelace(lovelace),
      datum: transaction.NoDatum,
      reference_script: None,
    },
  }
}

fn mock_output(lovelace: Int) -> Output {
  Output {
    address: address.from_verification_key(#"cc01"),
    value: assets.from_lovelace(lovelace),
    datum: transaction.NoDatum,
    reference_script: None,
  }
}

// -- Mint Handler Tests (Transaction-Level Validation) --

test mint_validates_with_receipt() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(10_000_000)],
      outputs: [mock_output(10_000_000)],
      mint: assets.from_asset(mock_policy, receipt_name, 1),
    }
  tvmp_vault.mint(Void, mock_policy, tx)
}

test mint_allows_fee_reduction() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(10_000_000)],
      outputs: [mock_output(8_000_000)],
      mint: assets.from_asset(mock_policy, receipt_name, 1),
    }
  tvmp_vault.mint(Void, mock_policy, tx)
}

test mint_rejects_value_creation() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(5_000_000)],
      outputs: [mock_output(10_000_000)],
      mint: assets.from_asset(mock_policy, receipt_name, 1),
    }
  tvmp_vault.mint(Void, mock_policy, tx)
}

test mint_rejects_wrong_receipt() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(10_000_000)],
      outputs: [mock_output(10_000_000)],
      mint: assets.from_asset(mock_policy, "WRONG", 1),
    }
  tvmp_vault.mint(Void, mock_policy, tx)
}

test mint_rejects_two_receipts() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(10_000_000)],
      outputs: [mock_output(10_000_000)],
      mint: assets.from_asset(mock_policy, receipt_name, 2),
    }
  tvmp_vault.mint(Void, mock_policy, tx)
}

// -- Spend Handler Tests (Policy Presence Check) --

test spend_with_receipt_minted_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, receipt_name, 1),
    }
  tvmp_vault.spend(None, Void, mock_oref(), tx)
}

test spend_without_policy_fails() fail {
  tvmp_vault.spend(None, Void, mock_oref(), transaction.placeholder)
}

test spend_with_wrong_policy_fails() fail {
  let wrong_policy =
    #"ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(wrong_policy, receipt_name, 1),
    }
  tvmp_vault.spend(None, Void, mock_oref(), tx)
}

// -- Property-Based Test --

test prop_value_preserved_always_valid(
  lovelace via fuzz.int_between(2_000_000, 100_000_000),
) {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(lovelace)],
      outputs: [mock_output(lovelace)],
      mint: assets.from_asset(mock_policy, receipt_name, 1),
    }
  tvmp_vault.mint(Void, mock_policy, tx)
}
```

## Key Concepts Demonstrated

1. **`assets.from_asset(policy, name, 0)` normalises away** — zero-quantity assets don't register in `assets.policies()`. Must mint actual tokens (qty >= 1).
2. **Receipt token pattern** — mint exactly 1 token to prove the mint handler ran. The token name (`"RECEIPT"`) prevents minting arbitrary tokens.
3. **`dict.to_pairs(minted) == [Pair(name, qty)]`** — exact match ensures only one token type minted with exact quantity.
4. **`assets.policies(tx.mint)`** — returns list of PolicyIds present in the mint field.
5. **Multi-purpose validator** — same `validator` block contains both `mint` and `spend` handlers (same script hash).
6. **Transaction-level validation** — mint handler checks global invariant (output_total <= input_total) across ALL inputs/outputs.
7. **Spend handler delegation** — spend handler is a lightweight O(1) check that minting occurred, avoiding redundant per-input validation.

## When to Use

- Transaction must enforce global invariants across all inputs and outputs
- Multiple UTxOs from the same script spent in one transaction
- Need to avoid redundant validation logic running per-input
- Alternative to Withdraw Zero Trick when a staking validator isn't appropriate
