# Standard Library Reference

## Key Modules

### cardano/transaction

The primary type validators work with.

```aiken
type Transaction {
  inputs: List<Input>,
  reference_inputs: List<Input>,
  outputs: List<Output>,
  fee: Lovelace,
  mint: Value,
  certificates: List<Certificate>,
  withdrawals: Pairs<Credential, Lovelace>,
  validity_range: ValidityRange,
  extra_signatories: List<VerificationKeyHash>,
  redeemers: Pairs<ScriptPurpose, Redeemer>,
  datums: Dict<DataHash, Data>,
  id: TransactionId,
  votes: Dict<Voter, Dict<GovernanceActionId, Vote>>,
  proposal_procedures: List<ProposalProcedure>,
  current_treasury_amount: Option<Lovelace>,
  treasury_donation: Option<Lovelace>,
}
```

**Key functions:**
```aiken
// Find the input being validated
transaction.find_input(tx.inputs, oref) -> Option<Input>

// Find outputs to a script
transaction.find_script_outputs(tx.outputs, script_credential) -> List<Output>

// Find datum by hash
transaction.find_datum(tx.datums, hash) -> Option<Data>

// Empty transaction for testing
transaction.placeholder -> Transaction
```

### cardano/assets

Value manipulation — ADA and native tokens.

```aiken
// Create values
assets.from_lovelace(2_000_000) -> Value
assets.from_asset(policy_id, asset_name, quantity) -> Value
assets.zero -> Value

// Query values
assets.lovelace_of(value) -> Int
assets.quantity_of(value, policy_id, asset_name) -> Int
assets.tokens(value, policy_id) -> Dict<AssetName, Int>

// Combine values
assets.merge(a, b) -> Value
assets.add(value, policy_id, asset_name, quantity) -> Value
assets.negate(value) -> Value

// Inspect
assets.flatten(value) -> List<(PolicyId, AssetName, Int)>
assets.without_lovelace(value) -> Value
assets.policies(value) -> List<PolicyId>

// Check NFT
assets.quantity_of(value, policy, name) == 1
```

### cardano/address

Address construction and inspection.

```aiken
// Create addresses
address.from_verification_key(pkh) -> Address
address.from_script(script_hash) -> Address

// With staking
address.from_verification_key(pkh)
  |> address.with_delegation_key(stake_pkh)

address.from_script(script_hash)
  |> address.with_delegation_script(stake_script_hash)

// Inspect
type Credential {
  VerificationKey(VerificationKeyHash)
  Script(ScriptHash)
}
```

### aiken/interval

Time/slot range operations. Cardano uses validity ranges, not timestamps.

```aiken
// Construct intervals
interval.after(time) -> Interval<Int>          // [time, +inf)
interval.before(time) -> Interval<Int>         // (-inf, time]
interval.between(start, end) -> Interval<Int>  // [start, end]
interval.entirely_after(time) -> Interval<Int>  // (time, +inf)
interval.entirely_before(time) -> Interval<Int> // (-inf, time)

// Check validity range
interval.is_entirely_after(tx.validity_range, deadline) -> Bool
interval.is_entirely_before(tx.validity_range, deadline) -> Bool
interval.contains(tx.validity_range, time) -> Bool
interval.is_empty(interval) -> Bool

// Combine
interval.hull(a, b) -> Interval       // smallest containing both
interval.intersection(a, b) -> Interval
```

**Important:** `is_entirely_after(range, time)` means the ENTIRE range is after
`time`. This requires the transaction to set `invalid_before` (lower bound) to
at least `time + 1`. The wallet/off-chain code must set this.

### aiken/collection/list

Functional list operations. Most commonly used module.

```aiken
// Query
list.has(xs, elem) -> Bool
list.any(xs, predicate) -> Bool
list.all(xs, predicate) -> Bool
list.find(xs, predicate) -> Option<a>
list.at(xs, index) -> Option<a>
list.length(xs) -> Int
list.is_empty(xs) -> Bool

// Transform
list.map(xs, fn) -> List<b>
list.filter(xs, predicate) -> List<a>
list.flat_map(xs, fn) -> List<b>
list.indexed_map(xs, fn(index, elem)) -> List<b>

// Fold
list.foldl(xs, initial, fn(elem, acc)) -> b
list.foldr(xs, initial, fn(elem, acc)) -> b
list.reduce(xs, fn(a, a)) -> Option<a>

// Sort
list.sort(xs, compare_fn) -> List<a>
list.unique(xs) -> List<a>

// Combine
list.zip(xs, ys) -> List<Pair<a, b>>
list.concat(xs, ys) -> List<a>

// Split
list.head(xs) -> Option<a>
list.tail(xs) -> Option<List<a>>
list.partition(xs, predicate) -> (List<a>, List<a>)
list.span(xs, predicate) -> (List<a>, List<a>)
```

### aiken/collection/dict

Ordered key-value store (keys must be comparable).

