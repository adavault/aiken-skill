# Gotchas & Hard-Won Learnings

Compiler findings and pitfalls discovered through iterative testing against
Aiken v1.1.21 with stdlib v3.0.0. Every item here was confirmed by
compilation error or unexpected runtime behavior.

## Type System

### Types in validator signatures must be `pub`

If a custom type is used in a validator handler's parameter list, it must be
declared `pub`. Otherwise the compiler raises a `private_leak` error.

```aiken
// BAD — compiler error: private_leak
type MyDatum { owner: ByteArray }

validator my_validator {
  spend(datum: Option<MyDatum>, ...) { ... }
}

// GOOD
pub type MyDatum { owner: ByteArray }
```

This applies to all types referenced in handler signatures — datum types,
redeemer types, and any types they contain.

### `Interval` is not generic

Use `Interval` not `Interval<Int>`. The stdlib type takes no type parameters.

```aiken
// BAD — "wrong number of type parameters" error
fn normalise(range: Interval<Int>) -> NormalisedRange { ... }

// GOOD
fn normalise(range: Interval) -> NormalisedRange { ... }
```

### `RegisterCredential` has uninhabitable `Never` type deposit

The `RegisterCredential` certificate variant has a deposit field typed as
`Never` — it cannot be constructed in tests. Use `RegisterAndDelegateCredential`
or other certificate types instead when testing registration scenarios.

### `ProtocolParametersUpdate` is opaque

Cannot be constructed in tests. When testing governance propose handlers that
match on `ProtocolParameters`, use `NicePoll` or `TreasuryWithdrawal` as
governance actions instead.

### `return_address` on `ProposalProcedure` is `Credential`, not `Address`

```aiken
// BAD — type mismatch
ProposalProcedure {
  return_address: address.from_verification_key(key),  // This is Address
  ...
}

// GOOD
ProposalProcedure {
  return_address: VerificationKey(key),  // This is Credential
  ...
}
```

## Value & Assets

### `assets.from_asset(policy, name, 0)` normalises away

Zero-quantity assets are removed during value normalisation. This means:
- They don't appear in `assets.policies(value)`
- `assets.quantity_of(value, policy, name)` returns 0
- You cannot use zero-quantity minting to signal "policy was present"

This is critical for the TVMP pattern — you must mint an actual receipt token
(quantity >= 1), not zero-quantity.

```aiken
// BAD — policy disappears from tx.mint
// Minting 0 tokens does NOT make the policy appear in assets.policies()

// GOOD — mint a receipt token with quantity 1
let minted = assets.tokens(tx.mint, policy_id)
dict.to_pairs(minted) == [Pair("RECEIPT", 1)]
```

### Build multi-asset values with pipe

Use `assets.from_lovelace() |> assets.add()` to build values containing
native tokens in tests:

```aiken
let value = assets.from_lovelace(2_000_000)
  |> assets.add(policy_id, token_name, 1)
```

## Certificate Types

### Field name is `stake_pool`, not `pool_id`

The `DelegateBlockProduction` and `DelegateBoth` delegate variants use
`stake_pool` as the field name:

```aiken
// BAD — no such field
DelegateBlockProduction { pool_id } -> ...

// GOOD
DelegateBlockProduction { stake_pool } -> ...
```

### Must check both `DelegateCredential` and `RegisterAndDelegateCredential`

Pool delegation can occur via either certificate type. If you only check
`DelegateCredential`, an attacker can delegate via `RegisterAndDelegateCredential`
to bypass your pool whitelist.

Similarly, `DelegateBoth` must be checked alongside `DelegateBlockProduction` —
both carry a `stake_pool` field.

## Governance Types

### Vote constructors are qualified

Access as `governance.Yes`, `governance.No`, `governance.Abstain` — not
unqualified `Yes`/`No`/`Abstain`.

```aiken
use cardano/governance

// BAD — unresolved constructors
let vote = Yes

// GOOD
let vote = governance.Yes
```

