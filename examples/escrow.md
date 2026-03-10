# Escrow — Time-Locked Two-Party Exchange

A time-locked escrow where two parties exchange assets. Both must sign to
complete. If the deadline passes, the seller can reclaim. Seller can cancel
anytime.

## Types

```aiken
pub type EscrowDatum {
  seller: ByteArray,
  buyer: ByteArray,
  price: Int,
  deadline: Int,
}

pub type EscrowRedeemer {
  Complete   // Both parties agree
  Refund     // Deadline passed — seller reclaims
  Cancel     // Seller cancels before completion
}
```

## Validator

```aiken
// validators/escrow.ak

use aiken/collection/list
use aiken/interval
use cardano/transaction.{OutputReference, Transaction}

validator escrow {
  spend(
    datum: Option<EscrowDatum>,
    redeemer: EscrowRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      Complete -> {
        let seller_signed = list.has(tx.extra_signatories, d.seller)?
        let buyer_signed = list.has(tx.extra_signatories, d.buyer)?
        seller_signed && buyer_signed
      }

      Refund -> {
        let seller_signed = list.has(tx.extra_signatories, d.seller)?
        let past_deadline =
          interval.is_entirely_after(tx.validity_range, d.deadline)?
        seller_signed && past_deadline
      }

      Cancel ->
        list.has(tx.extra_signatories, d.seller)?
    }
  }
}
```

## Tests

```aiken
// validators/escrow.ak (continued)

const mock_seller =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_buyer =
  #"112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"

const mock_deadline = 1_700_000_000_000

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> EscrowDatum {
  EscrowDatum {
    seller: mock_seller,
    buyer: mock_buyer,
    price: 50_000_000,
    deadline: mock_deadline,
  }
}

test complete_with_both_signatures_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_seller, mock_buyer],
    }
  escrow.spend(Some(mock_datum()), Complete, mock_oref(), tx)
}

test complete_without_buyer_signature_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_seller],
    }
  escrow.spend(Some(mock_datum()), Complete, mock_oref(), tx)
}

test refund_after_deadline_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_seller],
      validity_range: interval.after(mock_deadline + 1),
    }
  escrow.spend(Some(mock_datum()), Refund, mock_oref(), tx)
}

test refund_before_deadline_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_seller],
      validity_range: interval.before(mock_deadline),
    }
  escrow.spend(Some(mock_datum()), Refund, mock_oref(), tx)
}

test cancel_by_seller_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_seller],
    }
  escrow.spend(Some(mock_datum()), Cancel, mock_oref(), tx)
}

test cancel_by_non_seller_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_buyer],
    }
  escrow.spend(Some(mock_datum()), Cancel, mock_oref(), tx)
}

// Property: any pair of matching signers can complete
test prop_matching_signers_can_complete(
  params via fuzz.both(fuzz.bytearray_fixed(28), fuzz.bytearray_fixed(28)),
) {
  let (seller, buyer) = params
  let datum =
    EscrowDatum {
      seller: seller,
      buyer: buyer,
      price: 50_000_000,
      deadline: mock_deadline,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [seller, buyer],
    }
  escrow.spend(Some(datum), Complete, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Multi-action redeemer** — Complete, Refund, Cancel with different validation per action
2. **Dual signature requirement** — `seller_signed && buyer_signed` for atomic agreement
3. **`interval.is_entirely_after`** — time-lock: seller can only refund after deadline
4. **`interval.after(time)`** — construct test validity range `[time, +inf)`
5. **`interval.before(time)`** — construct `(-inf, time]` range for "too early" tests
6. **Cancel anytime** — seller always has an exit before completion