```aiken
// Create
dict.from_pairs(pairs) -> Dict<k, v>
dict.empty -> Dict<k, v>

// Query
dict.get(dict, key) -> Option<v>
dict.has_key(dict, key) -> Bool
dict.size(dict) -> Int
dict.keys(dict) -> List<k>
dict.values(dict) -> List<v>
dict.to_pairs(dict) -> List<Pair<k, v>>

// Modify
dict.insert(dict, key, value, merge_fn) -> Dict<k, v>
dict.delete(dict, key) -> Dict<k, v>
dict.union(a, b, merge_fn) -> Dict<k, v>

// Transform
dict.map(dict, fn(key, value)) -> Dict<k, v2>
dict.filter(dict, fn(key, value)) -> Dict<k, v>
dict.foldl(dict, initial, fn(key, value, acc)) -> a
```

### aiken/crypto

Cryptographic operations and hash types.

```aiken
// Hash types
type Hash<alg, a> = ByteArray

type Blake2b_224   // 28 bytes — used for key hashes, script hashes
type Blake2b_256   // 32 bytes — used for transaction IDs, datum hashes
type Sha2_256
type Sha3_256
type Keccak_256

// Common type aliases
type VerificationKeyHash = Hash<Blake2b_224, VerificationKey>
type ScriptHash = Hash<Blake2b_224, Script>
type TransactionId = Hash<Blake2b_256, Transaction>
type DataHash = Hash<Blake2b_256, Data>

// Signature verification
crypto.verify_ed25519_signature(key, msg, sig) -> Bool
crypto.verify_ecdsa_secp256k1_signature(key, msg, sig) -> Bool
crypto.verify_schnorr_secp256k1_signature(key, msg, sig) -> Bool
```

### aiken/math/rational

Precise fractional arithmetic (important for financial calculations).

```aiken
use aiken/math/rational

// Create
rational.new(numerator, denominator) -> Option<Rational>
rational.from_int(n) -> Rational

// Arithmetic
rational.add(a, b) -> Rational
rational.sub(a, b) -> Rational
rational.mul(a, b) -> Rational
rational.div(a, b) -> Option<Rational>
rational.negate(r) -> Rational
rational.abs(r) -> Rational

// Compare
rational.compare(a, b) -> Ordering

// Convert
rational.truncate(r) -> Int
rational.ceil(r) -> Int
rational.floor(r) -> Int
rational.round(r) -> Int
```

### aiken/crypto

Cryptographic operations. **Confirmed working** — used in tagged output pattern.

```aiken
use aiken/crypto

// Hash any ByteArray
crypto.blake2b_256(bytes) -> ByteArray     // 32 bytes output
crypto.blake2b_224(bytes) -> ByteArray     // 28 bytes output
crypto.sha2_256(bytes) -> ByteArray
crypto.sha3_256(bytes) -> ByteArray
crypto.keccak_256(bytes) -> ByteArray

// Signature verification
crypto.verify_ed25519_signature(key, msg, sig) -> Bool

// Common pattern: unique tag from output reference
fn tag_from_oref(oref: OutputReference) -> ByteArray {
  crypto.blake2b_256(cbor.serialise(oref))
}
```

### aiken/cbor

CBOR encoding. **Must be explicitly imported** — `use aiken/cbor`.

```aiken
use aiken/cbor

// Inspect any value as CBOR diagnostic notation
trace cbor.diagnostic(my_datum)

// Serialize ANY Aiken value to ByteArray (confirmed working)
cbor.serialise(value) -> ByteArray

// Common use: serialize for hashing
crypto.blake2b_256(cbor.serialise(my_oref))
```

### cardano/governance

Governance types for Conway-era voting, proposals, and DRep operations.
**Confirmed working** — used in governance vote, publish, and propose examples.

```aiken
use cardano/governance.{GovernanceActionId, StakePool, Vote, Voter,
  ProposalProcedure, GovernanceAction, TreasuryWithdrawal, NicePoll}

// Voter types
type Voter {
  ConstitutionalCommitteeMember(Credential)
  DelegateRepresentative(Credential)
  StakePool(VerificationKeyHash)
}

// Vote options
type Vote { No, Yes, Abstain }
// Access as: governance.Yes, governance.No, governance.Abstain

// Governance action reference
type GovernanceActionId {
  transaction: TransactionId,
  proposal_procedure: Int,
}

// Proposal structure
type ProposalProcedure {
  deposit: Lovelace,
  return_address: Credential,  // NOT Address — use VerificationKey(hash)
  governance_action: GovernanceAction,
}

// Governance action types
type GovernanceAction {
  ProtocolParameters { ancestor, new_parameters, guardrails }
  HardFork { ancestor, new_version }
  TreasuryWithdrawal { beneficiaries: Pairs<Credential, Lovelace>, guardrails: Option<ScriptHash> }
  NoConfidence { ancestor }
  ConstitutionalCommittee { ancestor, evicted_members, added_members, quorum }
  NewConstitution { ancestor, constitution }
  NicePoll  // simplest action, no fields
}

// Transaction fields:
tx.votes: Pairs<Voter, Pairs<GovernanceActionId, Vote>>
tx.proposal_procedures: List<ProposalProcedure>
```

**Key gotchas:**
- `ProtocolParametersUpdate` is an opaque type — cannot be constructed in tests
- `return_address` on `ProposalProcedure` is `Credential`, not `Address`
- Vote constructors are qualified: `governance.Yes` not just `Yes`