### `tx.votes` is nested pairs

```aiken
// Type: Pairs<Voter, Pairs<GovernanceActionId, Vote>>
tx.votes
```

Not a flat list — it's pairs of (voter, pairs of (action, vote)).

## Testing Patterns

### Parameterized validators pass params first in tests

When calling a parameterized validator in tests, parameters come before the
handler arguments:

```aiken
// validator gift_card(utxo_ref: OutputReference, token_name: ByteArray) {
//   mint(redeemer, policy_id, tx) { ... }
// }

// In test:
gift_card.mint(utxo_ref, token_name, redeemer, policy_id, tx)
//             ^^^^^^^^  ^^^^^^^^^^  ← params first
//                                   ^^^^^^^^  ^^^^^^^^^  ^^  ← then handler args
```

### `transaction.placeholder` for test transactions

Use spread syntax to override only the fields you care about:

```aiken
let tx = Transaction {
  ..transaction.placeholder,
  extra_signatories: [signer],
  validity_range: interval.after(time),
}
```

### `InlineDatum` auto-coerces custom types

When constructing test inputs/outputs, Aiken automatically coerces custom
types to `Data` inside `InlineDatum`:

```aiken
// This works — no explicit cast needed
datum: InlineDatum(MyCustomType { field: value })
```

When reading back in a validator, cast with `expect`:

```aiken
expect InlineDatum(raw) = output.datum
expect typed: MyCustomType = raw
```

### `fail` tests with property-based testing

Property tests marked `fail` expect failure for ALL generated values. If the
fuzzer can generate a value that makes the test pass (e.g., generating the
same key as the datum owner), the property test itself fails. With 28 bytes
of entropy (2^224), collisions are astronomically unlikely but theoretically
possible.

### `fuzz.both()` for multiple fuzz parameters

Combine two fuzzers into a tuple:

```aiken
test prop_test(params via fuzz.both(fuzz.bytearray_fixed(28), fuzz.int())) {
  let (key, amount) = params
  ...
}
```

### Mock output references need 32-byte transaction IDs

```aiken
fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    //               ^^ 64 hex chars = 32 bytes (Blake2b_256)
    output_index: 0,
  }
}
```

### Mock key hashes are 28 bytes (not 32)

Verification key hashes and script hashes use Blake2b_224 = 28 bytes = 56 hex chars:

```aiken
const mock_owner =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"
//  ^^ 56 hex chars = 28 bytes
```

## Import Patterns

### `cbor` must be explicitly imported

`use aiken/cbor` is required before using `cbor.serialise()` or
`cbor.diagnostic()`. It's not available by default.

### Common import sets by pattern

```aiken
// Basic spend validator
use aiken/collection/list
use cardano/assets
use cardano/transaction.{OutputReference, Transaction}

// With time validation
use aiken/interval

// With continuing outputs (state machine)
use cardano/transaction.{InlineDatum, Input, Output, OutputReference, Transaction}
use cardano/address

// With native tokens in tests
use cardano/assets  // assets.from_lovelace(), assets.add()

// With governance
use cardano/governance.{GovernanceActionId, Vote, Voter}

// With certificates
use cardano/certificate.{DelegateBlockProduction, DelegateBoth,
  DelegateCredential, RegisterAndDelegateCredential}

// With hashing
use aiken/cbor
use aiken/crypto

// With property tests
use aiken/fuzz
```

## Compiler Behavior

### Unused imports cause warnings

Aiken warns on unused imports. Clean them up before finalizing — they won't
fail the build but they add noise.

### `?` is postfix only

The trace-on-false operator is `expr?` — postfix. There is no infix form
`expr ? "message"`.

```aiken
// GOOD
list.has(tx.extra_signatories, owner)?

// BAD — not valid syntax
list.has(tx.extra_signatories, owner) ? "must be signed by owner"
```

### `and { }` block for multi-condition checks

Use `and { }` with `?` on each condition for clear error traces:

