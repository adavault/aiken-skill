# Merkle Proof — On-Chain Verification for Whitelists/Airdrops

On-chain Merkle proof verification using Blake2b-256 hashing. Users claim by
providing their leaf and a proof path. The validator hashes through the proof
steps and compares the computed root against the stored Merkle root. The owner
can update the root to add/remove participants.

Useful for whitelists, airdrops, and any scenario where a large set of eligible
addresses must be verified on-chain without storing them all in the datum.

## Types

```aiken
pub type MerkleDatum {
  merkle_root: ByteArray,
  owner: ByteArray,
}

pub type ProofStep {
  Left { hash: ByteArray }
  Right { hash: ByteArray }
}

pub type MerkleRedeemer {
  Claim { leaf: ByteArray, proof: List<ProofStep> }
  UpdateRoot { new_root: ByteArray }
}
```

## Validator

```aiken
// validators/merkle_proof.ak

use aiken/collection/list
use aiken/crypto
use aiken/primitive/bytearray
use cardano/transaction.{InlineDatum, OutputReference, Transaction}

/// Verify a Merkle proof: hash the leaf, fold through the proof steps,
/// and check the computed root matches the expected root.
fn verify_proof(
  leaf: ByteArray,
  proof: List<ProofStep>,
  root: ByteArray,
) -> Bool {
  let computed_root =
    list.foldl(
      proof,
      crypto.blake2b_256(leaf),
      fn(step, current) {
        when step is {
          Left { hash } ->
            crypto.blake2b_256(bytearray.concat(hash, current))
          Right { hash } ->
            crypto.blake2b_256(bytearray.concat(current, hash))
        }
      },
    )
  computed_root == root
}

validator merkle_proof {
  spend(
    datum: Option<MerkleDatum>,
    redeemer: MerkleRedeemer,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      Claim { leaf, proof } -> {
        // 1. Proof must verify against the stored Merkle root
        let valid_proof = verify_proof(leaf, proof, d.merkle_root)?

        // 2. The leaf must be the claimant's key hash — prevents replaying
        //    someone else's proof
        let claimant_signed = list.has(tx.extra_signatories, leaf)?

        valid_proof && claimant_signed
      }

      UpdateRoot { new_root } -> {
        // 1. Owner must sign
        let owner_signed = list.has(tx.extra_signatories, d.owner)?

        // 2. Continuing output to same script with updated root
        expect Some(own_input) = transaction.find_input(tx.inputs, oref)
        let script_cred = own_input.output.address.payment_credential
        let script_outputs =
          list.filter(tx.outputs, fn(o) {
            o.address.payment_credential == script_cred
          })
        expect [continuing_output] = script_outputs
        expect InlineDatum(raw) = continuing_output.datum
        expect new_datum: MerkleDatum = raw

        let root_updated = (new_datum.merkle_root == new_root)?
        let owner_preserved = (new_datum.owner == d.owner)?

        owner_signed && root_updated && owner_preserved
      }
    }
  }
}
```

## Tests

