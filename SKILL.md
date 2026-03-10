---
name: aiken-smart-contract
description: >
  Write, test, and debug Aiken smart contracts for Cardano.
  Use when writing validators, minting policies, or any on-chain Plutus code.
  Triggers on: Aiken, validator, smart contract, Cardano on-chain, Plutus,
  minting policy, spend validator, datum, redeemer, plutus.json, blueprint.
  Covers language syntax, validator patterns, property-based testing,
  security best practices, and stdlib usage.
user-invocable: true
---

# Aiken Smart Contract Development

You are an expert Aiken smart contract developer for Cardano. Aiken is a pure
functional language that compiles to UPLC (Untyped Plutus Lambda Calculus)
targeting Plutus V3.

## Core Principles

1. **Validators are predicates** — they return Bool. True = authorize, False = reject.
2. **No side effects** — pure functional, no mutation, no loops (use recursion).
3. **Local reasoning only** — eUTxO model means each validator sees only its own context.
4. **Test everything** — Aiken's test runner uses the real CEK machine. Tests are production-accurate.
5. **Security first** — understand double satisfaction, datum hijacking, and other eUTxO-specific attacks.

## Decision Tree

When asked to write a smart contract:

1. **Identify the validator purpose**: spend, mint, withdraw, publish, vote, propose
2. **Design the datum and redeemer types** before writing logic
3. **Write the validator** using the correct handler signature
4. **Write tests immediately** — unit tests first, then property-based
5. **Review for security** — check the security patterns in [security.md](reference/security.md)
6. **Build and verify** — `aiken build` must succeed, `aiken check` must pass

## Handler Signatures (Aiken v1.1+)

```aiken
// Spending validator — most common
validator my_validator {
  spend(
    datum: Option<MyDatum>,    // Always Option — may be missing
    redeemer: MyRedeemer,
    output_reference: OutputReference,
    transaction: Transaction,
  ) {
    todo
  }
}

// Minting policy
validator my_policy {
  mint(
    redeemer: MyRedeemer,
    policy_id: PolicyId,
    transaction: Transaction,
  ) {
    todo
  }
}

// Withdrawal validator
validator my_withdrawal {
  withdraw(
    redeemer: MyRedeemer,
    credential: Credential,
    transaction: Transaction,
  ) {
    todo
  }
}

// Certificate publishing
validator my_cert {
  publish(
    redeemer: MyRedeemer,
    certificate: Certificate,
    transaction: Transaction,
  ) {
    todo
  }
}

// Governance voting
validator my_vote {
  vote(
    redeemer: MyRedeemer,
    voter: Voter,
    transaction: Transaction,
  ) {
    todo
  }
}

// Governance proposal
validator my_proposal {
  propose(
    redeemer: MyRedeemer,
    proposal_procedure: ProposalProcedure,
    transaction: Transaction,
  ) {
    todo
  }
}

// Fallback for unhandled purposes
validator my_fallback {
  else(script_context: ScriptContext) {
    todo
  }
}
```

## Multi-Purpose Validators

A single validator can handle multiple purposes (same script hash):

```aiken
validator token_lock {
  mint(redeemer: MintAction, policy_id: PolicyId, tx: Transaction) {
    // Minting logic
    todo
  }

  spend(datum: Option<LockDatum>, redeemer: SpendAction, _oref: OutputReference, tx: Transaction) {
    // Spending logic — require token burn
    todo
  }
}
```

## Parameterized Validators

Validators can take compile-time parameters:

```aiken
validator my_validator(owner: VerificationKeyHash, deadline: POSIXTime) {
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    let must_be_signed = list.has(tx.extra_signatories, owner)
    let must_be_after_deadline =
      interval.is_entirely_after(tx.validity_range, deadline)
    must_be_signed && must_be_after_deadline
  }
}
```

Parameters are applied when building the script address from the blueprint.

## Key Language Idioms

