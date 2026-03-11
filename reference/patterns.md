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

## Contract Upgrade / Migration Pattern

**Problem:** Plutus scripts are immutable once deployed. If a vulnerability is
found or a feature needs adding, you cannot modify the on-chain script. Funds
locked at the old script address are stuck unless the validator has an upgrade
path.

**Solution:** Include a `Migrate` redeemer that allows an authorized party to
move funds from the old contract to a new one. The migration must be tightly
controlled — it's essentially a backdoor, so security is critical.

### Approach 1: Admin-Authorized Migration

The simplest pattern. An admin key (or multisig) can authorize moving funds to
any new script address. Fast to implement but centralizes trust.

```aiken
type VaultRedeemer {
  Withdraw       // Normal spend
  Migrate        // Move to new contract version
}

validator vault_v1(admin: ByteArray) {
  spend(datum: Option<Datum>, redeemer: VaultRedeemer, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      Withdraw -> {
        // Normal validation...
        list.has(tx.extra_signatories, d.owner)?
      }
      Migrate -> {
        // Only admin can trigger migration
        list.has(tx.extra_signatories, admin)?
      }
    }
  }
}
```

**Trade-off:** Simple, but admin has unilateral power over all locked funds.
Suitable for early-stage contracts with known, trusted operators.

### Approach 2: Destination-Locked Migration

Migration is restricted to a specific new script hash, baked in as a parameter
or stored in datum. The admin can trigger migration, but funds can only move
to the pre-approved destination.

```aiken
validator vault_v1(admin: ByteArray, upgrade_script_hash: ByteArray) {
  spend(datum: Option<Datum>, redeemer: VaultRedeemer, oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      Withdraw -> list.has(tx.extra_signatories, d.owner)?
      Migrate -> {
        // Admin must sign
        let admin_signed = list.has(tx.extra_signatories, admin)?

        // Funds must go to the approved new script address
        let new_script_addr = address.from_script(upgrade_script_hash)
        let funds_to_new_script =
          list.any(tx.outputs, fn(o) {
            o.address == new_script_addr
          })?

        admin_signed && funds_to_new_script
      }
    }
  }
}
```

**Trade-off:** Admin cannot steal funds — only redirect to the pre-approved
script. But the upgrade target must be known at V1 deployment time (or use
a reference input to look it up dynamically).

### Approach 3: Owner-Initiated Migration

Each user migrates their own funds. No admin key needed. The owner authorizes
the move and the validator ensures funds go to the new script with equivalent
datum.

```aiken
validator vault_v1(new_script_hash: ByteArray) {
  spend(datum: Option<Datum>, redeemer: VaultRedeemer, oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      Withdraw -> list.has(tx.extra_signatories, d.owner)?
      Migrate -> {
        // Owner must sign their own migration
        let owner_signed = list.has(tx.extra_signatories, d.owner)?

        // Funds go to new script
        let new_script_addr = address.from_script(new_script_hash)
        let migrated_output =
          list.any(tx.outputs, fn(o) {
            o.address == new_script_addr
          })?

        owner_signed && migrated_output
      }
    }
  }
}
```

**Trade-off:** No centralized trust — each user controls their own migration.
But requires every user to actively migrate, which may take time. Orphaned
UTxOs at the old script persist indefinitely.

### Approach 4: Reference Input for Upgrade Target

Uses a governance NFT in a reference input to point to the approved upgrade
script. Allows changing the upgrade target without redeploying V1.

```aiken
validator vault_v1(governance_nft_policy: ByteArray, governance_nft_name: ByteArray) {
  spend(datum: Option<Datum>, redeemer: VaultRedeemer, oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      Withdraw -> list.has(tx.extra_signatories, d.owner)?
      Migrate -> {
        let owner_signed = list.has(tx.extra_signatories, d.owner)?

        // Read upgrade target from governance reference input
        expect Some(gov_ref) =
          list.find(tx.reference_inputs, fn(i) {
            assets.quantity_of(i.output.value, governance_nft_policy, governance_nft_name) == 1
          })
        expect InlineDatum(raw) = gov_ref.output.datum
        expect upgrade_target: ByteArray = raw

        // Funds must go to the governance-approved script
        let new_script_addr = address.from_script(upgrade_target)
        let migrated =
          list.any(tx.outputs, fn(o) {
            o.address == new_script_addr
          })?

        owner_signed && migrated
      }
    }
  }
}
```

**Trade-off:** Most flexible — upgrade target can change without redeploying.
But requires governance NFT infrastructure and adds complexity.