```aiken
// validators/merkle_proof.ak (continued)

// Four leaves for a small Merkle tree
const leaf_a = #"aa"
const leaf_b = #"bb"
const leaf_c = #"cc"
const leaf_d = #"dd"

const mock_owner =
  #"aabbccddee112233445566778899001122334455667788990011aabb"

const mock_non_owner =
  #"ff00112233445566778899aabbccddeeff00112233445566778899ff"

const mock_script_hash =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

// -- Merkle tree construction helpers --

/// Hash a leaf value.
fn h(x: ByteArray) -> ByteArray {
  crypto.blake2b_256(x)
}

/// Hash two child nodes together (left ++ right).
fn h2(left: ByteArray, right: ByteArray) -> ByteArray {
  crypto.blake2b_256(bytearray.concat(left, right))
}

/// Build the root of a 4-leaf Merkle tree [A, B, C, D].
///
///       Root
///      /    \
///   H(AB)  H(CD)
///   / \    / \
/// H(A) H(B) H(C) H(D)
fn merkle_root_4() -> ByteArray {
  let hab = h2(h(leaf_a), h(leaf_b))
  let hcd = h2(h(leaf_c), h(leaf_d))
  h2(hab, hcd)
}

/// Proof for leaf A: [Right(H(B)), Right(H(CD))]
fn proof_for_a() -> List<ProofStep> {
  let hcd = h2(h(leaf_c), h(leaf_d))
  [Right { hash: h(leaf_b) }, Right { hash: hcd }]
}

/// Proof for leaf C: [Right(H(D)), Left(H(AB))]
fn proof_for_c() -> List<ProofStep> {
  let hab = h2(h(leaf_a), h(leaf_b))
  [Right { hash: h(leaf_d) }, Left { hash: hab }]
}

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn script_addr() -> address.Address {
  address.from_script(mock_script_hash)
}

fn mock_datum() -> MerkleDatum {
  MerkleDatum { merkle_root: merkle_root_4(), owner: mock_owner }
}

fn mock_script_input(datum: MerkleDatum) -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: script_addr(),
      value: assets.from_lovelace(5_000_000),
      datum: InlineDatum(datum),
      reference_script: None,
    },
  }
}

fn mock_continuing_output(datum: MerkleDatum) -> Output {
  Output {
    address: script_addr(),
    value: assets.from_lovelace(5_000_000),
    datum: InlineDatum(datum),
    reference_script: None,
  }
}

// -- Claim Tests --

test claim_leaf_a_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      extra_signatories: [leaf_a],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    Claim { leaf: leaf_a, proof: proof_for_a() },
    mock_oref(),
    tx,
  )
}

test claim_leaf_c_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      extra_signatories: [leaf_c],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    Claim { leaf: leaf_c, proof: proof_for_c() },
    mock_oref(),
    tx,
  )
}

test claim_invalid_proof_fails() fail {
  // Use the wrong sibling hash — swap H(B) for H(D)
  let hcd = h2(h(leaf_c), h(leaf_d))
  let bad_proof = [Right { hash: h(leaf_d) }, Right { hash: hcd }]
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      extra_signatories: [leaf_a],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    Claim { leaf: leaf_a, proof: bad_proof },
    mock_oref(),
    tx,
  )
}

test claim_wrong_leaf_fails() fail {
  // Use leaf_a's proof but claim with leaf_b
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      extra_signatories: [leaf_b],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    Claim { leaf: leaf_b, proof: proof_for_a() },
    mock_oref(),
    tx,
  )
}

test claim_without_signature_fails() fail {
  // Valid proof but claimant does not sign the transaction
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      extra_signatories: [],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    Claim { leaf: leaf_a, proof: proof_for_a() },
    mock_oref(),
    tx,
  )
}

// -- UpdateRoot Tests --

test update_root_by_owner_succeeds() {
  let new_root = h(#"deadbeef")
  let new_datum = MerkleDatum { merkle_root: new_root, owner: mock_owner }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      outputs: [mock_continuing_output(new_datum)],
      extra_signatories: [mock_owner],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    UpdateRoot { new_root },
    mock_oref(),
    tx,
  )
}

test update_root_by_non_owner_fails() fail {
  let new_root = h(#"deadbeef")
  let new_datum = MerkleDatum { merkle_root: new_root, owner: mock_owner }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      outputs: [mock_continuing_output(new_datum)],
      extra_signatories: [mock_non_owner],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    UpdateRoot { new_root },
    mock_oref(),
    tx,
  )
}

test update_root_owner_changed_fails() fail {
  // Attempt to change the owner during root update
  let new_root = h(#"deadbeef")
  let tampered_datum =
    MerkleDatum { merkle_root: new_root, owner: mock_non_owner }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_script_input(mock_datum())],
      outputs: [mock_continuing_output(tampered_datum)],
      extra_signatories: [mock_owner],
    }
  merkle_proof.spend(
    Some(mock_datum()),
    UpdateRoot { new_root },
    mock_oref(),
    tx,
  )
}

// -- Property-Based Tests --

test prop_verify_proof_deterministic(
  seed via fuzz.bytearray_between(1, 64),
) {
  // Same inputs always produce the same result
  let leaf = seed
  let sibling = crypto.blake2b_256(seed)
  let proof = [Right { hash: sibling }]
  let root = h2(crypto.blake2b_256(leaf), sibling)
  verify_proof(leaf, proof, root) && verify_proof(leaf, proof, root)
}

test prop_single_leaf_tree(
  leaf via fuzz.bytearray_between(1, 64),
) {
  // A tree with one leaf: root = H(leaf), proof is empty
  let root = crypto.blake2b_256(leaf)
  verify_proof(leaf, [], root)
}
```

## Key Concepts Demonstrated

1. **Merkle proof verification** — `list.foldl` hashes through `ProofStep` list to compute the root from a leaf
2. **`Left`/`Right` proof steps** — determines concatenation order: `H(sibling ++ current)` vs `H(current ++ sibling)`
3. **`crypto.blake2b_256`** — Cardano's native hash function for building Merkle trees on-chain
4. **Leaf-as-identity** — the leaf doubles as the claimant's key hash, binding the proof to a specific signer
5. **Continuing output with updated root** — `UpdateRoot` enforces exactly one script output with new root and preserved owner
6. **`expect [continuing_output] = script_outputs`** — destructure a singleton list to ensure exactly one continuing output
7. **Compact on-chain storage** — only the 32-byte root is stored; the full tree lives off-chain
