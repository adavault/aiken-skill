# Marketplace — NFT Listing, Buying, and Cancelling

A simple NFT marketplace where sellers list NFTs at a fixed price. Buyers pay
the asking price to the seller to purchase. Sellers can cancel (delist) at any
time by signing the transaction.

## Types

```aiken
pub type ListingDatum {
  seller: ByteArray,
  price: Int,
  nft_policy: ByteArray,
  nft_name: ByteArray,
}

pub type MarketplaceRedeemer {
  Buy
  Cancel
}
```

## Validator

```aiken
// validators/marketplace.ak

use aiken/collection/list
use cardano/address
use cardano/assets
use cardano/transaction.{Input, InlineDatum, NoDatum, Output, OutputReference,
  Transaction}

validator marketplace {
  spend(
    datum: Option<ListingDatum>,
    redeemer: MarketplaceRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      Buy -> {
        // Seller must receive at least the listing price
        let seller_addr = address.from_verification_key(d.seller)
        let seller_paid =
          list.any(
            tx.outputs,
            fn(o) {
              o.address == seller_addr && assets.lovelace_of(o.value) >= d.price
            },
          )?
        seller_paid
      }

      Cancel ->
        // Only the seller can cancel a listing
        list.has(tx.extra_signatories, d.seller)?
    }
  }
}
```

## Tests

```aiken
// validators/marketplace.ak (continued)

const mock_seller =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_buyer =
  #"112233445566778899aabbccddeeff00112233445566778899aabbccddeeff00"

const mock_nft_policy =
  #"cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc"

const mock_nft_name = "CoolNFT"

const mock_price = 100_000_000

const mock_script_hash =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> ListingDatum {
  ListingDatum {
    seller: mock_seller,
    price: mock_price,
    nft_policy: mock_nft_policy,
    nft_name: mock_nft_name,
  }
}

fn mock_listing_input() -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: address.from_script(mock_script_hash),
      value: assets.from_lovelace(2_000_000)
        |> assets.add(mock_nft_policy, mock_nft_name, 1),
      datum: InlineDatum(mock_datum()),
      reference_script: None,
    },
  }
}

fn seller_payment_output(amount: Int) -> Output {
  Output {
    address: address.from_verification_key(mock_seller),
    value: assets.from_lovelace(amount),
    datum: NoDatum,
    reference_script: None,
  }
}

fn buyer_nft_output() -> Output {
  Output {
    address: address.from_verification_key(mock_buyer),
    value: assets.from_lovelace(2_000_000)
      |> assets.add(mock_nft_policy, mock_nft_name, 1),
    datum: NoDatum,
    reference_script: None,
  }
}

test buy_succeeds_when_seller_paid() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_listing_input()],
      outputs: [seller_payment_output(mock_price), buyer_nft_output()],
    }
  marketplace.spend(Some(mock_datum()), Buy, mock_oref(), tx)
}

test buy_succeeds_when_seller_overpaid() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_listing_input()],
      outputs: [
        seller_payment_output(mock_price + 10_000_000),
        buyer_nft_output(),
      ],
    }
  marketplace.spend(Some(mock_datum()), Buy, mock_oref(), tx)
}

test buy_fails_if_seller_underpaid() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_listing_input()],
      outputs: [
        seller_payment_output(mock_price - 1),
        buyer_nft_output(),
      ],
    }
  marketplace.spend(Some(mock_datum()), Buy, mock_oref(), tx)
}

test buy_fails_if_no_payment_to_seller() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_listing_input()],
      outputs: [buyer_nft_output()],
    }
  marketplace.spend(Some(mock_datum()), Buy, mock_oref(), tx)
}

test cancel_by_seller_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_seller],
    }
  marketplace.spend(Some(mock_datum()), Cancel, mock_oref(), tx)
}

test cancel_by_non_seller_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_buyer],
    }
  marketplace.spend(Some(mock_datum()), Cancel, mock_oref(), tx)
}

test cancel_with_no_signatories_fails() fail {
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [] }
  marketplace.spend(Some(mock_datum()), Cancel, mock_oref(), tx)
}

// Property: seller always gets paid regardless of price
test prop_seller_always_gets_paid_on_buy(
  params via fuzz.both(fuzz.bytearray_fixed(28), fuzz.int_at_least(1)),
) {
  let (seller, price) = params
  let datum =
    ListingDatum {
      seller: seller,
      price: price,
      nft_policy: mock_nft_policy,
      nft_name: mock_nft_name,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      outputs: [
        Output {
          address: address.from_verification_key(seller),
          value: assets.from_lovelace(price),
          datum: NoDatum,
          reference_script: None,
        },
      ],
    }
  marketplace.spend(Some(datum), Buy, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Output scanning for payment** — `list.any(tx.outputs, fn(o) { o.address == seller_addr && ... })` verifies seller gets paid
2. **`address.from_verification_key()`** — constructs address from key hash for output comparison
3. **`assets.lovelace_of(output.value) >= d.price`** — allows overpayment, blocks underpayment
4. **NFT in value** — `assets.add(policy, name, 1)` pipes onto `assets.from_lovelace()` to build multi-asset values
5. **No signature required for Buy** — anyone can purchase; the payment check is the authorization
6. **Seller-only Cancel** — `list.has(tx.extra_signatories, d.seller)` for delist authorization
7. **Datum stores NFT identity** — `nft_policy` + `nft_name` enable off-chain listing discovery

## Security Notes

- **Double satisfaction risk**: Multiple listings in one transaction — buyer could pay seller once and satisfy multiple validators. Mitigate with NFT authentication or unique listing tokens.
- **Payment output scanning**: `list.any` stops at first match — if seller has multiple outputs, any single one meeting the price threshold suffices.
- **No royalties**: This simple model pays only the seller. Production marketplaces add creator royalty outputs.
- **Off-chain coordination**: The NFT must actually be at the script address — the validator doesn't check for it (that's a UTxO-level guarantee from Cardano ledger rules).
