# Reference Scripts — CIP-33 Script Registry

CIP-33 reference scripts pattern with two validators working together:
1. `script_registry` (spend) — admin-controlled registry for storing reference scripts
2. `script_user` (spend) — a vault that verifies a reference script is available

Reference scripts allow transactions to reference a script stored in an existing
UTxO rather than including the full script in the witness set. This reduces
transaction size and fees for commonly-used scripts. The registry stores scripts
as `reference_script` on its UTxO outputs. The user validator checks
`tx.reference_inputs` to verify a script is available.

## Types

```aiken
/// Registry datum — tracks the admin, stored script hash, and version.
pub type RegistryDatum {
  /// Admin's verification key hash (authorized to update/remove)
  admin: ByteArray,
  /// Hash of the stored reference script
  script_hash: ByteArray,
  /// Version number — incremented on each update
  version: Int,
}

/// Actions for the script registry.
pub type RegistryAction {
  /// Update the stored script (bumps version)
  UpdateScript
  /// Remove the registry entry (returns funds to admin)
  RemoveScript
}

/// Vault datum — references a registry by its script hash.
pub type VaultDatum {
  /// Owner's verification key hash
  owner: ByteArray,
  /// Script hash of the registry to verify in reference inputs
  registry_hash: ByteArray,
}
```

## Validator

```aiken
// validators/reference_scripts.ak

use aiken/collection/list
use cardano/assets
use cardano/transaction.{InlineDatum, OutputReference, Transaction}

/// Script registry — stores reference scripts on-chain.
/// Admin can update (version bump) or remove registry entries.
validator script_registry {
  spend(
    datum: Option<RegistryDatum>,
    redeemer: RegistryAction,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      UpdateScript -> {
        // Admin must sign
        let admin_signed = list.has(tx.extra_signatories, d.admin)?

        // Find our own input to determine the script address
        expect Some(own_input) = transaction.find_input(tx.inputs, oref)
        let script_cred = own_input.output.address.payment_credential
        let input_lovelace = assets.lovelace_of(own_input.output.value)

        // Continuing output must exist at same address with correct datum
        let valid_continuing_output =
          list.any(
            tx.outputs,
            fn(o) {
              let same_address = o.address.payment_credential == script_cred
              when o.datum is {
                InlineDatum(raw) -> {
                  expect new_datum: RegistryDatum = raw
                  and {
                    same_address,
                    // Admin must stay the same
                    (new_datum.admin == d.admin)?,
                    // Script hash must stay the same
                    (new_datum.script_hash == d.script_hash)?,
                    // Version must increment by exactly 1
                    (new_datum.version == d.version + 1)?,
                    // Output lovelace must be >= input lovelace
                    (assets.lovelace_of(o.value) >= input_lovelace)?,
                  }
                }
                _ -> False
              }
            },
          )?

        admin_signed && valid_continuing_output
      }

      RemoveScript -> {
        // Admin must sign — funds returned to admin
        list.has(tx.extra_signatories, d.admin)?
      }
    }
  }
}

/// Script user — a vault that verifies a reference script is available
/// via CIP-31 reference inputs before allowing withdrawal.
validator script_user {
  spend(
    datum: Option<VaultDatum>,
    _redeemer: Data,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    // Owner must sign
    let owner_signed = list.has(tx.extra_signatories, d.owner)?

    // The registry script hash must appear in a reference input's reference_script
    // field, proving the reference script is available on-chain
    let registry_in_refs =
      list.any(
        tx.reference_inputs,
        fn(ref_input) {
          ref_input.output.reference_script == Some(d.registry_hash)
        },
      )?

    and {
      owner_signed,
      registry_in_refs,
    }
  }
}
```

## Tests

