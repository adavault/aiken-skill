# Notary — Proof-of-Existence on Cardano

A proof-of-existence notary service. No equivalent exists on Cardano today
(Bitcoin has OP_RETURN-based services like OpenTimestamps).

Each notarization mints a unique NFT whose datum stores a document hash, optional
URI, and notarizer identity. The transaction's validity range — enforced by the
Cardano ledger — serves as the cryptographic timestamp. No trusted third-party
timestamp required.

Parameterized per operator (SPO/notary service), so each deploys their own
instance with their own fee.

## Types

```aiken
pub type NotaryDatum {
  document_hash: ByteArray,
  hash_algorithm: ByteArray,     // "SHA-256", "BLAKE2b-256", etc.
  uri: Option<ByteArray>,        // Optional: where the source document lives
  notarizer: ByteArray,          // Who notarized it
  description: Option<ByteArray>, // Optional context
}

pub type NotaryRedeemer {
  Notarize { output_ref: OutputReference }  // Mint a notarization NFT
  Burn                                       // Revoke a notarization
}
```

## Validator

```aiken
// validators/notary.ak

use aiken/cbor
use aiken/collection/dict
use aiken/collection/list
use aiken/crypto
use cardano/address
use cardano/assets.{PolicyId}
use cardano/transaction.{InlineDatum, Input, NoDatum, Output, OutputReference,
  Transaction}

/// Derive a unique token name from an output reference.
/// CBOR-serialise then blake2b-256 hash → 32-byte unique name.
fn token_name_from_ref(oref: OutputReference) -> ByteArray {
  oref
    |> cbor.serialise
    |> crypto.blake2b_256
}

fn notarizer_address(notarizer: ByteArray) -> address.Address {
  address.from_verification_key(notarizer)
}

/// Parameterized mint validator for proof-of-existence notarization.
///
/// Parameters:
///   notarizer    — verification key hash of the authorized notarizer (28 bytes)
///   fee_lovelace — minimum lovelace fee paid to the notarizer per notarization
validator notary(notarizer: ByteArray, fee_lovelace: Int) {
  mint(redeemer: NotaryRedeemer, policy_id: PolicyId, tx: Transaction) {
    when redeemer is {
      Notarize { output_ref } -> {
        let expected_token_name = token_name_from_ref(output_ref)
        let minted = assets.tokens(tx.mint, policy_id)
        let notarizer_addr = notarizer_address(notarizer)

        // 1. Notarizer must sign the transaction
        let signed =
          list.has(tx.extra_signatories, notarizer)?

        // 2. The referenced UTxO must be consumed (one-shot guarantee)
        let consumes_ref =
          list.any(
            tx.inputs,
            fn(i) { i.output_reference == output_ref },
          )?

        // 3. Exactly one token minted under this policy with the derived name
        let mints_one =
          (dict.to_pairs(minted) == [Pair(expected_token_name, 1)])?

        // 4. Fee paid to notarizer address
        let fee_paid =
          list.any(
            tx.outputs,
            fn(o) {
              o.address == notarizer_addr && assets.lovelace_of(o.value) >= fee_lovelace
            },
          )?

        // 5. An output exists carrying the NFT with a valid inline NotaryDatum
        let has_datum_output =
          list.any(
            tx.outputs,
            fn(o) {
              let has_nft =
                assets.quantity_of(o.value, policy_id, expected_token_name) == 1
              when o.datum is {
                InlineDatum(raw) -> {
                  expect _datum: NotaryDatum = raw
                  has_nft
                }
                _ -> False
              }
            },
          )?

        signed && consumes_ref && mints_one && fee_paid && has_datum_output
      }

      Burn -> {
        let minted = assets.tokens(tx.mint, policy_id)
        let pairs = dict.to_pairs(minted)

        // Exactly one token burned (quantity -1)
        let burns_one =
          when pairs is {
            [Pair(_, qty)] -> (qty == -1)?
            _ -> fail @"must burn exactly one token"
          }

        // Notarizer must sign the burn
        let signed = list.has(tx.extra_signatories, notarizer)?

        burns_one && signed
      }
    }
  }
}
```

