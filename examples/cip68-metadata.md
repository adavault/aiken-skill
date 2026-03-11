# CIP-68 Metadata — Rich Metadata with Paired Tokens

CIP-68 rich metadata pattern with paired reference and user tokens. A single
`cip68_nft` validator handles both minting and spending:
- Mint handler: creates or burns reference + user token pairs with CIP-68 label prefixes
- Spend handler: allows admin to update the reference token's inline datum metadata

The reference token (label 222) carries the metadata as an inline datum at a
script address. The user token (label 333) is held in the owner's wallet.

## Types

```aiken
// CIP-68 label prefixes (4 bytes each)
const ref_label = #"000de140"  // (222) reference NFT
const user_label = #"0014df10"  // (333) user NFT

/// CIP-68 Rich Metadata — stored as inline datum on the reference token.
pub type CIP68Datum {
  metadata: Data,
  version: Int,
  extra: Data,
}

/// Actions for the CIP-68 minting policy and spend validator.
pub type CIP68Action {
  /// Mint a reference + user token pair
  Mint { token_name: ByteArray }
  /// Update the reference token's datum (spend handler)
  UpdateMetadata
  /// Burn both reference and user tokens
  Burn { token_name: ByteArray }
}
```

## Validator

```aiken
// validators/cip68_metadata.ak

use aiken/collection/dict
use aiken/collection/list
use aiken/primitive/bytearray
use cardano/assets.{PolicyId}
use cardano/transaction.{InlineDatum, OutputReference, Transaction}

/// CIP-68 NFT validator — parameterized by admin key hash.
///
/// Mint handler: creates or burns reference + user token pairs.
/// Spend handler: allows metadata updates on the reference token.
validator cip68_nft(admin: ByteArray) {
  mint(redeemer: CIP68Action, policy_id: PolicyId, tx: Transaction) {
    let minted = assets.tokens(tx.mint, policy_id)
    let pairs = dict.to_pairs(minted)

    when redeemer is {
      Mint { token_name } -> {
        // Admin must sign
        let admin_signed = list.has(tx.extra_signatories, admin)?

        // Must mint exactly 1 reference token and 1 user token
        let ref_name = bytearray.concat(ref_label, token_name)
        let usr_name = bytearray.concat(user_label, token_name)
        let correct_mint =
          (pairs == [Pair(ref_name, 1), Pair(usr_name, 1)])?

        // Reference token output must exist with an inline datum
        let has_ref_output =
          list.any(
            tx.outputs,
            fn(o) {
              let has_token =
                assets.quantity_of(o.value, policy_id, ref_name) == 1
              when o.datum is {
                InlineDatum(_) -> has_token
                _ -> False
              }
            },
          )?

        admin_signed && correct_mint && has_ref_output
      }

      Burn { token_name } -> {
        // Admin must sign
        let admin_signed = list.has(tx.extra_signatories, admin)?

        // Must burn exactly 1 reference token and 1 user token
        let ref_name = bytearray.concat(ref_label, token_name)
        let usr_name = bytearray.concat(user_label, token_name)
        let correct_burn =
          (pairs == [Pair(ref_name, -1), Pair(usr_name, -1)])?

        admin_signed && correct_burn
      }

      // UpdateMetadata is only valid for the spend handler
      UpdateMetadata -> False
    }
  }

  spend(
    datum: Option<CIP68Datum>,
    _redeemer: CIP68Action,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(_d) = datum

    // Admin must sign
    let admin_signed = list.has(tx.extra_signatories, admin)?

    // Find our own input to determine the script address and reference token
    expect Some(own_input) = transaction.find_input(tx.inputs, oref)
    let script_cred = own_input.output.address.payment_credential

    // Continuing output must exist at the same script address with an inline datum
    let has_continuing_output =
      list.any(
        tx.outputs,
        fn(o) {
          let same_address = o.address.payment_credential == script_cred
          when o.datum is {
            InlineDatum(_) -> same_address
            _ -> False
          }
        },
      )?

    admin_signed && has_continuing_output
  }
}
```

## Tests