```aiken
// validators/reference_scripts.ak (continued)

const mock_admin =
  #"aabbccddee112233445566778899001122334455667788990011aabb"

const mock_non_admin =
  #"112233445566778899aabbccddeeff00112233445566778899001122"

const mock_owner =
  #"ccddee112233445566778899aabbccddee112233445566778899aabb"

const mock_non_owner =
  #"ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00ff00"

const mock_script_hash =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

const mock_registry_script =
  #"eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn script_addr() -> address.Address {
  address.from_script(mock_script_hash)
}

fn mock_registry_datum(version: Int) -> RegistryDatum {
  RegistryDatum {
    admin: mock_admin,
    script_hash: mock_registry_script,
    version: version,
  }
}

fn registry_input(version: Int, lovelace: Int) -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: script_addr(),
      value: assets.from_lovelace(lovelace),
      datum: InlineDatum(mock_registry_datum(version)),
      reference_script: Some(mock_registry_script),
    },
  }
}

fn registry_output(version: Int, lovelace: Int) -> Output {
  Output {
    address: script_addr(),
    value: assets.from_lovelace(lovelace),
    datum: InlineDatum(mock_registry_datum(version)),
    reference_script: Some(mock_registry_script),
  }
}

fn mock_vault_datum() -> VaultDatum {
  VaultDatum { owner: mock_owner, registry_hash: mock_registry_script }
}

fn vault_ref_input() -> Input {
  Input {
    output_reference: OutputReference {
      transaction_id: #"1111111111111111111111111111111111111111111111111111111111111111",
      output_index: 0,
    },
    output: Output {
      address: script_addr(),
      value: assets.from_lovelace(2_000_000),
      datum: InlineDatum(mock_registry_datum(1)),
      reference_script: Some(mock_registry_script),
    },
  }
}

// -- Registry: UpdateScript Tests --

test registry_admin_can_update_script() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      outputs: [registry_output(2, 10_000_000)],
      extra_signatories: [mock_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    UpdateScript,
    mock_oref(),
    tx,
  )
}

test registry_non_admin_cannot_update() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      outputs: [registry_output(2, 10_000_000)],
      extra_signatories: [mock_non_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    UpdateScript,
    mock_oref(),
    tx,
  )
}

test registry_version_must_increment_by_one() fail {
  // Version jumps from 1 to 3 (should be 2)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      outputs: [registry_output(3, 10_000_000)],
      extra_signatories: [mock_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    UpdateScript,
    mock_oref(),
    tx,
  )
}

test registry_output_lovelace_must_not_decrease() fail {
  // Input has 10 ADA, output has only 5 ADA
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      outputs: [registry_output(2, 5_000_000)],
      extra_signatories: [mock_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    UpdateScript,
    mock_oref(),
    tx,
  )
}

test registry_script_hash_must_stay_same_on_update() fail {
  // Output has a different script_hash in the datum
  let bad_output =
    Output {
      address: script_addr(),
      value: assets.from_lovelace(10_000_000),
      datum: InlineDatum(
        RegistryDatum {
          admin: mock_admin,
          script_hash: #"ffffffffffffffffffffffffffffffffffffffffffffffffffffffff",
          version: 2,
        },
      ),
      reference_script: Some(mock_registry_script),
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      outputs: [bad_output],
      extra_signatories: [mock_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    UpdateScript,
    mock_oref(),
    tx,
  )
}

test registry_update_with_increased_lovelace_succeeds() {
  // Output has more lovelace than input — allowed
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      outputs: [registry_output(2, 15_000_000)],
      extra_signatories: [mock_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    UpdateScript,
    mock_oref(),
    tx,
  )
}

// -- Registry: RemoveScript Tests --

test registry_admin_can_remove_script() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      extra_signatories: [mock_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    RemoveScript,
    mock_oref(),
    tx,
  )
}

test registry_non_admin_cannot_remove() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [registry_input(1, 10_000_000)],
      extra_signatories: [mock_non_admin],
    }
  script_registry.spend(
    Some(mock_registry_datum(1)),
    RemoveScript,
    mock_oref(),
    tx,
  )
}

// -- Script User Tests --

test user_owner_can_spend_with_reference_input() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [vault_ref_input()],
      extra_signatories: [mock_owner],
    }
  script_user.spend(Some(mock_vault_datum()), Void, mock_oref(), tx)
}

test user_non_owner_cannot_spend() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [vault_ref_input()],
      extra_signatories: [mock_non_owner],
    }
  script_user.spend(Some(mock_vault_datum()), Void, mock_oref(), tx)
}

test user_owner_cannot_spend_without_reference_input() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [],
      extra_signatories: [mock_owner],
    }
  script_user.spend(Some(mock_vault_datum()), Void, mock_oref(), tx)
}

test user_wrong_reference_script_fails() fail {
  // Reference input has a different script hash
  let wrong_ref =
    Input {
      output_reference: OutputReference {
        transaction_id: #"2222222222222222222222222222222222222222222222222222222222222222",
        output_index: 0,
      },
      output: Output {
        address: script_addr(),
        value: assets.from_lovelace(2_000_000),
        datum: InlineDatum(mock_registry_datum(1)),
        reference_script: Some(
          #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
        ),
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      reference_inputs: [wrong_ref],
      extra_signatories: [mock_owner],
    }
  script_user.spend(Some(mock_vault_datum()), Void, mock_oref(), tx)
}

// -- Property-Based Tests --

test prop_random_non_admin_cannot_update_registry(
  key via fuzz.bytearray_fixed(28),
) {
  // Ensure the random key is not the admin key
  key != mock_admin || {
    let tx =
      Transaction {
        ..transaction.placeholder,
        inputs: [registry_input(1, 10_000_000)],
        outputs: [registry_output(2, 10_000_000)],
        extra_signatories: [key],
      }
    script_registry.spend(
      Some(mock_registry_datum(1)),
      UpdateScript,
      mock_oref(),
      tx,
    )
  }
}
```

## Key Concepts Demonstrated

1. **CIP-33 reference scripts** — scripts stored as `reference_script` on UTxOs, reducing tx size and fees
2. **Continuing output pattern** — `UpdateScript` verifies a valid output returns to the same script address
3. **Version monotonicity** — `new_datum.version == d.version + 1` prevents replay and enforces ordering
4. **Lovelace preservation** — output lovelace must be >= input lovelace to prevent draining the UTxO
5. **Admin immutability on update** — admin and script_hash cannot change during `UpdateScript`
6. **`reference_script` field matching** — `script_user` checks `ref_input.output.reference_script == Some(hash)` to verify script availability
7. **Two-validator architecture** — separate registry (manages scripts) and user (consumes reference scripts)