```aiken
// Pattern matching (exhaustive)
when redeemer is {
  Lock -> handle_lock(datum, tx)
  Unlock -> handle_unlock(datum, tx)
}

// expect — unsafe downcast, fails if wrong
expect Some(datum) = datum_opt
expect InlineDatum(raw) = output.datum
expect typed_datum: MyDatum = raw

// Pipe operator — chain operations
tx.outputs
  |> list.filter(fn(o) { o.address == script_address })
  |> list.any(fn(o) { check_output(o) })

// Trace for debugging (removed in production with --trace-level silent)
trace @"checking signature"
let signed = list.has(tx.extra_signatories, owner)
// ? operator — postfix, traces expression name if False
list.has(tx.extra_signatories, owner)?
// Produces trace: "list.has(tx.extra_signatories, owner) ? False"
// NOTE: ? is postfix only (expr?), not infix (expr ? "msg")
```

## Common Validation Patterns

```aiken
// Check transaction signed by key
list.has(tx.extra_signatories, owner_pkh)

// Check time (must be after deadline)
interval.is_entirely_after(tx.validity_range, deadline)

// Check time (must be before deadline)
interval.is_entirely_before(tx.validity_range, deadline)

// Find own input
expect Some(own_input) = transaction.find_input(tx.inputs, oref)

// Find outputs to a script address
transaction.find_script_outputs(tx.outputs, script_hash)

// Check NFT exists in value
assets.quantity_of(value, policy_id, asset_name) == 1

// Merge values
assets.merge(value_a, value_b)

// Check lovelace amount
assets.lovelace_of(output.value) >= min_amount
```

## Testing

Always write tests alongside validators. See [testing.md](reference/testing.md) for full details.

```aiken
// Unit test
test must_be_signed() {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_signer],
  }
  my_validator.spend(Some(datum), redeemer, mock_oref, tx)
}

// Expected failure
test must_fail_without_signature() fail {
  my_validator.spend(Some(datum), redeemer, mock_oref, transaction.placeholder)
}

// Parameterized validator — pass params first, then handler args
// validator gift_card(utxo_ref: OutputReference, token_name: ByteArray) { mint(...) }
// Call as: gift_card.mint(utxo_ref, token_name, redeemer, policy_id, tx)

// Property-based test
test prop_any_signer_works(signer via fuzz.bytearray_fixed(28)) {
  let datum = MyDatum { owner: signer }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [signer],
  }
  my_validator.spend(Some(datum), Unlock, mock_oref, tx)
}
```

## CLI Workflow

```bash
aiken new my-project          # Scaffold new project
aiken build                   # Compile, generate plutus.json
aiken check                   # Typecheck + run all tests
aiken check -m "test_name"    # Run specific test
aiken fmt                     # Format code
aiken docs                    # Generate documentation
aiken blueprint address       # Generate script address from blueprint
```

## Reference Material

For detailed information, consult:

- [Language reference](reference/language.md) — types, syntax, modules, encoding
- [Validator patterns](reference/validators.md) — common validator architectures
- [Testing guide](reference/testing.md) — unit, property-based, scenario testing
- [Security patterns](reference/security.md) — eUTxO attack vectors and mitigations
- [Standard library](reference/stdlib.md) — key modules and functions
- [Design patterns](reference/patterns.md) — withdraw-zero trick, UTxO indexers, etc.

## Examples

Working examples with full test suites (all compiler-validated):

**Phase 1 — Core Patterns:**
- [Hello World](examples/hello-world.md) — simplest spend validator
- [Vesting](examples/vesting.md) — time-locked spending with dual authorization
- [Gift Card](examples/gift-card.md) — mint+spend dual handler with one-shot NFT

**Phase 2 — Security & Design Patterns:**
- [Multi-Sig](examples/multi-sig.md) — M-of-N threshold signatures
- [State Machine](examples/state-machine.md) — continuing output pattern with state transitions
- [NFT Vault](examples/nft-vault.md) — datum hijacking prevention with NFT authentication

**Phase 3 — Advanced Optimization Patterns:**
- [Withdraw Zero](examples/withdraw-zero.md) — batch validation via withdrawal delegation
- [UTxO Indexer](examples/utxo-indexer.md) — O(1) input-output linking with redeemer indices
- [Tagged Output](examples/tagged-output.md) — double satisfaction prevention with crypto hashing
- [Validity Range](examples/validity-range.md) — interval normalisation for time-based validation
- [TVMP](examples/tvmp.md) — transaction-level validation via minting policy receipt tokens
- [Pool Restriction](examples/pool-restriction.md) — certificate-based delegation control with pool whitelist
