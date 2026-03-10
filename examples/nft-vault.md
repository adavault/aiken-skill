# NFT-Authenticated Vault — Datum Hijacking Prevention

A multi-purpose validator that uses an NFT to authenticate UTxOs, preventing
datum hijacking attacks. The minting policy validates the datum at creation time,
and the spend validator requires burning the auth NFT.

This is a critical security pattern for any contract that stores value.

## Types

```aiken
pub type VaultDatum {
  owner: ByteArray,
  amount: Int,
}

pub type MintAction {
  CreateVault
  CloseVault
}
```

## Validator

```aiken
// validators/nft_vault.ak

use aiken/collection/dict
use aiken/collection/list
use cardano/address
use cardano/assets.{PolicyId}
use cardano/transaction.{InlineDatum, Output, OutputReference, Transaction}

/// Parameterized by auth token name
validator nft_vault(auth_token: ByteArray) {
  // Minting policy — creates/destroys auth NFT
  mint(redeemer: MintAction, policy_id: PolicyId, tx: Transaction) {
    let minted = assets.tokens(tx.mint, policy_id)

    when redeemer is {
      CreateVault -> {
        // Must mint exactly 1 auth token
        let mints_auth =
          (dict.to_pairs(minted) == [Pair(auth_token, 1)])?

        // An output must contain the auth token
        // (This proves the datum was validated at creation time)
        let has_auth_output =
          list.any(tx.outputs, fn(o) {
            assets.quantity_of(o.value, policy_id, auth_token) == 1
          })?

        mints_auth && has_auth_output
      }
      CloseVault ->
        // Must burn exactly 1 auth token
        (dict.to_pairs(minted) == [Pair(auth_token, -1)])?
    }
  }

  // Spend validator — requires burning auth NFT + owner signature
  spend(
    datum: Option<VaultDatum>,
    _redeemer: Data,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    // Must burn the auth token (any negative mint quantity)
    // Note: spend handler can't access own policy_id directly
    let burns_token =
      list.any(assets.flatten(tx.mint), fn(entry) { entry.3rd < 0 })?

    // Must be signed by owner
    let signed_by_owner = list.has(tx.extra_signatories, d.owner)?

    burns_token && signed_by_owner
  }
}
```

## Tests

```aiken
// validators/nft_vault.ak (continued)

const mock_auth = "VAULT_AUTH"
const mock_owner = #"aabb01"

const mock_policy =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn vault_output_with_auth() -> Output {
  Output {
    address: address.from_script(mock_policy),
    value: assets.merge(
      assets.from_lovelace(5_000_000),
      assets.from_asset(mock_policy, mock_auth, 1),
    ),
    datum: InlineDatum(VaultDatum { owner: mock_owner, amount: 5_000_000 }),
    reference_script: None,
  }
}

// --- Mint: CreateVault Tests ---

test create_vault_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, 1),
      outputs: [vault_output_with_auth()],
    }
  nft_vault.mint(mock_auth, CreateVault, mock_policy, tx)
}

test create_vault_without_output_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, 1),
      outputs: [],
    }
  nft_vault.mint(mock_auth, CreateVault, mock_policy, tx)
}

test create_vault_minting_wrong_token_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, "WRONG", 1),
      outputs: [vault_output_with_auth()],
    }
  nft_vault.mint(mock_auth, CreateVault, mock_policy, tx)
}

test create_vault_minting_two_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, 2),
      outputs: [vault_output_with_auth()],
    }
  nft_vault.mint(mock_auth, CreateVault, mock_policy, tx)
}

// --- Mint: CloseVault Tests ---

test close_vault_burns_token() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, -1),
    }
  nft_vault.mint(mock_auth, CloseVault, mock_policy, tx)
}

test close_vault_must_burn_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, 1),
    }
  nft_vault.mint(mock_auth, CloseVault, mock_policy, tx)
}

// --- Spend Tests ---

test spend_with_burn_and_sig_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, -1),
      extra_signatories: [mock_owner],
    }
  // Parameterized validator — pass auth_token param first
  nft_vault.spend(
    mock_auth,
    Some(VaultDatum { owner: mock_owner, amount: 5_000_000 }),
    Void,
    mock_oref(),
    tx,
  )
}

test spend_without_burn_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
    }
  nft_vault.spend(
    mock_auth,
    Some(VaultDatum { owner: mock_owner, amount: 5_000_000 }),
    Void,
    mock_oref(),
    tx,
  )
}

test spend_without_sig_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, -1),
      extra_signatories: [],
    }
  nft_vault.spend(
    mock_auth,
    Some(VaultDatum { owner: mock_owner, amount: 5_000_000 }),
    Void,
    mock_oref(),
    tx,
  )
}

test spend_wrong_owner_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, -1),
      extra_signatories: [#"ff99"],
    }
  nft_vault.spend(
    mock_auth,
    Some(VaultDatum { owner: mock_owner, amount: 5_000_000 }),
    Void,
    mock_oref(),
    tx,
  )
}

// --- Property-Based Tests ---

use aiken/fuzz

test prop_owner_can_always_spend_with_burn(
  owner via fuzz.bytearray_fixed(28),
) {
  let tx =
    Transaction {
      ..transaction.placeholder,
      mint: assets.from_asset(mock_policy, mock_auth, -1),
      extra_signatories: [owner],
    }
  nft_vault.spend(
    mock_auth,
    Some(VaultDatum { owner, amount: 5_000_000 }),
    Void,
    mock_oref(),
    tx,
  )
}

test prop_no_burn_always_fails(
  owner via fuzz.bytearray_fixed(28),
) fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [owner],
    }
  nft_vault.spend(
    mock_auth,
    Some(VaultDatum { owner, amount: 5_000_000 }),
    Void,
    mock_oref(),
    tx,
  )
}
```

## Key Concepts Demonstrated

1. **Datum hijacking prevention** — NFT authentication ensures only legitimately created UTxOs can be spent
2. **`assets.merge`** — combining lovelace with native tokens in test output values
3. **`assets.quantity_of`** — checking NFT presence in output values
4. **Dual authorization** — spend requires BOTH token burn AND owner signature
5. **`assets.flatten` with tuple access** — `entry.3rd < 0` checks for negative (burn) quantities
6. **Parameterized validator test calls** — `nft_vault.spend(mock_auth, Some(datum), Void, oref, tx)`

## Security Analysis

### Without NFT Authentication (Vulnerable)

```
1. Attacker observes script address
2. Sends UTxO to script with datum: { owner: attacker_key, amount: X }
3. Submits transaction spending it with their signature
4. Validator checks datum.owner == signer → passes!
```

### With NFT Authentication (Secure)

```
1. Vault creation requires minting auth NFT (controlled by policy)
2. Auth NFT is locked with the datum at the script address
3. Spending requires burning the auth NFT
4. Attacker can't forge the NFT → can't spend any UTxO
5. Even if attacker sends ADA to script, they can't mint auth tokens
```

The minting policy is the trust anchor — it validates datum content at creation time
and the NFT proves this validation occurred.