## Tests

```aiken
// validators/notary.ak (continued)

const mock_notarizer =
  #"aabbccddee112233445566778899001122334455667788990011aabb"

const mock_fee = 2_000_000

const mock_policy_id =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

const mock_output_ref =
  OutputReference {
    transaction_id: #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    output_index: 0,
  }

const mock_document_hash =
  #"e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"

fn mock_token_name() -> ByteArray {
  token_name_from_ref(mock_output_ref)
}

fn mock_consumed_input() -> Input {
  Input {
    output_reference: mock_output_ref,
    output: Output {
      address: address.from_verification_key(mock_notarizer),
      value: assets.from_lovelace(5_000_000),
      datum: NoDatum,
      reference_script: None,
    },
  }
}

fn mock_notary_datum() -> NotaryDatum {
  NotaryDatum {
    document_hash: mock_document_hash,
    hash_algorithm: "SHA-256",
    uri: None,
    notarizer: mock_notarizer,
    description: None,
  }
}

fn mock_nft_output(policy_id: PolicyId, token_name: ByteArray) -> Output {
  Output {
    address: address.from_script(mock_policy_id),
    value: assets.from_lovelace(2_000_000)
      |> assets.add(policy_id, token_name, 1),
    datum: InlineDatum(mock_notary_datum()),
    reference_script: None,
  }
}

fn mock_fee_output() -> Output {
  Output {
    address: address.from_verification_key(mock_notarizer),
    value: assets.from_lovelace(mock_fee),
    datum: NoDatum,
    reference_script: None,
  }
}

fn mock_fee_output_amount(amount: Int) -> Output {
  Output {
    address: address.from_verification_key(mock_notarizer),
    value: assets.from_lovelace(amount),
    datum: NoDatum,
    reference_script: None,
  }
}

fn valid_notarize_tx() -> Transaction {
  let tn = mock_token_name()
  Transaction {
    ..transaction.placeholder,
    inputs: [mock_consumed_input()],
    outputs: [mock_nft_output(mock_policy_id, tn), mock_fee_output()],
    mint: assets.from_asset(mock_policy_id, tn, 1),
    extra_signatories: [mock_notarizer],
  }
}

// -- Happy path --

test notarize_succeeds() {
  let redeemer = Notarize { output_ref: mock_output_ref }
  notary.mint(mock_notarizer, mock_fee, redeemer, mock_policy_id, valid_notarize_tx())
}

test notarize_with_uri_succeeds() {
  let tn = mock_token_name()
  let datum_with_uri =
    NotaryDatum {
      document_hash: mock_document_hash,
      hash_algorithm: "SHA-256",
      uri: Some("ipfs://QmTest123"),
      notarizer: mock_notarizer,
      description: None,
    }
  let nft_output =
    Output {
      address: address.from_script(mock_policy_id),
      value: assets.from_lovelace(2_000_000)
        |> assets.add(mock_policy_id, tn, 1),
      datum: InlineDatum(datum_with_uri),
      reference_script: None,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [nft_output, mock_fee_output()],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

test notarize_overpay_fee_succeeds() {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [
        mock_nft_output(mock_policy_id, tn),
        mock_fee_output_amount(10_000_000),
      ],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

// -- Failure paths --

test notarize_without_signature_fails() fail {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [mock_nft_output(mock_policy_id, tn), mock_fee_output()],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

test notarize_underpay_fee_fails() fail {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [
        mock_nft_output(mock_policy_id, tn),
        mock_fee_output_amount(1_000_000),
      ],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

test notarize_wrong_utxo_fails() fail {
  let tn = mock_token_name()
  let wrong_ref =
    OutputReference {
      transaction_id: #"bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
      output_index: 99,
    }
  let wrong_input =
    Input {
      output_reference: wrong_ref,
      output: Output {
        address: address.from_verification_key(mock_notarizer),
        value: assets.from_lovelace(5_000_000),
        datum: NoDatum,
        reference_script: None,
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [wrong_input],
      outputs: [mock_nft_output(mock_policy_id, tn), mock_fee_output()],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

test notarize_double_mint_fails() fail {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [mock_nft_output(mock_policy_id, tn), mock_fee_output()],
      mint: assets.from_asset(mock_policy_id, tn, 2),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

// -- Burn --

test burn_succeeds() {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy_id, tn, -1),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(mock_notarizer, mock_fee, Burn, mock_policy_id, tx)
}

test burn_without_signature_fails() fail {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy_id, tn, -1),
      extra_signatories: [],
    }
  notary.mint(mock_notarizer, mock_fee, Burn, mock_policy_id, tx)
}

// -- Property tests --

test prop_notarize_always_requires_signature(
  stranger via fuzz.bytearray_fixed(28),
) fail {
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [mock_nft_output(mock_policy_id, tn), mock_fee_output()],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [stranger],
    }
  notary.mint(
    mock_notarizer, mock_fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}

test prop_any_fee_amount_works(
  params via fuzz.both(
    fuzz.int_between(1, 100_000_000),
    fuzz.int_between(1, 100_000_000),
  ),
) {
  let (fee, extra) = params
  let payment = fee + extra
  let tn = mock_token_name()
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_consumed_input()],
      outputs: [
        mock_nft_output(mock_policy_id, tn),
        mock_fee_output_amount(payment),
      ],
      mint: assets.from_asset(mock_policy_id, tn, 1),
      extra_signatories: [mock_notarizer],
    }
  notary.mint(
    mock_notarizer, fee,
    Notarize { output_ref: mock_output_ref }, mock_policy_id, tx,
  )
}
```

