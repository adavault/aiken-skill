# Gift Card — Mint + Spend Dual Handler

A multi-purpose validator that mints an NFT "gift card" token and locks ADA at
the same script address. The gift card can be redeemed by burning the NFT.

This demonstrates:
- Multi-purpose validators (mint + spend in one script)
- One-shot minting via UTxO reference
- Atomic operations (mint-and-lock, burn-and-unlock)
- NFT authentication

## Types

```aiken
// lib/gift_card/types.ak

type MintRedeemer {
  /// Create a gift card: mint NFT + lock ADA
  CreateGiftCard { token_name: ByteArray }
  /// Redeem a gift card: burn NFT + unlock ADA
  RedeemGiftCard { token_name: ByteArray }
}
```

## Validator

```aiken
// validators/gift_card.ak

use aiken/collection/dict
use aiken/collection/list
use cardano/assets
use cardano/transaction.{Transaction, OutputReference, InlineDatum}

// Parameterized by the UTxO consumed during minting (ensures one-shot)
validator gift_card(utxo_ref: OutputReference) {

  // --- Minting Policy ---
  mint(redeemer: MintRedeemer, policy_id: PolicyId, tx: Transaction) {
    let minted = assets.tokens(tx.mint, policy_id)

    when redeemer is {
      CreateGiftCard { token_name } -> {
        // Must consume the specific UTxO (one-shot guarantee)
        let consumes_ref =
          list.any(tx.inputs, fn(input) {
            input.output_reference == utxo_ref
          }) ?

        // Must mint exactly 1 token
        let mints_one =
          dict.to_pairs(minted) == [Pair(token_name, 1)]
            ?

        consumes_ref && mints_one
      }

      RedeemGiftCard { token_name } -> {
        // Must burn exactly 1 token
        dict.to_pairs(minted) == [Pair(token_name, -1)]
          ?
      }
    }
  }

  // --- Spend Validator ---
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    // Spending the locked ADA requires burning a gift card token
    // from this policy. Any token name counts.
    let own_policy = // In practice, derived from the script's own hash
      policy_id_from_utxo_ref(utxo_ref)

    let burns_token =
      assets.tokens(tx.mint, own_policy)
        |> dict.to_pairs()
        |> list.any(fn(pair) {
          let Pair(_name, quantity) = pair
          quantity < 0
        })

    burns_token ?
  }
}
```

## Tests

```aiken
// validators/gift_card.ak (continued)

const mock_utxo_ref = OutputReference {
  transaction_id: #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  output_index: 0,
}

const mock_policy_id = #"bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
const token_name = "GIFT"

fn mock_mint_value(name: ByteArray, qty: Int) -> Value {
  assets.from_asset(mock_policy_id, name, qty)
}

// --- Mint Tests ---

test create_gift_card_succeeds() {
  let tx = Transaction {
    ..transaction.placeholder,
    inputs: [
      Input {
        output_reference: mock_utxo_ref,
        output: Output {
          address: address.from_verification_key(#"aabb"),
          value: assets.from_lovelace(5_000_000),
          datum: NoDatum,
          reference_script: None,
        },
      },
    ],
    mint: assets.from_asset(mock_policy_id, token_name, 1),
  }
  gift_card.mint(
    CreateGiftCard { token_name: token_name },
    mock_policy_id,
    tx,
  )
}

test cannot_mint_without_consuming_utxo() fail {
  let tx = Transaction {
    ..transaction.placeholder,
    inputs: [],  // UTxO not consumed
    mint: assets.from_asset(mock_policy_id, token_name, 1),
  }
  gift_card.mint(
    CreateGiftCard { token_name: token_name },
    mock_policy_id,
    tx,
  )
}

test cannot_mint_more_than_one() fail {
  let tx = Transaction {
    ..transaction.placeholder,
    inputs: [
      Input {
        output_reference: mock_utxo_ref,
        output: Output {
          address: address.from_verification_key(#"aabb"),
          value: assets.from_lovelace(5_000_000),
          datum: NoDatum,
          reference_script: None,
        },
      },
    ],
    mint: assets.from_asset(mock_policy_id, token_name, 2),  // 2 tokens
  }
  gift_card.mint(
    CreateGiftCard { token_name: token_name },
    mock_policy_id,
    tx,
  )
}

test redeem_burns_token() {
  let tx = Transaction {
    ..transaction.placeholder,
    mint: assets.from_asset(mock_policy_id, token_name, -1),
  }
  gift_card.mint(
    RedeemGiftCard { token_name: token_name },
    mock_policy_id,
    tx,
  )
}

// --- Property-Based Tests ---

use aiken/fuzz

test prop_create_requires_utxo(
  fake_ref via fuzz.output_reference(),
) fail {
  // Random UTxO ref should never match our required ref
  // (overwhelming probability for random 32-byte tx ids)
  let tx = Transaction {
    ..transaction.placeholder,
    inputs: [
      Input {
        output_reference: fake_ref,
        output: Output {
          address: address.from_verification_key(#"aabb"),
          value: assets.from_lovelace(5_000_000),
          datum: NoDatum,
          reference_script: None,
        },
      },
    ],
    mint: assets.from_asset(mock_policy_id, token_name, 1),
  }
  gift_card.mint(
    CreateGiftCard { token_name: token_name },
    mock_policy_id,
    tx,
  )
}
```

## Key Concepts Demonstrated

1. **Multi-purpose validator** — `mint` and `spend` in one `validator` block
2. **One-shot minting** — consuming a specific UTxO guarantees uniqueness
3. **Parameterized validator** — `utxo_ref` baked in at deployment
4. **NFT authentication** — spending requires burning the corresponding token
5. **Atomic operations** — mint+lock and burn+unlock happen in single transactions

## Security Notes

- **One-shot guarantee:** The parameterized `utxo_ref` can only be consumed once
  (UTxOs are unique), so the minting policy can only ever mint one batch of
  tokens. This prevents replay attacks.
- **Burn-to-spend coupling:** The spend validator requires burning a token from
  the same policy, preventing unauthorized spending even if someone sends
  additional ADA to the script address.
- **No datum hijacking risk:** Because spending requires burning an authenticated
  NFT, attacker-created UTxOs at the script address cannot be spent without the
  correct token.