```aiken
// validators/cip68_metadata.ak (continued)

const mock_admin =
  #"aabbccddee112233445566778899001122334455667788990011aabb"

const mock_non_admin =
  #"112233445566778899aabbccddeeff00112233445566778899001122"

const mock_policy =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

const mock_token_name = "MyNFT"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_ref_name() -> ByteArray {
  bytearray.concat(ref_label, mock_token_name)
}

fn mock_usr_name() -> ByteArray {
  bytearray.concat(user_label, mock_token_name)
}

fn script_addr() -> address.Address {
  address.from_script(mock_policy)
}

fn mock_metadata() -> CIP68Datum {
  CIP68Datum { metadata: "test metadata", version: 1, extra: Void }
}

fn mock_updated_metadata() -> CIP68Datum {
  CIP68Datum { metadata: "updated metadata", version: 2, extra: Void }
}

fn ref_token_output(datum: CIP68Datum) -> Output {
  Output {
    address: script_addr(),
    value: assets.from_lovelace(2_000_000)
      |> assets.add(mock_policy, mock_ref_name(), 1),
    datum: InlineDatum(datum),
    reference_script: None,
  }
}

fn mint_value() -> assets.Value {
  assets.from_asset(mock_policy, mock_ref_name(), 1)
    |> assets.add(mock_policy, mock_usr_name(), 1)
}

fn burn_value() -> assets.Value {
  assets.from_asset(mock_policy, mock_ref_name(), -1)
    |> assets.add(mock_policy, mock_usr_name(), -1)
}

fn ref_input(datum: CIP68Datum) -> Input {
  Input {
    output_reference: mock_oref(),
    output: ref_token_output(datum),
  }
}

// -- Mint Tests --

test mint_creates_both_tokens_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: mint_value(),
      outputs: [ref_token_output(mock_metadata())],
      extra_signatories: [mock_admin],
    }
  cip68_nft.mint(mock_admin, Mint { token_name: mock_token_name }, mock_policy, tx)
}

test mint_without_admin_signature_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: mint_value(),
      outputs: [ref_token_output(mock_metadata())],
      extra_signatories: [mock_non_admin],
    }
  cip68_nft.mint(mock_admin, Mint { token_name: mock_token_name }, mock_policy, tx)
}

test mint_only_ref_token_missing_user_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_ref_name(), 1),
      outputs: [ref_token_output(mock_metadata())],
      extra_signatories: [mock_admin],
    }
  cip68_nft.mint(mock_admin, Mint { token_name: mock_token_name }, mock_policy, tx)
}

test mint_without_ref_output_fails() fail {
  let user_only_output =
    Output {
      address: address.from_verification_key(mock_admin),
      value: assets.from_lovelace(2_000_000)
        |> assets.add(mock_policy, mock_usr_name(), 1),
      datum: transaction.NoDatum,
      reference_script: None,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: mint_value(),
      outputs: [user_only_output],
      extra_signatories: [mock_admin],
    }
  cip68_nft.mint(mock_admin, Mint { token_name: mock_token_name }, mock_policy, tx)
}

// -- Burn Tests --

test burn_both_tokens_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: burn_value(),
      extra_signatories: [mock_admin],
    }
  cip68_nft.mint(mock_admin, Burn { token_name: mock_token_name }, mock_policy, tx)
}

test burn_without_admin_signature_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: burn_value(),
      extra_signatories: [mock_non_admin],
    }
  cip68_nft.mint(mock_admin, Burn { token_name: mock_token_name }, mock_policy, tx)
}

test burn_only_ref_token_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_ref_name(), -1),
      extra_signatories: [mock_admin],
    }
  cip68_nft.mint(mock_admin, Burn { token_name: mock_token_name }, mock_policy, tx)
}

// -- Spend: UpdateMetadata Tests --

test update_metadata_with_admin_sig_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [ref_input(mock_metadata())],
      outputs: [ref_token_output(mock_updated_metadata())],
      extra_signatories: [mock_admin],
    }
  cip68_nft.spend(
    mock_admin,
    Some(mock_metadata()),
    UpdateMetadata,
    mock_oref(),
    tx,
  )
}

test update_metadata_without_admin_sig_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [ref_input(mock_metadata())],
      outputs: [ref_token_output(mock_updated_metadata())],
      extra_signatories: [mock_non_admin],
    }
  cip68_nft.spend(
    mock_admin,
    Some(mock_metadata()),
    UpdateMetadata,
    mock_oref(),
    tx,
  )
}

test update_metadata_without_continuing_output_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [ref_input(mock_metadata())],
      outputs: [],
      extra_signatories: [mock_admin],
    }
  cip68_nft.spend(
    mock_admin,
    Some(mock_metadata()),
    UpdateMetadata,
    mock_oref(),
    tx,
  )
}

// -- Property-Based Tests --

test prop_any_token_name_with_admin_sig_mints(
  name via fuzz.bytearray_between(1, 32),
) {
  let ref_name = bytearray.concat(ref_label, name)
  let usr_name = bytearray.concat(user_label, name)
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, ref_name, 1)
        |> assets.add(mock_policy, usr_name, 1),
      outputs: [
        Output {
          address: script_addr(),
          value: assets.from_lovelace(2_000_000)
            |> assets.add(mock_policy, ref_name, 1),
          datum: InlineDatum(mock_metadata()),
          reference_script: None,
        },
      ],
      extra_signatories: [mock_admin],
    }
  cip68_nft.mint(mock_admin, Mint { token_name: name }, mock_policy, tx)
}
```

## Key Concepts Demonstrated

1. **CIP-68 label prefixes** — `#"000de140"` (222) for reference NFT, `#"0014df10"` (333) for user NFT
2. **Paired token minting** — `dict.to_pairs(minted)` enforces exactly one reference + one user token
3. **Reference token carries metadata** — inline datum on the reference token output at the script address
4. **`bytearray.concat`** — constructs CIP-68 token names from label prefix + base name
5. **Shared redeemer type across handlers** — `CIP68Action` used by both mint and spend, with `UpdateMetadata` rejected in mint
6. **Continuing output for metadata update** — spend handler requires an output back to the same script with `InlineDatum`
7. **Parameterized validator** — `cip68_nft(admin)` bakes in the admin key hash at compile time
