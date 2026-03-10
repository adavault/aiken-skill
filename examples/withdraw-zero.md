# Withdraw Zero Trick — Batch Validation

Delegates per-input validation to a withdrawal script that runs once per
transaction. The spend validator does an O(1) check for withdrawal presence.

This is the most important optimization pattern for production contracts that
process multiple UTxOs per transaction.

## Architecture

```
Without withdraw-zero:           With withdraw-zero:
  Input 1 → validate()             Input 1 → check withdrawal exists (O(1))
  Input 2 → validate()             Input 2 → check withdrawal exists (O(1))
  Input 3 → validate()             Input 3 → check withdrawal exists (O(1))
  (3x full validation)             Withdraw → validate_all() (runs ONCE)
```

## Validators

```aiken
// validators/withdraw_zero.ak

use aiken/collection/list
use cardano/address.{Credential, Script}
use cardano/assets
use cardano/transaction.{OutputReference, Transaction}

// Spend validator — lightweight delegation check
// Parameterized by the staking script hash it delegates to
validator batch_spend(staking_validator_hash: ByteArray) {
  spend(
    _datum: Option<Data>,
    _redeemer: Data,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    // Just check that the withdrawal validator was executed
    list.any(tx.withdrawals, fn(w) {
      let Pair(cred, _amount) = w
      cred == Script(staking_validator_hash)
    })?
  }
}

// Withdrawal validator — runs ONCE per transaction
// Validates a batch invariant: total outputs <= total inputs
validator batch_validator {
  withdraw(_redeemer: Data, _credential: Credential, tx: Transaction) {
    let input_lovelace =
      list.foldl(tx.inputs, 0, fn(input, acc) {
        acc + assets.lovelace_of(input.output.value)
      })
    let output_lovelace =
      list.foldl(tx.outputs, 0, fn(output, acc) {
        acc + assets.lovelace_of(output.value)
      })
    (output_lovelace <= input_lovelace)?
  }
}
```

## Tests

```aiken
// validators/withdraw_zero.ak (continued)

const mock_staking_hash =
  #"eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"

// --- Spend: Delegation Check Tests ---

test spend_with_withdrawal_present_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      // Construct a withdrawal entry: Pair(credential, amount)
      withdrawals: [Pair(Script(mock_staking_hash), 0)],
    }
  // Parameterized validator: pass staking_hash first
  batch_spend.spend(mock_staking_hash, None, Void, mock_oref(), tx)
}

test spend_without_withdrawal_fails() fail {
  batch_spend.spend(
    mock_staking_hash,
    None,
    Void,
    mock_oref(),
    transaction.placeholder,
  )
}

test spend_with_wrong_withdrawal_fails() fail {
  let wrong_hash =
    #"ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
  let tx =
    Transaction {
      ..transaction.placeholder,
      withdrawals: [Pair(Script(wrong_hash), 0)],
    }
  batch_spend.spend(mock_staking_hash, None, Void, mock_oref(), tx)
}

test spend_with_vkey_withdrawal_fails() fail {
  // VerificationKey credential should NOT match Script check
  let tx =
    Transaction {
      ..transaction.placeholder,
      withdrawals: [Pair(address.VerificationKey(mock_staking_hash), 0)],
    }
  batch_spend.spend(mock_staking_hash, None, Void, mock_oref(), tx)
}

// --- Withdraw: Batch Validation Tests ---

test batch_value_preserved_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(10_000_000), mock_input(5_000_000)],
      outputs: [mock_output(15_000_000)],
    }
  // Withdraw handler: pass redeemer, credential, tx
  batch_validator.withdraw(Void, Script(mock_staking_hash), tx)
}

test batch_value_creation_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input(5_000_000)],
      outputs: [mock_output(10_000_000)],
    }
  batch_validator.withdraw(Void, Script(mock_staking_hash), tx)
}
```

## Key Concepts Demonstrated

1. **`withdraw` handler** — new handler type: `withdraw(redeemer, credential, tx)`
2. **`tx.withdrawals`** — `Pairs<Credential, Lovelace>` (i.e., `List<Pair<Credential, Int>>`)
3. **`Pair` destructuring in closures** — `let Pair(cred, _amount) = w`
4. **`Script(hash)` credential constructor** — imported from `cardano/address`
5. **`address.VerificationKey(hash)`** — the other credential variant
6. **`list.foldl`** — accumulator pattern for summing values
7. **Withdraw amount = 0** — the script executes but no ADA moves (the "zero" trick)
8. **Separate validators** — spend and withdraw are different validators (different scripts)

## When to Use

- DEX order batching (many swaps in one transaction)
- Bulk token distributions
- Multi-UTxO claims
- Any validator processing >2 script inputs per transaction

The cost savings are proportional to the number of inputs: N inputs × O(validation) becomes
N × O(1) + 1 × O(validation).
