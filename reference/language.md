# Aiken Language Reference

## Types

### Primitives

```aiken
// Integers — arbitrary precision, no overflow
let x: Int = 42
let big = 1_000_000_000

// Bool
let flag: Bool = True

// ByteArray — three literal forms
let bytes = #[0, 255, 128]           // byte list
let utf8 = "hello"                    // UTF-8 encoded
let hex = #"deadbeef"                 // hex encoded

// String — for traces/debugging ONLY, not for on-chain data
let msg = @"debug message"

// Void — unit type
let unit: Void = Void
```

### Custom Types

**Important:** Types used in validator handler signatures (datum, redeemer) must
be `pub`. Without `pub`, the compiler raises a `private_leak` error.

```aiken
// Product type (single constructor) — pub required if used in validator signature
pub type Datum {
  owner: VerificationKeyHash,
  deadline: Int,
  amount: Int,
}

// Sum type (multiple constructors)
type Redeemer {
  Lock { amount: Int }
  Unlock
  Cancel
}

// Generic type
type Result<a, b> {
  Ok(a)
  Err(b)
}

// Opaque type — internal structure hidden from importers
pub opaque type ValidatedAmount {
  inner: Int,
}
```

### Type Aliases

```aiken
type POSIXTime = Int
type Lovelace = Int
```

### PlutusData Encoding

Constructors are encoded with ascending indices by default:

```aiken
type Action {
  Mint      // index 0
  Burn      // index 1
  Transfer  // index 2
}
```

Override with `@tag`:
```aiken
type Action {
  @tag(0) Mint
  @tag(121) Burn     // custom constr index
}
```

List encoding override:
```aiken
type Pair {
  @list             // encode as list instead of constr
  Pair(Int, Int)
}
```

## Pattern Matching

```aiken
// when/is — must be exhaustive
when action is {
  Mint -> handle_mint()
  Burn -> handle_burn()
  Transfer -> handle_transfer()
}

// Nested patterns
when datum is {
  Some(Datum { owner, deadline, .. }) -> check(owner, deadline)
  None -> fail @"missing datum"
}

// With guards (if)
when amount is {
  n if n > 0 -> handle_positive(n)
  n if n == 0 -> handle_zero()
  _ -> fail @"negative"
}

// if/is — soft cast with fallback
if datum is Some(d) {
  use_datum(d)
} else {
  default_behavior()
}
```

## expect — Unsafe Downcast

```aiken
// Fails transaction if pattern doesn't match
expect Some(datum) = datum_opt
expect InlineDatum(raw) = output.datum
expect typed: MyDatum = raw

// Common pattern: extract inline datum
expect Some(input) = transaction.find_input(tx.inputs, oref)
expect InlineDatum(raw) = input.output.datum
expect datum: VaultDatum = raw
```

## Functions

```aiken
// Named function
fn add(a: Int, b: Int) -> Int {
  a + b
}

// Public function (exported from module)
pub fn validate(tx: Transaction) -> Bool {
  todo
}

// Anonymous function
let double = fn(x) { x * 2 }

// Pipe operator
[1, 2, 3]
  |> list.map(fn(x) { x * 2 })
  |> list.filter(fn(x) { x > 2 })
  |> list.length
```

## Backpassing

Continuation-style composition with `<-`:

```aiken
// Instead of nested callbacks:
let result =
  list.map(items, fn(item) {
    do_something(item, fn(x) {
      finalize(x)
    })
  })

// Use backpassing:
let x <- list.map(items)
let y <- do_something(x)
finalize(y)
```

## Modules

**Important:** All `use` imports must appear at the top of the file, before any
type definitions, constants, or validators. Aiken does not allow imports after
other declarations.

```aiken
// Import
use aiken/collection/list
use cardano/transaction.{Transaction, find_input}
use cardano/assets.{lovelace_of}

// Qualified usage
list.has(signatories, owner)

// Unqualified (imported directly)
let input = find_input(tx.inputs, oref)

// Module visibility
pub fn exported() { .. }    // visible to importers
fn internal() { .. }        // module-private
```

## Environment Modules

Network-specific configuration via `env/` directory:

```
lib/
  env/
    preview.ak      // use with: aiken build -e preview
    preprod.ak       // use with: aiken build -e preprod
    mainnet.ak       // use with: aiken build -e mainnet
```

```aiken
// env/preview.ak
pub const network_id = 0

// env/mainnet.ak
pub const network_id = 1

// Usage in validator
use env

fn check_network() {
  env.network_id == 1
}
```

## Config Values

Values from `aiken.toml` injected as `config` module:

```toml
# aiken.toml
[config]
min_lock_amount = 5_000_000
max_lock_duration = 365
```

```aiken
use config

fn validate_amount(amount: Int) -> Bool {
  amount >= config.min_lock_amount
}
```

## Trace and Debug

```aiken
// trace — emit debug string (stripped in production)
trace @"entering validation"

// ? operator — trace on False
list.has(tx.extra_signatories, owner)?

// cbor.diagnostic — inspect any value
trace cbor.diagnostic(datum)

// Build with trace levels
// aiken build                        — compact traces
// aiken build --trace-level verbose  — full traces
// aiken build --trace-level silent   — no traces (production)
```

## Record Update Syntax

```aiken
let updated_datum = Datum { ..original_datum, amount: new_amount }

// Common pattern: modify transaction.placeholder for tests
let tx = Transaction {
  ..transaction.placeholder,
  extra_signatories: [signer],
  validity_range: interval.after(deadline),
}
```

## Constants

```aiken
const min_ada: Int = 2_000_000
const token_name: ByteArray = "VAULT"
```
