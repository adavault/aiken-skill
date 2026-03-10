# Vesting — Time-Locked Spending

A validator that locks ADA until a specified time. The owner can always
withdraw; the beneficiary can only withdraw after the lock period expires.

## Types

```aiken
// lib/vesting/types.ak

type VestingDatum {
  /// POSIX time (milliseconds) when funds unlock
  lock_until: Int,
  /// Owner who locked the funds — can always withdraw
  owner: VerificationKeyHash,
  /// Beneficiary who receives after lock period
  beneficiary: VerificationKeyHash,
}

type VestingRedeemer {
  /// Owner withdraws (no time restriction)
  OwnerWithdraw
  /// Beneficiary claims (time-restricted)
  BeneficiaryClaim
}
```

## Validator

```aiken
// validators/vesting.ak

use aiken/collection/list
use aiken/interval
use cardano/transaction.{Transaction, OutputReference}
use vesting/types.{VestingDatum, VestingRedeemer, OwnerWithdraw, BeneficiaryClaim}

validator vesting {
  spend(
    datum: Option<VestingDatum>,
    redeemer: VestingRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      // Owner can withdraw at any time
      OwnerWithdraw ->
        list.has(tx.extra_signatories, d.owner)
          ?

      // Beneficiary can only withdraw after lock period
      BeneficiaryClaim -> {
        let signed_by_beneficiary =
          list.has(tx.extra_signatories, d.beneficiary)
            ?

        let lock_expired =
          interval.is_entirely_after(tx.validity_range, d.lock_until)
            ?

        signed_by_beneficiary && lock_expired
      }
    }
  }
}
```

## Tests

```aiken
// validators/vesting.ak (continued)

const mock_owner = #"aabbccddee112233445566778899001122334455667788990011223344"
const mock_beneficiary = #"112233445566778899aabbccddeeff0011223344556677889900aabb"
const lock_time = 1_700_000_000_000  // some POSIX time in milliseconds

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> VestingDatum {
  VestingDatum {
    lock_until: lock_time,
    owner: mock_owner,
    beneficiary: mock_beneficiary,
  }
}

// --- Owner Tests ---

test owner_can_withdraw_before_lock() {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_owner],
    // Validity range before lock time — doesn't matter for owner
    validity_range: interval.before(lock_time - 1_000),
  }
  vesting.spend(Some(mock_datum()), OwnerWithdraw, mock_oref(), tx)
}

test owner_can_withdraw_after_lock() {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_owner],
    validity_range: interval.after(lock_time + 1_000),
  }
  vesting.spend(Some(mock_datum()), OwnerWithdraw, mock_oref(), tx)
}

test non_owner_cannot_use_owner_withdraw() fail {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_beneficiary],
  }
  vesting.spend(Some(mock_datum()), OwnerWithdraw, mock_oref(), tx)
}

// --- Beneficiary Tests ---

test beneficiary_can_claim_after_lock() {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_beneficiary],
    // Must be entirely after lock_time
    validity_range: interval.after(lock_time + 1),
  }
  vesting.spend(Some(mock_datum()), BeneficiaryClaim, mock_oref(), tx)
}

test beneficiary_cannot_claim_before_lock() fail {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [mock_beneficiary],
    validity_range: interval.before(lock_time),
  }
  vesting.spend(Some(mock_datum()), BeneficiaryClaim, mock_oref(), tx)
}

test beneficiary_needs_signature() fail {
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [],  // no signature
    validity_range: interval.after(lock_time + 1),
  }
  vesting.spend(Some(mock_datum()), BeneficiaryClaim, mock_oref(), tx)
}

// --- Property-Based Tests ---

use aiken/fuzz

// Multiple fuzzed values: use fuzz.both() — tests allow max 1 argument
test prop_owner_always_authorized(
  params via fuzz.both(fuzz.bytearray_fixed(28), fuzz.int()),
) {
  let (owner, time) = params
  let datum = VestingDatum {
    lock_until: lock_time,
    owner: owner,
    beneficiary: mock_beneficiary,
  }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [owner],
    validity_range: interval.after(time),
  }
  // Owner can always withdraw regardless of time
  vesting.spend(Some(datum), OwnerWithdraw, mock_oref(), tx)
}

// No arithmetic in fuzzer args — use a constant
const after_lock_time = 1_700_000_000_001

test prop_beneficiary_needs_both_sig_and_time(
  params via fuzz.both(
    fuzz.bytearray_fixed(28),
    fuzz.int_at_least(after_lock_time),
  ),
) {
  let (beneficiary, claim_time) = params
  let datum = VestingDatum {
    lock_until: lock_time,
    owner: mock_owner,
    beneficiary: beneficiary,
  }
  let tx = Transaction {
    ..transaction.placeholder,
    extra_signatories: [beneficiary],
    validity_range: interval.after(claim_time),
  }
  vesting.spend(Some(datum), BeneficiaryClaim, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Time validation** — `interval.is_entirely_after()` with validity range
2. **Dual authorization** — different rules per redeemer variant
3. **interval.after() vs interval.before()** — constructing test ranges
4. **Trace operator** — postfix `?` auto-traces expression name on failure
5. **Off-chain requirement** — wallet must set `invalid_before` for time checks

## Off-Chain Integration Note

For the beneficiary to claim, the off-chain code (MeshJS/Lucid) must set the
transaction's `invalid_before` field to a slot after `lock_until`. Without this,
`interval.is_entirely_after()` returns False because the validity range extends
to negative infinity.

```typescript
// MeshJS example
const tx = new Transaction({ initiator: wallet })
  .setTimeToStart(lockUntilSlot + 1)  // Sets invalid_before
  .redeemValue({ ... })
  .build();
```
