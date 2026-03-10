# Design Patterns

Advanced architectural patterns for production Aiken contracts.
Based on [Anastasia Labs design patterns](https://github.com/Anastasia-Labs/aiken-design-patterns).

## Withdraw Zero Trick

**Problem:** When a transaction spends multiple UTxOs from the same script, the
validator runs once per input. This is wasteful if validation logic is identical
or can be batched.

**Solution:** Delegate validation to a withdrawal validator. The spend validator
only checks that a withdrawal from the staking script exists (O(1) check).
The withdrawal validator runs once per transaction and performs all validation.

**Confirmed working** — See [withdraw-zero.md](../examples/withdraw-zero.md).

```aiken
use cardano/address.{Credential, Script}

// Spend validator — lightweight check
validator batch_spend(staking_validator_hash: ByteArray) {
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    // Check withdrawal by iterating tx.withdrawals (Pairs<Credential, Lovelace>)
    list.any(tx.withdrawals, fn(w) {
      let Pair(cred, _amount) = w
      cred == Script(staking_validator_hash)
    })?
  }
}

// Withdrawal validator — runs ONCE, validates everything
validator batch_validator {
  withdraw(_redeemer: Data, _credential: Credential, tx: Transaction) {
    validate_batch(tx.inputs, tx.outputs)
  }
}
```

**Key:** `tx.withdrawals` is `List<Pair<Credential, Int>>`. Destructure with
`let Pair(cred, _amount) = w`. Credential constructors: `Script(hash)` and
`address.VerificationKey(hash)` — import from `cardano/address`.

**When to use:** Any validator that processes multiple UTxOs per transaction
(DEXes, batch operations, bulk claims).

## UTxO Indexing

**Problem:** Linking specific inputs to specific outputs without iterating.

**Solution:** Use redeemer to pass indices mapping inputs to outputs.

**Confirmed working** — See [utxo-indexer.md](../examples/utxo-indexer.md).

```aiken
type IndexedRedeemer {
  input_index: Int,
  output_index: Int,
}

validator indexed_validator {
  spend(datum: Option<Datum>, redeemer: IndexedRedeemer, oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    // Direct lookup — O(1) instead of searching
    expect Some(input) = list.at(tx.inputs, redeemer.input_index)
    expect Some(output) = list.at(tx.outputs, redeemer.output_index)

    // CRITICAL: Verify the index points to our actual input
    let correct_input = (input.output_reference == oref)?
    // Validate the input → output transition
    correct_input && validate_transition(d, input, output)
  }
}
```

**Key:** Always verify `input.output_reference == oref` — without this, an
attacker can point the index to a different input and bypass validation.

## Transaction Level Validator Minting Policy (TVMP)

**Problem:** Spend validators run per-input but you need transaction-level
validation (e.g., checking global invariants across all inputs/outputs).

**Solution:** Couple a minting policy with a spend validator. The spend
validator checks that minting occurred. The minting policy runs once and
validates the entire transaction.

```aiken
validator coupled {
  // Minting policy — runs once per tx, validates everything
  mint(_redeemer: Data, policy_id: PolicyId, tx: Transaction) {
    // Ensure mint is exactly 0 (we're not actually creating tokens)
    let minted = assets.tokens(tx.mint, policy_id)
    dict.is_empty(minted) || validate_transaction(tx)
  }

  // Spend validator — just checks minting policy ran
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    let policy_id = // own script hash
    list.any(
      assets.policies(tx.mint),
      fn(p) { p == policy_id }
    )
  }
}
```

## Validity Range Normalisation

**Problem:** Cardano validity ranges use complex interval types with
open/closed/positive-infinity/negative-infinity bounds. Raw pattern matching
is verbose and error-prone.

**Solution:** Normalise to a clean enum early in validation.

```aiken
type NormalisedRange {
  ClosedRange { from: Int, to: Int }
  FromNegInf { to: Int }
  ToPosInf { from: Int }
  Always
  InvalidRange
}

fn normalise(range: Interval<Int>) -> NormalisedRange {
  when (range.lower_bound.bound_type, range.upper_bound.bound_type) is {
    (Finite(lo), Finite(hi)) -> ClosedRange { from: lo, to: hi }
    (NegativeInfinity, Finite(hi)) -> FromNegInf { to: hi }
    (Finite(lo), PositiveInfinity) -> ToPosInf { from: lo }
    (NegativeInfinity, PositiveInfinity) -> Always
    _ -> InvalidRange
  }
}

// Usage
fn validate_timing(tx: Transaction, deadline: Int) -> Bool {
  when normalise(tx.validity_range) is {
    ToPosInf { from } -> from > deadline
    ClosedRange { from, .. } -> from > deadline
    _ -> False
  }
}
```

## Merkelized Validator

**Problem:** Complex validation logic exceeds transaction size limits or
execution budget.

**Solution:** Split validation into a main validator and one or more withdrawal
scripts that perform expensive sub-computations. The main validator delegates to
withdrawals via the withdraw-zero trick.

```aiken
// Main validator
validator main {
  spend(datum: Option<Datum>, redeemer: MainRedeemer, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      SimpleAction -> simple_check(d, tx)
      ComplexAction { withdrawal_hash } -> {
        // Delegate to withdrawal script
        pairs.has_key(tx.withdrawals, Script(withdrawal_hash))
      }
    }
  }
}

// Withdrawal script handles expensive computation
validator complex_check {
  withdraw(_redeemer: Data, _credential: Credential, tx: Transaction) {
    perform_expensive_validation(tx)
  }
}
```

## State Machine Pattern

**Problem:** Contract maintains state across multiple transactions.

**Solution:** Continuing output pattern — validator requires its output to go
back to the same script address with updated datum.

```aiken
type State {
  Initialised { owner: VerificationKeyHash }
  Active { owner: VerificationKeyHash, balance: Int }
  Closed
}

type Action {
  Activate { deposit: Int }
  AddFunds { amount: Int }
  Close
}

validator state_machine {
  spend(datum: Option<State>, redeemer: Action, oref: OutputReference, tx: Transaction) {
    expect Some(state) = datum
    expect Some(own_input) = transaction.find_input(tx.inputs, oref)
    let script_cred = own_input.output.address.payment_credential

    when (state, redeemer) is {
      (Initialised { owner }, Activate { deposit }) -> {
        // Must produce continuing output with Active state
        expect [continuing_output] =
          transaction.find_script_outputs(tx.outputs, script_cred)
        expect InlineDatum(raw) = continuing_output.datum
        expect new_state: State = raw

        new_state == Active { owner, balance: deposit } &&
        assets.lovelace_of(continuing_output.value) >= deposit &&
        list.has(tx.extra_signatories, owner)
      }
      (Active { owner, balance }, AddFunds { amount }) -> {
        expect [continuing_output] =
          transaction.find_script_outputs(tx.outputs, script_cred)
        expect InlineDatum(raw) = continuing_output.datum
        expect new_state: State = raw

        new_state == Active { owner, balance: balance + amount } &&
        assets.lovelace_of(continuing_output.value) >= balance + amount
      }
      (Active { owner, .. }, Close) -> {
        // No continuing output — funds returned to owner
        list.has(tx.extra_signatories, owner)
      }
      _ -> fail @"invalid state transition"
    }
  }
}
```

## Pool Restriction Pattern

For ADAvault-specific use: restricting delegation to specific pools.

```aiken
type VaultDatum {
  owner: VerificationKeyHash,
  allowed_pools: List<PoolId>,
}

// Check delegation certificate targets allowed pool
fn validate_delegation(tx: Transaction, allowed_pools: List<PoolId>) -> Bool {
  list.all(
    tx.certificates,
    fn(cert) {
      when cert is {
        DelegateCredential { delegate: DelegateBlockProduction { pool_id } } ->
          list.has(allowed_pools, pool_id)
        _ -> True  // non-delegation certs are fine
      }
    }
  )
}
```
