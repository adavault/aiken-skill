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

### Tests can only have 0 or 1 argument

Aiken tests accept at most one parameter. Multiple fuzz parameters cause
`aiken::check::illegal::test::arity` — and the compiler may silently exit
with code 1 and **no error output** when stdout is not a TTY (e.g., in CI
or piped through redirects). Use `script -q -c "aiken build" /dev/null` to
force TTY output if errors disappear.

Use `fuzz.both()` to combine two fuzzers into a tuple:

```aiken
// BAD — 2 arguments, compiler error (may be silent!)
test prop_test(t1 via fuzz.int(), t2 via fuzz.int()) { ... }

// GOOD — single tuple argument
test prop_test(params via fuzz.both(fuzz.int(), fuzz.int())) {
  let (t1, t2) = params
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

### `expect` vs `let` for side-effect booleans

Use `expect` (not `let`) when you want a boolean expression's `?` trace to
actually execute. Unused `let` bindings are **fully optimized away** — they
produce no side effects:

```aiken
// BAD — _time_ok is unused, entire expression (including ?) is removed
let _time_ok = interval.is_entirely_after(tx.validity_range, t)?

// GOOD — expect enforces the side effect
expect interval.is_entirely_after(tx.validity_range, t)?
```

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

### `selectUtxosFrom()` required for continuing outputs

When spending a script UTxO and sending all ADA back to the script (state
machine, check-in), there's no ADA left for fees. The tx builder needs
wallet UTxOs explicitly:

```typescript
// BAD — "Insufficient lovelace" or silent failure
await txBuilder
  .spendingPlutusScriptV3()
  .txIn(scriptUtxo.input.txHash, scriptUtxo.input.outputIndex)
  // ... script chain ...
  .txOut(scriptAddress, [{ unit: "lovelace", quantity: "5000000" }])
  .complete();

// GOOD — wallet UTxOs available for fee coverage
const walletUtxos = await kupo.fetchAddressUTxOs(walletAddress);
await txBuilder
  // ... same chain ...
  .selectUtxosFrom(walletUtxos)
  .complete();
```

### `invalidBefore` needs 60s safety margin

Setting `invalidBefore` to the exact current slot causes "submitted too early"
due to clock skew between the local machine and the ledger tip:

```typescript
// BAD — may fail with "submitted too early"
const currentSlot = Math.floor(Date.now() / 1000) - 1666656000;
txBuilder.invalidBefore(currentSlot);

// GOOD — 60s safety margin
txBuilder.invalidBefore(currentSlot - 60);
```

### UTxO contention across sequential transactions

Two transactions submitted in the same block that reference the same UTxO will
cause "unknown UTxO references" (code 3117) for the second. Wait ~30s (next
block) between dependent transactions.

### Conway staking registration is permissionless

`registerStakeCertificate(rewardAddress)` needs no script witness, no
redeemer, and no collateral. It's a simple certificate in the tx body:

```typescript
// No .spendingPlutusScriptV3(), no .txInCollateral() needed
await txBuilder
  .registerStakeCertificate(rewardAddress)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  .selectUtxosFrom(utxos)
  .complete();
```

### Aiken `else` handler blocks unmatched purposes

Aiken auto-generates a default `else` handler that fails for all unmatched
handler purposes. If your script only has a `withdraw` handler, you cannot
deregister the staking credential (a `publish` action) because the `else`
handler rejects it. Fix: add an explicit `else` handler or accept the
credential stays registered.

### Kupo `rollback_to` must be within 36h safe zone

Using `slot_no: 0` returns a "hint" warning instead of registering the
pattern. Always use a recent slot from the checkpoints API:

```bash
SLOT=$(curl -s http://localhost:1442/v1/checkpoints \
  | python3 -c "import json,sys; print(json.load(sys.stdin)[-1]['slot_no'])")
```

### `List<Pair<K,V>>` compiles to Plutus map encoding

Aiken's `List<Pair<ByteArray, Int>>` becomes a map type in the blueprint.
Use `{ map: [{ k: ..., v: ... }] }` not `{ list: [{ list: [...] }] }`:

```typescript
// BAD — wrong encoding for List<Pair<...>>
{ list: [{ list: [{ bytes: "aa" }, { int: 50 }] }] }