```aiken
and {
  (new_datum.owner == old_datum.owner)?,
  (new_datum.beneficiary == old_datum.beneficiary)?,
  (new_datum.deadline == expected_deadline)?,
}
```

Each condition traces independently on failure.

## Off-Chain (MeshJS) Pitfalls

These gotchas apply when writing TypeScript code that builds transactions
for Aiken validators using MeshJS v1.9.x-beta.

### `"JSON"` format required for all Plutus data

The default Mesh type format for datum, redeemer, and script parameters
causes "Cannot convert undefined to a BigInt" errors during tx serialization.
Always pass `"JSON"` as the format argument:

```typescript
// BAD — default Mesh format, BigInt serialization errors
txBuilder.txOutInlineDatumValue(datum);
txBuilder.mintRedeemerValue(redeemer);
blueprint.paramScript(code, params);

// GOOD — JSON format
txBuilder.txOutInlineDatumValue(datum, "JSON");
txBuilder.mintRedeemerValue(redeemer, "JSON");
blueprint.paramScript(code, params, "JSON");
```

### `constructor` not `alternative` in JSON format

The JSON format parser checks for a `constructor` key. Using `alternative`
(which is the Mesh type key) produces "Malformed Plutus data json":

```typescript
// BAD — "Malformed Plutus data json"
{ alternative: 0, fields: [...] }

// GOOD
{ constructor: 0, fields: [...] }
```

### `complete()` does not sign

`MeshTxBuilder.complete()` only builds the transaction body. You must call
`completeSigning()` separately — otherwise submission fails with "Some
signatures are missing":

```typescript
await txBuilder.signingKey(skey).complete();
txBuilder.completeSigning();  // ← REQUIRED
const signedTx = txBuilder.txHex;
```

### Collateral must be pure ADA

UTxOs containing native tokens cannot serve as collateral. Find a UTxO
that contains only lovelace:

```typescript
const collateral = utxos.find((u) =>
  u.output.amount.length === 1 && u.output.amount[0].unit === "lovelace"
);
```

### Enterprise vs base address mismatch

`AppWallet.getPaymentAddress()` returns a base address (with stake key).
If the funded wallet uses an enterprise address, UTxO lookups fail silently
(different address = no UTxOs found). Use `getEnterpriseAddress()`.

### Node.js 20 WebSocket polyfill

`OgmiosProvider` uses browser WebSocket API. Node.js 20 lacks it:

```typescript
import WebSocket from "ws";
(globalThis as any).WebSocket = WebSocket;
```

### `SpendingBlueprint` requires `noParamScript()` for non-parameterized validators

For spend validators without parameters, use `noParamScript()` — not `paramScript()`
with empty params:

```typescript
// Non-parameterized
const blueprint = new SpendingBlueprint("V3", 0, "");
blueprint.noParamScript(compiledCode);

// Parameterized
const blueprint = new SpendingBlueprint("V3", 0, "");
blueprint.paramScript(compiledCode, [{ bytes: ownerHash }], "JSON");
```

### Kupo `--match` patterns cannot change on existing DB

Changing `--match` arguments on a Kupo instance with an existing database causes
`ConflictingOptionsException`. Two options:

1. **HTTP API** (no downtime): `PUT /v1/patterns/{credential}/*` with
   `{"rollback_to": {"slot_no": N}}` body. Kupo re-indexes from that slot.
2. **Nuke DB** (requires restart): Delete the database volume and restart
   with all needed credentials from origin.

### `toBytes()` returns hex string, not Uint8Array

`Address.toBytes()` returns a hex string. Don't wrap it in
`Buffer.from(...).toString("hex")` — that double-encodes:

```typescript
// BAD — double encoding
const addrBytes = usedAddr.toBytes();
const keyHash = Buffer.from(addrBytes.slice(1, 29)).toString("hex");

// GOOD — already hex
const addrHex = usedAddr.toBytes() as unknown as string;
const keyHash = addrHex.slice(2, 58);
```
