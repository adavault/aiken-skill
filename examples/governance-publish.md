# Governance Publish — DRep Registration Controller

Controls who can register, update, or unregister as a Delegate Representative
(DRep) using the `publish` handler. Rejects non-DRep certificate types.

## Types

```aiken
pub type Action {
  Register
  Update
  Unregister
}
```

## Certificate Type Reference (DRep-related)

```aiken
// From cardano/certificate:
RegisterDelegateRepresentative { delegate_representative: Credential, deposit: Lovelace }
UpdateDelegateRepresentative { delegate_representative: Credential }
UnregisterDelegateRepresentative { delegate_representative: Credential, refund: Lovelace }
```

## Validator

```aiken
// validators/governance_publish.ak

use aiken/collection/list
use cardano/address.{Credential, VerificationKey}
use cardano/certificate.{
  Certificate, DelegateBlockProduction, DelegateCredential,
  RegisterDelegateRepresentative, UnregisterDelegateRepresentative,
  UpdateDelegateRepresentative,
}
use cardano/transaction.{Transaction}

validator drep_controller(admin: ByteArray) {
  publish(_redeemer: Action, certificate: Certificate, tx: Transaction) {
    let admin_signed = list.has(tx.extra_signatories, admin)
    when certificate is {
      RegisterDelegateRepresentative { .. } -> admin_signed?
      UpdateDelegateRepresentative { .. } -> admin_signed?
      UnregisterDelegateRepresentative { .. } -> admin_signed?
      _ -> fail @"This script only handles DRep operations"
    }
  }
}
```

## Tests

```aiken
// validators/governance_publish.ak (continued)

const mock_admin =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

fn mock_drep_cred() -> Credential {
  VerificationKey(
    #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd",
  )
}

test register_drep_with_admin_succeeds() {
  let cert =
    RegisterDelegateRepresentative {
      delegate_representative: mock_drep_cred(),
      deposit: 500_000_000,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_admin],
    }
  drep_controller.publish(mock_admin, Register, cert, tx)
}

test register_drep_without_admin_fails() fail {
  let cert =
    RegisterDelegateRepresentative {
      delegate_representative: mock_drep_cred(),
      deposit: 500_000_000,
    }
  drep_controller.publish(mock_admin, Register, cert, transaction.placeholder)
}

test update_drep_with_admin_succeeds() {
  let cert =
    UpdateDelegateRepresentative { delegate_representative: mock_drep_cred() }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_admin],
    }
  drep_controller.publish(mock_admin, Update, cert, tx)
}

test unregister_drep_with_admin_succeeds() {
  let cert =
    UnregisterDelegateRepresentative {
      delegate_representative: mock_drep_cred(),
      refund: 500_000_000,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_admin],
    }
  drep_controller.publish(mock_admin, Unregister, cert, tx)
}

test non_drep_certificate_fails() fail {
  let cert =
    DelegateCredential {
      credential: mock_drep_cred(),
      delegate: DelegateBlockProduction {
        stake_pool: #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_admin],
    }
  drep_controller.publish(mock_admin, Register, cert, tx)
}

// Property: any admin key works when signing
test prop_admin_signed_register_always_passes(
  admin_key via fuzz.bytearray_fixed(28),
) {
  let cert =
    RegisterDelegateRepresentative {
      delegate_representative: mock_drep_cred(),
      deposit: 500_000_000,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [admin_key],
    }
  drep_controller.publish(admin_key, Register, cert, tx)
}
```

## Key Concepts Demonstrated

1. **`publish` handler signature** — `publish(redeemer, certificate: Certificate, tx: Transaction)`
2. **Certificate pattern matching** — `when certificate is { RegisterDelegateRepresentative { .. } -> ... }`
3. **DRep certificate constructors** — Register, Update, Unregister with different field signatures
4. **Narrowly scoped script** — wildcard `_ -> fail` rejects unrelated certificate types
5. **`RegisterCredential` has `Never` type** — cannot be constructed in tests (uninhabitable deposit field)
6. **Parameterized validator test calling** — `drep_controller.publish(param, redeemer, cert, tx)`