// GOOD — map encoding
{ map: [{ k: { bytes: "aa" }, v: { int: 50 } }] }
```

### Multi-handler validators share the same script hash

For validators with multiple handlers (e.g., `mint` + `spend`), the policy ID
equals the spend script hash — they're the same compiled code. Use the same
parameter values for both `MintingBlueprint` and `SpendingBlueprint`.

### `tx.inputs` are sorted — redeemer indices must match

On Cardano, `tx.inputs` are sorted lexicographically by `(txHash, outputIndex)`.
The UTxO indexer pattern (and any pattern using `list.at(tx.inputs, index)`)
must account for this. When the tx builder auto-selects fee UTxOs, they may
sort before the script UTxO, shifting its index.

**Fix:** Determine the sort order at build time and set redeemer indices
accordingly. Use explicit `.txIn()` for fee inputs so you control the ordering:

```typescript
// Determine where the script UTxO falls in sorted order
const scriptRef = `${scriptUtxo.input.txHash}#${scriptUtxo.input.outputIndex}`;
const feeRef = `${feeUtxo.input.txHash}#${feeUtxo.input.outputIndex}`;
const scriptInputIndex = scriptRef < feeRef ? 0 : 1;

const redeemer = {
  constructor: 0,
  fields: [
    { int: scriptInputIndex },  // determined by sort order
    { int: 0 },                 // output index (first txOut)
  ],
};

await txBuilder
  .spendingPlutusScriptV3()
  .txIn(scriptUtxo.input.txHash, scriptUtxo.input.outputIndex)
  // ... script chain ...
  .txIn(feeUtxo.input.txHash, feeUtxo.input.outputIndex)  // explicit fee input
  // ... rest of chain (no selectUtxosFrom needed) ...
  .complete();
```

### `invalidBefore` needs 180s margin on preview

The preview testnet ledger tip can lag 60-200s behind wall clock due to
Poisson block distribution. Use `currentSlot - 180` (not 60) to avoid
"submitted too early" errors:

```typescript
const currentSlot = Math.floor(Date.now() / 1000) - 1666656000;
txBuilder.invalidBefore(currentSlot - 180);
```

### `invalidBefore` POSIX alignment for `is_entirely_after`

When a validator checks `is_entirely_after(validity_range, current_time - 1)` and the
redeemer carries `current_time`, derive it from the slot rather than `Date.now()`.
Slot→POSIX conversion rounds to whole seconds; `Date.now()` has millisecond precision
and can produce a value slightly ahead of the slot boundary, failing the check:

```typescript
// BAD — ms precision from Date.now() can exceed slot POSIX
const currentTimeMs = Date.now() - 180_000;

// GOOD — derive from the slot to guarantee alignment
const invalidBeforeSlot = currentSlot - 180;
const currentTimeMs = slotToPosixMs(invalidBeforeSlot) - 1;
```

### CIP-68 spend handler must branch on redeemer variant

CIP-68 spend handlers must check the redeemer variant — `Burn` actions need to
release the UTxO without requiring a continuing output. If the spend handler
unconditionally demands a continuing output, burn transactions fail because
the tokens are being destroyed:

```aiken
// BAD — unconditional continuing output check blocks burn
spend(...) {
  admin_signed && has_continuing_output
}

// GOOD — branch on redeemer
spend(datum, redeemer, oref, tx) {
  when redeemer is {
    Burn { .. } -> admin_signed
    _ -> admin_signed && has_continuing_output
  }
}
```

## Aiken Language & Stdlib

### `use` imports must be at the top of the file

All `use` statements must appear before the first non-import declaration
(const, fn, type, validator, test). Adding a `use` statement mid-file
(e.g., before test helpers) causes a parser error:

```
I found an unexpected token 'use'. I am looking for: const, fn, test, validator, ...
```

Add all imports needed by both the validator and its tests at the top.

### `builtin.less_than_bytearray` for ByteArray comparison

The `<` operator is not supported for `ByteArray`. Use the builtin:

```aiken
// BAD — type error
a < b  // where a, b: ByteArray

