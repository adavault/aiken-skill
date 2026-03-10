# Pool Restriction — Certificate-Based Delegation Control

Restricts delegation to a whitelist of allowed stake pools. Critical for
protocols like ADAvault that need to ensure vault funds only delegate to
specific pools.

**Key learning:** Must check ALL certificate types that involve pool delegation:
`DelegateCredential` and `RegisterAndDelegateCredential`, each with both
`DelegateBlockProduction` and `DelegateBoth` delegate variants. The field name
is `stake_pool` (not `pool_id`).

## Types

```aiken
pub type PoolId =
  ByteArray

pub type VaultDatum {
  owner: ByteArray,
  allowed_pools: List<PoolId>,
}
```

## Certificate Type Reference

Relevant constructors from `cardano/certificate`:

```aiken
// Certificate constructors involving pool delegation:
DelegateCredential { credential: Credential, delegate: Delegate }
RegisterAndDelegateCredential { credential: Credential, delegate: Delegate, deposit: Lovelace }

// Delegate variants:
DelegateBlockProduction { stake_pool: StakePoolId }  // pool only
DelegateVote { delegate_representative: DelegateRepresentative }  // voting only — no pool
DelegateBoth { stake_pool: StakePoolId, delegate_representative: DelegateRepresentative }

// DelegateRepresentative variants:
Registered(Credential)  // specific DRep
AlwaysAbstain            // always abstain
AlwaysNoConfidence       // always no confidence
```

## Validator

```aiken
// validators/pool_restriction.ak

use aiken/collection/list
use cardano/address.{Credential, VerificationKey}
use cardano/certificate.{
  Certificate, DelegateBoth, DelegateBlockProduction, DelegateCredential,
  RegisterAndDelegateCredential,
}
use cardano/transaction.{OutputReference, Transaction}

/// Check that every delegation certificate targets an allowed pool
fn validate_delegation(
  certs: List<Certificate>,
  allowed_pools: List<PoolId>,
) -> Bool {
  list.all(
    certs,
    fn(cert) {
      when cert is {
        DelegateCredential { delegate, .. } ->
          when delegate is {
            DelegateBlockProduction { stake_pool } ->
              list.has(allowed_pools, stake_pool)?
            DelegateBoth { stake_pool, .. } ->
              list.has(allowed_pools, stake_pool)?
            // DelegateVote only — no pool involved, allow
            _ -> True
          }
        RegisterAndDelegateCredential { delegate, .. } ->
          when delegate is {
            DelegateBlockProduction { stake_pool } ->
              list.has(allowed_pools, stake_pool)?
            DelegateBoth { stake_pool, .. } ->
              list.has(allowed_pools, stake_pool)?
            _ -> True
          }
        // Non-delegation certificates are fine
        _ -> True
      }
    },
  )
}

validator pool_lock {
  spend(
    datum: Option<VaultDatum>,
    _redeemer: Data,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum
    let signed = list.has(tx.extra_signatories, d.owner)?
    let valid_delegation = validate_delegation(tx.certificates, d.allowed_pools)
    signed && valid_delegation
  }
}
```

## Tests

```aiken
// validators/pool_restriction.ak (continued)

const mock_owner = #"cc01"

const pool_adv =
  #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

const pool_adv2 =
  #"bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"

const pool_other =
  #"cccccccccccccccccccccccccccccccccccccccccccccccccccccccccc"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> VaultDatum {
  VaultDatum { owner: mock_owner, allowed_pools: [pool_adv, pool_adv2] }
}

fn mock_stake_cred() -> Credential {
  VerificationKey(#"dd01")
}

test delegate_to_allowed_pool_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        DelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBlockProduction { stake_pool: pool_adv },
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test delegate_to_disallowed_pool_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        DelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBlockProduction { stake_pool: pool_other },
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test delegate_both_to_allowed_pool_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        DelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBoth {
            stake_pool: pool_adv,
            delegate_representative: certificate.AlwaysAbstain,
          },
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test delegate_both_to_disallowed_pool_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        DelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBoth {
            stake_pool: pool_other,
            delegate_representative: certificate.AlwaysAbstain,
          },
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test register_and_delegate_to_allowed_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        RegisterAndDelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBlockProduction { stake_pool: pool_adv },
          deposit: 2_000_000,
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test register_and_delegate_to_disallowed_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        RegisterAndDelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBlockProduction { stake_pool: pool_other },
          deposit: 2_000_000,
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test no_certificates_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

test unsigned_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      certificates: [
        DelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBlockProduction { stake_pool: pool_adv },
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}

// Property: any allowed pool always passes
test prop_allowed_pool_always_passes(idx via fuzz.int_between(0, 1)) {
  let pool =
    if idx == 0 {
      pool_adv
    } else {
      pool_adv2
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
      certificates: [
        DelegateCredential {
          credential: mock_stake_cred(),
          delegate: DelegateBlockProduction { stake_pool: pool },
        },
      ],
    }
  pool_lock.spend(Some(mock_datum()), Void, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **`tx.certificates`** — `List<Certificate>` field on Transaction, accessible in tests via spread syntax
2. **`DelegateCredential`** — certificate constructor with `credential` and `delegate` fields
3. **`DelegateBlockProduction { stake_pool }`** — delegate variant for pool-only delegation (field is `stake_pool`, not `pool_id`)
4. **`DelegateBoth { stake_pool, delegate_representative }`** — combined pool + governance delegation
5. **`RegisterAndDelegateCredential`** — register + delegate in one certificate (includes `deposit` field)
6. **`certificate.AlwaysAbstain`** — DelegateRepresentative variant for governance voting
7. **`pub type` on validator datum** — types used in validator signatures must be `pub`
8. **Exhaustive certificate matching** — must handle ALL delegation certificate types to prevent bypass

## Security Note

Failing to check `RegisterAndDelegateCredential` alongside `DelegateCredential`
would allow an attacker to register-and-delegate in a single certificate,
bypassing pool restriction. Similarly, ignoring `DelegateBoth` would allow
delegation via the combined governance+pool certificate path.