### Migration Security Considerations

- **Always require authorization** — never allow unauthenticated migration
- **Validate the destination** — unbounded migration is indistinguishable from theft
- **Preserve user funds** — migration output value must be >= input value
- **Preserve datum** — if the new contract expects similar datum, validate it's carried over
- **Time-lock migrations** — consider requiring advance notice (e.g., governance vote period)
  before migration is enabled, so users can withdraw instead
- **One-way only** — V2 should not automatically trust V1 migration data; V2's mint handler
  should validate incoming migrated UTxOs independently
- **Test the migration path** — write explicit tests for the Migrate redeemer, including
  failure cases (wrong destination, missing signature, insufficient output value)

## Sorted Linked List (On-Chain Registry)

**Problem:** Need O(1) membership proofs for an on-chain set of registered items
(e.g., token policies, approved addresses, oracle feeds).

**Solution:** A sorted linked list where each node is a UTxO with an NFT marker
and inline datum containing `key`, `next`, and payload fields. The list is
ordered lexicographically by key.

**Confirmed working** — See [adavault/programmable-tokens](https://github.com/adavault/programmable-tokens) registry_mint.

```aiken
type RegistryNode {
  key: ByteArray,           // This node's key
  next: ByteArray,          // Next key in sorted order (#"" = end of list)
  // ... payload fields
}

// Origin (sentinel) node: key = #"", next = #"" (or first real key)
// Empty bytearray is always less than any non-empty bytearray
```

### Membership Proofs

```aiken
type Proof {
  Exists { node_idx: Int }       // Reference input[idx].key == query
  DoesNotExist { node_idx: Int } // covering.key < query < covering.next
}
```

- **Exists:** O(1) — point to the node via reference input index
- **DoesNotExist:** O(1) — point to the covering node that spans the gap

### Insert Operation

Find the covering node (where `covering.key < new_key < covering.next`),
spend it, and output two nodes:

```aiken
// Updated covering: only next pointer changes
let expected_covering = RegistryNode { ..covering, next: new_key }
expect updated_covering == expected_covering

// New node: inherits covering.next
expect new_node.key == new_key
expect new_node.next == covering.next
```

### Security Considerations

- **Count registry inputs:** During insert, verify exactly 1 registry node is
  spent. Otherwise an attacker can spend unrelated nodes alongside a valid insert.
- **Mint exactly 1 NFT:** `dict.to_pairs(minted) == [Pair(key, 1)]` prevents
  minting extra tokens or burning existing ones in the same transaction.
- **Guard spend with mint:** The spend validator can simply check
  `has_currency_symbol(tx.mint, registry_policy)` — the mint validator
  handles all complex validation.

## CIP-113 Programmable Token Coordination

**Problem:** Enforce custom transfer rules on every token movement while keeping
the spending validator cheap (it runs per-input).

**Solution:** A multi-validator architecture using withdraw-zero for transaction-level
coordination with pluggable transfer logic per token type.

**Confirmed working** — See [adavault/programmable-tokens](https://github.com/adavault/programmable-tokens).

### Architecture

```
programmable_logic_base (spend)     — O(1) check: is global in withdrawals?
programmable_logic_global (withdraw) — runs ONCE: sum inputs, check proofs, verify outputs
transfer_logic (withdraw)           — runs ONCE: token-specific rules (whitelist, limits)
registry (mint + spend)             — sorted linked list of registered token policies
```

All programmable tokens share a single payment credential. Ownership is
determined by per-holder stake credentials (VK or script). This enables
the global coordinator to process all tokens in one pass.

### Transfer Flow

1. Spend validator checks global coordinator is in withdrawals
2. Global coordinator sums authorized inputs (each stake cred must sign or be invoked)
3. For each non-ADA policy, consume a registry proof:
   - `TokenExists` → verify transfer logic is invoked, track as programmable
   - `TokenDoesNotExist` → covering node proves non-membership, skip
4. Verify outputs at programmable address contain >= all programmable tokens

### Key Design Decisions

- **`expect` for security-critical Bool checks:** When `TokenExists` proof is
  provided, the transfer logic MUST be in withdrawals. Use `expect has_key(...)`
  not `has_key(...)` as return value — prevents registered tokens from being
  silently treated as non-programmable.
- **Skip ADA in proof iteration:** `assets.to_dict(value)` includes ADA under
  policy `""`. Skip it before matching against proofs.
- **Balance invariant for third-party actions:** `total_output >= total_input`
  across ALL outputs at the programmable address, not per-pair. Seized tokens
  must stay in the system.