// GOOD
use aiken/builtin
builtin.less_than_bytearray(a, b)
```

Useful for sorted data structures (e.g., linked list key ordering).

### `dict.union_with` in stdlib v3 uses CPS-style callbacks

In stdlib v3.0.0, `dict.union_with` takes a `UnionStrategy` — a CPS-style
callback, not a raw merge function. For nested dict merging (e.g., summing
token quantities across policies):

```aiken
use aiken/collection/dict/strategy

// Simple numeric merge — use strategy.sum()
dict.union_with(a, b, strategy.sum())

// Nested dict merge — CPS callback with keep/discard
dict.union_with(a, b, fn(_key, val_a, val_b, keep, _discard) {
  keep(dict.union_with(val_a, val_b, strategy.sum()))
})
```

### `assets.tokens()` returns `Dict`, not `List`

Use `dict.is_empty()` and `dict.to_pairs()`, not list operations:

```aiken
// BAD — type error
let tokens = assets.tokens(value, cs)
list.is_empty(tokens)

// GOOD
dict.is_empty(tokens)
dict.to_pairs(tokens) == [Pair(expected_name, 1)]
```

### `expect <bool_expression>` fails the validator if False

Aiken's `expect` on a Bool expression acts as `expect True = expr`. This is
the idiomatic way to assert conditions in validators:

```aiken
// These are equivalent:
expect has_currency_symbol(output.value, cs)
expect True = has_currency_symbol(output.value, cs)

// Both fail the transaction if the function returns False
```

### Validator files are separate compilation units

Each `.ak` file in `validators/` is a separate compilation unit. You cannot
reference one validator from another validator file:

```aiken
// validators/my_validator_test.ak — FAILS
// Error: unknown module 'my_validator'
my_validator.spend(...)

// SOLUTION: put tests in the same file as the validator
// validators/my_validator.ak
validator my_validator { ... }
test my_test() { my_validator.spend(...) }
```

Library files in `lib/` can be shared across validator files via `use`.

### Record spread for cleaner datum comparison

Use record spread to create expected values, avoiding field-by-field checks:

```aiken
// VERBOSE
expect updated.key == original.key
expect updated.next == new_key
expect updated.field_a == original.field_a
expect updated.field_b == original.field_b

// CLEAN — spread original, override changed fields
let expected = MyType { ..original, next: new_key }
expect updated == expected
```

## Architectural Patterns

### Sorted linked list with covering node proofs

On-chain sorted linked lists use sentinel origin nodes and covering node
proofs for O(1) membership verification:

```aiken
// Origin (sentinel): key = #"", next = #"" (or first real key)
// Insert: find covering node where covering.key < new_key < covering.next
// The empty bytearray is always less than any non-empty bytearray:
//   builtin.less_than_bytearray(#"", #"anything") == True

// Membership proof: TokenExists — node.key == query
// Non-membership proof: TokenDoesNotExist — node.key < query < node.next
```

Insert splits a covering node into two outputs:
1. Updated covering: `{ ..covering, next: new_key }`
2. New node: `{ key: new_key, next: covering.next }`

Security: count registry inputs during insert to prevent extra-node-spend
attacks where an attacker spends unrelated nodes alongside a valid insert.

### Return value vs `expect` for security-critical Bool checks

When a function returns Bool and the result gates a security property,
use `expect` instead of returning the Bool to the caller:

```aiken
// DANGEROUS — caller can silently ignore False
fn check_transfer_logic(tx, withdrawals, cred) -> Bool {
  has_key(withdrawals, cred)  // False = token treated as non-programmable
}

// SAFE — transaction fails immediately if check fails
fn check_transfer_logic(tx, withdrawals, cred) -> Bool {
  expect has_key(withdrawals, cred)
  True
}
```

This prevents registered tokens from being silently treated as non-programmable,
which would allow value to escape a custody system.