## Key Concepts Demonstrated

1. **One-shot NFT via UTxO reference** — `token_name_from_ref(output_ref)` derives the token name from `blake2b_256(cbor.serialise(oref))`. Since a UTxO can only be consumed once, the NFT is guaranteed unique.
2. **`cbor.serialise()` + `crypto.blake2b_256()`** — composable serialisation-to-hash pipeline for deterministic token names.
3. **Validity range as timestamp** — the transaction's validity interval is set by the submitter and enforced by the Cardano ledger. No timestamp field needed in the datum — the block's slot provides the proof-of-existence time.
4. **Parameterized mint validator** — `notary(notarizer, fee_lovelace)` lets each operator deploy independently with their own identity and fee.
5. **Inline datum enforcement** — `expect _datum: NotaryDatum = raw` validates the datum structure without inspecting field values. The minting policy ensures datum is well-formed at creation time.
6. **Fee collection pattern** — `list.any(tx.outputs, fn(o) { o.address == notarizer_addr && lovelace >= fee })` verifies payment within the validator.
7. **Burn with authorization** — notarizer can revoke a notarization by burning the NFT. Useful for corrections or legal requirements.

## Comparison: Bitcoin OP_RETURN vs Cardano Notary

| Feature | Bitcoin (OP_RETURN) | Cardano (This Pattern) |
|---------|-------------------|----------------------|
| Data capacity | 80 bytes | Full structured datum |
| Logic | None | Validator-enforced rules |
| Timestamp | Block time (miner-set) | Validity range (ledger-enforced) |
| Receipt | Transaction hash | Transferable NFT |
| Queryable | Raw bytes only | Typed datum via indexer |
| Revocation | Not possible | Burn the NFT |
| Fee model | Miner fee only | Configurable operator fee |
| Verification | Parse raw OP_RETURN | Read NFT datum |

## Security Notes

- **One-shot guarantee** — the consumed UTxO reference makes the token name unique and non-reproducible. No second mint is possible with the same name.
- **Datum is validated structurally** — `expect _datum: NotaryDatum = raw` ensures correct CBOR encoding. Malformed data cannot be stored.
- **Fee is enforced on-chain** — cannot notarize without paying the operator.
- **Burn is restricted** — only the notarizer can burn, preventing third parties from revoking proofs.
- **Document hash is not verified** — the contract stores whatever hash is provided. Off-chain verification (comparing file hash to on-chain hash) is the user's responsibility.
- **URI is optional and unverified** — the URI may go stale. For permanent storage, use IPFS or Arweave URIs.
