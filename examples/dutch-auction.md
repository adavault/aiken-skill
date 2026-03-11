# Dutch Auction — Linear Price Decay Over Time

A Dutch auction where the price starts high and decreases linearly over time
until someone buys or the price reaches the reserve floor. The seller locks an
item at the script address. Buyers purchase at the current decayed price. The
seller can cancel and reclaim at any time.

## Types

```aiken
pub type AuctionDatum {
  /// Seller's verification key hash
  seller: ByteArray,
  /// Starting price in lovelace
  start_price: Int,
  /// Minimum price (floor) in lovelace
  reserve_price: Int,
  /// POSIX ms when auction starts
  start_time: Int,
  /// Lovelace decrease per millisecond
  decay_per_ms: Int,
}

pub type AuctionRedeemer {
  /// Buyer purchases at the computed price
  Buy { current_time: Int }
  /// Seller cancels the auction
  Cancel
}
```

## Validator

```aiken
// validators/dutch_auction.ak

use aiken/collection/list
use aiken/interval
use cardano/address
use cardano/assets
use cardano/transaction.{Output, OutputReference, Transaction}

fn compute_price(datum: AuctionDatum, current_time: Int) -> Int {
  let elapsed = current_time - datum.start_time
  let decay = elapsed * datum.decay_per_ms
  let price = datum.start_price - decay
  if price < datum.reserve_price {
    datum.reserve_price
  } else {
    price
  }
}

validator dutch_auction {
  spend(
    datum: Option<AuctionDatum>,
    redeemer: AuctionRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      Buy { current_time } -> {
        // Compute the current price based on time decay
        let current_price = compute_price(d, current_time)

        // Verify current_time is anchored to the tx validity range.
        // The tx validity range must be entirely after current_time - 1,
        // meaning the ledger guarantees this time has passed.
        expect
          interval.is_entirely_after(tx.validity_range, current_time - 1)?

        // Check that the seller is paid at least the current price
        let seller_paid =
          list.any(
            tx.outputs,
            fn(output) {
              output.address == address.from_verification_key(d.seller) && assets.lovelace_of(
                output.value,
              ) >= current_price
            },
          )

        seller_paid?
      }

      Cancel ->
        // Seller must sign to cancel
        list.has(tx.extra_signatories, d.seller)?
    }
  }
}
```

## Tests

```aiken
// validators/dutch_auction.ak (continued)

const mock_seller =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_buyer =
  #"112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"

const auction_start = 1_700_000_000_000

const start_price = 100_000_000

const reserve_price = 20_000_000

// 10 lovelace per ms = 10_000 ADA per second
const decay_per_ms = 10

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> AuctionDatum {
  AuctionDatum {
    seller: mock_seller,
    start_price,
    reserve_price,
    start_time: auction_start,
    decay_per_ms,
  }
}

fn seller_output(amount: Int) -> Output {
  Output {
    address: address.from_verification_key(mock_seller),
    value: assets.from_lovelace(amount),
    datum: transaction.NoDatum,
    reference_script: None,
  }
}

// -- Buy Tests --

test buy_at_start_time_pays_full_price_succeeds() {
  let d = mock_datum()
  let buy_time = auction_start
  // At start time, price = start_price (no decay)
  let tx =
    Transaction {
      ..transaction.placeholder,
      outputs: [seller_output(start_price)],
      validity_range: interval.after(buy_time),
    }
  dutch_auction.spend(Some(d), Buy { current_time: buy_time }, mock_oref(), tx)
}

test buy_after_decay_pays_reduced_price_succeeds() {
  let d = mock_datum()
  // 1 second elapsed -> decay = 1000 * 10 = 10_000 lovelace
  let buy_time = auction_start + 1_000
  let expected_price = start_price - 1_000 * decay_per_ms
  let tx =
    Transaction {
      ..transaction.placeholder,
      outputs: [seller_output(expected_price)],
      validity_range: interval.after(buy_time),
    }
  dutch_auction.spend(Some(d), Buy { current_time: buy_time }, mock_oref(), tx)
}

test buy_at_floor_price_fully_decayed_succeeds() {
  let d = mock_datum()
  // Enough time for price to decay well below reserve
  // (100M - 20M) / 10 = 8_000_000 ms for price to reach reserve
  let buy_time = auction_start + 20_000_000
  let tx =
    Transaction {
      ..transaction.placeholder,
      outputs: [seller_output(reserve_price)],
      validity_range: interval.after(buy_time),
    }
  dutch_auction.spend(Some(d), Buy { current_time: buy_time }, mock_oref(), tx)
}

test buy_with_insufficient_payment_fails() fail {
  let d = mock_datum()
  let buy_time = auction_start
  // Pay less than start_price
  let tx =
    Transaction {
      ..transaction.placeholder,
      outputs: [seller_output(start_price - 1)],
      validity_range: interval.after(buy_time),
    }
  dutch_auction.spend(Some(d), Buy { current_time: buy_time }, mock_oref(), tx)
}

test buy_with_time_outside_validity_range_fails() fail {
  let d = mock_datum()
  let buy_time = auction_start + 5_000
  // Validity range starts BEFORE buy_time, so is_entirely_after fails
  // interval.before(buy_time) means range is (-inf, buy_time)
  let tx =
    Transaction {
      ..transaction.placeholder,
      outputs: [seller_output(start_price)],
      validity_range: interval.before(buy_time),
    }
  dutch_auction.spend(Some(d), Buy { current_time: buy_time }, mock_oref(), tx)
}

// -- Cancel Tests --

test cancel_by_seller_succeeds() {
  let d = mock_datum()
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [mock_seller] }
  dutch_auction.spend(Some(d), Cancel, mock_oref(), tx)
}

test cancel_by_non_seller_fails() fail {
  let d = mock_datum()
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [mock_buyer] }
  dutch_auction.spend(Some(d), Cancel, mock_oref(), tx)
}

// -- Property-Based Tests --

test prop_price_never_below_reserve(elapsed_ms via fuzz.int_at_least(0)) {
  let d = mock_datum()
  let current_time = auction_start + elapsed_ms
  let price = compute_price(d, current_time)
  price >= reserve_price
}

test prop_price_never_above_start(elapsed_ms via fuzz.int_at_least(0)) {
  let d = mock_datum()
  let current_time = auction_start + elapsed_ms
  let price = compute_price(d, current_time)
  price <= start_price
}

test prop_price_decreases_over_time(
  times via fuzz.both(
    fuzz.int_between(0, 1_000_000_000),
    fuzz.int_between(0, 1_000_000_000),
  ),
) {
  let (t1, t2) = times
  let d = mock_datum()
  let early = auction_start + t1
  let late = auction_start + t1 + t2
  let price_early = compute_price(d, early)
  let price_late = compute_price(d, late)
  price_early >= price_late
}
```

## Key Concepts Demonstrated

1. **Linear price decay** — `compute_price` calculates `start_price - elapsed * decay_per_ms`, floored at `reserve_price`
2. **Validity range anchoring** — `interval.is_entirely_after(tx.validity_range, current_time - 1)` binds the redeemer time to the ledger clock
3. **Seller payment verification** — `list.any` over `tx.outputs` checks the seller receives at least the current price
4. **`address.from_verification_key`** — reconstruct the seller's address to match against output addresses
5. **Cancel anytime** — seller can always reclaim with a signature, no time constraint
6. **Property: price bounded** — fuzz tests prove price stays within `[reserve_price, start_price]` for all elapsed times
7. **Property: monotonic decay** — fuzz test proves `price(t1) >= price(t1 + t2)` for all non-negative offsets
