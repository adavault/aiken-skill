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

**Solution:** Couple a minting policy with a spend validator. The mint handler
validates the entire transaction and mints a "receipt" token. The spend handler
checks that the receipt was minted (policy present).

**Confirmed working** — See [tvmp.md](../examples/tvmp.md).

**CRITICAL:** `assets.from_asset(policy, name, 0)` normalises away — zero-quantity
assets do NOT appear in `assets.policies()`. You must mint an actual token
(quantity >= 1) for the policy to be detectable. Use receipt tokens, not zero-quantity.

```aiken
use aiken/collection/dict
use cardano/assets.{PolicyId}

const receipt_name = "RECEIPT"

validator tvmp_vault {
  // Minting policy — runs once per tx, validates everything
  // Mints exactly 1 receipt token to signal validation passed
  mint(_redeemer: Data, policy_id: PolicyId, tx: Transaction) {
    let minted = assets.tokens(tx.mint, policy_id)
    let mints_receipt =
      (dict.to_pairs(minted) == [Pair(receipt_name, 1)])?

    // Transaction-level validation here
    mints_receipt && validate_transaction(tx)
  }

  // Spend validator — just checks minting policy ran
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    let own_policy = #"aaaaaa..."  // own script hash
    list.any(
      assets.policies(tx.mint),
      fn(p) { p == own_policy }
    )?
  }
}
```

**Key:** `dict.to_pairs(minted) == [Pair(name, qty)]` ensures exactly one token
type minted with the exact quantity. This prevents minting arbitrary tokens.

## Validity Range Normalisation

**Problem:** Cardano validity ranges use complex interval types with
open/closed/positive-infinity/negative-infinity bounds. Raw pattern matching
is verbose and error-prone.

**Solution:** Normalise to a clean enum early in validation.

**Confirmed working** — See [validity-range.md](../examples/validity-range.md).

**IMPORTANT:** `Interval` is NOT generic in Aiken stdlib — use `Interval` not
`Interval<Int>`. The latter causes a "wrong number of type parameters" error.

```aiken
use aiken/interval.{Finite, Interval, IntervalBound, NegativeInfinity,
  PositiveInfinity}

type NormalisedRange {
  ClosedRange { from: Int, to: Int }
  FromNegInf { to: Int }
  ToPosInf { from: Int }
  Always
}

fn normalise(range: Interval) -> NormalisedRange {
  when (range.lower_bound.bound_type, range.upper_bound.bound_type) is {
    (Finite(lo), Finite(hi)) -> ClosedRange { from: lo, to: hi }
    (NegativeInfinity, Finite(hi)) -> FromNegInf { to: hi }
    (Finite(lo), PositiveInfinity) -> ToPosInf { from: lo }
    (NegativeInfinity, PositiveInfinity) -> Always
    _ -> fail @"invalid range bounds"
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

Restricting delegation to specific pools. Directly relevant to ADAvault vaults.

**Confirmed working** — See [pool-restriction.md](../examples/pool-restriction.md).

**CRITICAL:** Must check ALL certificate types that involve pool delegation —
not just `DelegateCredential` but also `RegisterAndDelegateCredential`. Both
support `DelegateBlockProduction` and `DelegateBoth` delegate variants.
The field name is `stake_pool` (not `pool_id`).

```aiken
use cardano/certificate.{
  DelegateBoth, DelegateBlockProduction, DelegateCredential,
  RegisterAndDelegateCredential,
}

fn validate_delegation(
  certs: List<Certificate>,
  allowed_pools: List<PoolId>,
) -> Bool {
  list.all(
    certs,
    fn(cert) {
      when cert is {
        DelegateCredential { delegate, .. } ->
          when delegate is {
            DelegateBlockProduction { stake_pool } ->
              list.has(allowed_pools, stake_pool)?
            DelegateBoth { stake_pool, .. } ->
              list.has(allowed_pools, stake_pool)?
            _ -> True  // DelegateVote only — no pool
          }
        RegisterAndDelegateCredential { delegate, .. } ->
          when delegate is {
            DelegateBlockProduction { stake_pool } ->
              list.has(allowed_pools, stake_pool)?
            DelegateBoth { stake_pool, .. } ->
              list.has(allowed_pools, stake_pool)?
            _ -> True
          }
        _ -> True  // non-delegation certs are fine
      }
    },
  )
}
```

**Types used in validator signatures must be `pub`** — Aiken enforces that
types referenced in validator handler parameters are publicly accessible.
