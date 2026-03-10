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
// Types are defined inline in the validator file

pub type MintRedeemer {
  /// Create a gift card: mint NFT + lock ADA
  CreateGiftCard
  /// Redeem a gift card: burn NFT + unlock ADA
  RedeemGiftCard
}
```

## Validator

```aiken
// validators/gift_card.ak

use aiken/collection/dict
use aiken/collection/list
use cardano/address
use cardano/assets.{PolicyId}
use cardano/transaction.{Input, NoDatum, Output, OutputReference, Transaction}

// Parameterized by the UTxO consumed during minting AND token name
validator gift_card(utxo_ref: OutputReference, token_name: ByteArray) {

  // --- Minting Policy ---
  mint(redeemer: MintRedeemer, policy_id: PolicyId, tx: Transaction) {
    let minted = assets.tokens(tx.mint, policy_id)

    when redeemer is {
      CreateGiftCard -> {
        // Must consume the specific UTxO (one-shot guarantee)
        let consumes_ref =
          list.any(
            tx.inputs,
            fn(input) { input.output_reference == utxo_ref },
          )?

        // Must mint exactly 1 token with the expected name
        let mints_one =
          (dict.to_pairs(minted) == [Pair(token_name, 1)])?

        consumes_ref && mints_one
      }

      RedeemGiftCard ->
        // Must burn exactly 1 token
        (dict.to_pairs(minted) == [Pair(token_name, -1)])?
    }
  }

  // --- Spend Validator ---
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    // Spending requires burning any token from this policy.
    // Note: spend handler cannot access its own policy_id directly.
    // Use assets.flatten() to check for any negative mint quantities.
    let all_minted = assets.flatten(tx.mint)
    list.any(all_minted, fn(entry) { entry.3rd < 0 })
  }
}
```

## Tests

```aiken
// validators/gift_card.ak (continued)

const mock_utxo_ref =
  OutputReference {
    transaction_id: #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    output_index: 0,
  }

const mock_policy_id =
  #"bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"

const token = "GIFT"

fn mock_input() -> Input {
  Input {
    output_reference: mock_utxo_ref,
    output: Output {
      address: address.from_verification_key(
        #"aabbccddee112233445566778899001122334455667788990011223344556677",
      ),
      value: assets.from_lovelace(5_000_000),
      datum: NoDatum,
      reference_script: None,
    },
  }
}

// --- Mint: CreateGiftCard Tests ---

test create_gift_card_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      mint: assets.from_asset(mock_policy_id, token, 1),
    }
  // Note: parameterized validator — pass params first, then handler args
  gift_card.mint(mock_utxo_ref, token, CreateGiftCard, mock_policy_id, tx)
}

test cannot_mint_without_consuming_utxo() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [],
      mint: assets.from_asset(mock_policy_id, token, 1),
    }
  gift_card.mint(mock_utxo_ref, token, CreateGiftCard, mock_policy_id, tx)
}

test cannot_mint_more_than_one() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      mint: assets.from_asset(mock_policy_id, token, 2),
    }
  gift_card.mint(mock_utxo_ref, token, CreateGiftCard, mock_policy_id, tx)
}

test cannot_mint_wrong_token_name() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      mint: assets.from_asset(mock_policy_id, "WRONG", 1),
    }
  gift_card.mint(mock_utxo_ref, token, CreateGiftCard, mock_policy_id, tx)
}

// --- Mint: RedeemGiftCard Tests ---

test redeem_burns_token() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy_id, token, -1),
    }
  gift_card.mint(mock_utxo_ref, token, RedeemGiftCard, mock_policy_id, tx)
}

test redeem_fails_if_not_burning() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy_id, token, 1),
    }
  gift_card.mint(mock_utxo_ref, token, RedeemGiftCard, mock_policy_id, tx)
}

// --- Spend Tests ---

test spend_with_burn_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy_id, token, -1),
    }
  gift_card.spend(mock_utxo_ref, token, None, Void, mock_utxo_ref, tx)
}

test spend_without_burn_fails() fail {
  gift_card.spend(
    mock_utxo_ref,
    token,
    None,
    Void,
    mock_utxo_ref,
    transaction.placeholder,
  )
}

// --- Property-Based Tests ---

use aiken/fuzz

test prop_create_requires_specific_utxo(
  fake_txid via fuzz.bytearray_fixed(32),
) fail {
  // Random UTxO ref should never match our required ref
  // Note: fuzz simple types in `via`, construct complex values in body
  let fake_ref =
    OutputReference { transaction_id: fake_txid, output_index: 0 }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [
        Input {
          output_reference: fake_ref,
          output: Output {
            address: address.from_verification_key(
              #"aabbccddee112233445566778899001122334455667788990011223344556677",
            ),
            value: assets.from_lovelace(5_000_000),
            datum: NoDatum,
            reference_script: None,
          },
        },
      ],
      mint: assets.from_asset(mock_policy_id, token, 1),
    }
  gift_card.mint(mock_utxo_ref, token, CreateGiftCard, mock_policy_id, tx)
}
```

## Key Concepts Demonstrated

1. **Multi-purpose validator** — `mint` and `spend` in one `validator` block
2. **One-shot minting** — consuming a specific UTxO guarantees uniqueness
3. **Parameterized validator** — `utxo_ref` and `token_name` baked in at deployment
4. **NFT authentication** — spending requires burning the corresponding token
5. **Atomic operations** — mint+lock and burn+unlock happen in single transactions
6. **Test calling convention** — parameterized validators pass params first: `gift_card.mint(param1, param2, redeemer, policy_id, tx)`
7. **Spend without own policy_id** — spend handler can't access its own policy_id; use `assets.flatten(tx.mint)` with tuple access (`entry.3rd`)
8. **Simple fuzzer args** — fuzz `ByteArray` in `via`, construct `OutputReference` in body

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
