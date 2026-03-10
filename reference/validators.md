# Validator Patterns

## Validator Architecture

Every Aiken validator is a predicate attached to a script address or minting policy.
When a transaction interacts with the script, the validator is executed and must return
`True` to authorize the action.

### Single-Purpose Validator

Most validators handle one purpose:

```aiken
use cardano/transaction.{Transaction}

type Datum {
  owner: VerificationKeyHash,
}

type Redeemer {
  Claim
}

validator simple_lock {
  spend(datum: Option<Datum>, redeemer: Redeemer, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      Claim -> list.has(tx.extra_signatories, d.owner)
    }
  }
}
```

### Multi-Purpose Validator (Spend + Mint)

Same script hash handles both spending and minting — useful for token-gated
contracts where the token and the locked UTxO are coupled:

```aiken
validator token_lock {
  mint(redeemer: MintRedeemer, policy_id: PolicyId, tx: Transaction) {
    when redeemer is {
      MintToken -> {
        // Ensure exactly one token minted
        let minted = assets.tokens(tx.mint, policy_id)
        dict.to_pairs(minted) == [Pair(token_name, 1)]
      }
      BurnToken -> {
        let minted = assets.tokens(tx.mint, policy_id)
        dict.to_pairs(minted) == [Pair(token_name, -1)]
      }
    }
  }

  spend(datum: Option<LockDatum>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    // Spending requires burning the authentication token
    let policy_id = // ... derive from own script hash
    assets.quantity_of(tx.mint, policy_id, token_name) == -1
  }
}
```

### Governance Validators (Conway Era)

**Confirmed working** — See governance-vote.md, governance-publish.md, governance-propose.md.

```aiken
// Vote handler — authorize governance votes
validator spo_vote(pool_operator: ByteArray) {
  vote(redeemer: VoteRedeemer, _voter: Voter, tx: Transaction) {
    list.has(tx.extra_signatories, pool_operator)?
  }
}

// Publish handler — control certificate operations (DRep registration, etc.)
validator drep_controller(admin: ByteArray) {
  publish(_redeemer: Action, certificate: Certificate, tx: Transaction) {
    when certificate is {
      RegisterDelegateRepresentative { .. } -> list.has(tx.extra_signatories, admin)?
      _ -> fail @"unsupported certificate"
    }
  }
}

// Propose handler — guardrail for governance proposals
validator treasury_guardrail(proposer: ByteArray) {
  propose(_redeemer: Data, proposal: ProposalProcedure, tx: Transaction) {
    when proposal.governance_action is {
      TreasuryWithdrawal { .. } -> list.has(tx.extra_signatories, proposer)?
      _ -> fail @"only treasury withdrawals allowed"
    }
  }
}
```

**Test calling convention** for parameterized validators:
```aiken
// validator_name.handler(param, redeemer, handler_arg, tx)
spo_vote.vote(operator_key, CastVote, mock_voter(), tx)
drep_controller.publish(admin_key, Register, cert, tx)
treasury_guardrail.propose(proposer_key, Submit, proposal, tx)
```

### Parameterized Validator

Parameters are fixed at deployment time. Different parameters = different script address:

```aiken
validator time_lock(owner: VerificationKeyHash, lock_until: POSIXTime) {
  spend(_datum: Option<Data>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    let signed = list.has(tx.extra_signatories, owner)
    let after_deadline = interval.is_entirely_after(tx.validity_range, lock_until)
    signed && after_deadline
  }
}
```

## Common Validator Recipes

### Owner-Only Spend

```aiken
validator owner_lock {
  spend(datum: Option<Datum>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    list.has(tx.extra_signatories, d.owner)?
  }
}
```

### Time-Locked Vault

```aiken
use aiken/interval

type VaultDatum {
  owner: VerificationKeyHash,
  beneficiary: VerificationKeyHash,
  lock_until: POSIXTime,
}

type VaultRedeemer {
  OwnerWithdraw
  BeneficiaryWithdraw
}

validator vault {
  spend(datum: Option<VaultDatum>, redeemer: VaultRedeemer, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    when redeemer is {
      // Owner can always withdraw
      OwnerWithdraw -> list.has(tx.extra_signatories, d.owner)
      // Beneficiary can withdraw only after lock period
      BeneficiaryWithdraw -> {
        let signed = list.has(tx.extra_signatories, d.beneficiary)
        let unlocked = interval.is_entirely_after(tx.validity_range, d.lock_until)
        signed? && unlocked?
      }
    }
  }
}
```

### One-Shot Minting Policy

Mints exactly once by requiring a specific UTxO as input (UTxOs are unique):

```aiken
validator one_shot(utxo_ref: OutputReference) {
  mint(_redeemer: Data, _policy_id: PolicyId, tx: Transaction) {
    // The UTxO must be consumed — can only happen once
    list.any(tx.inputs, fn(input) { input.output_reference == utxo_ref })
  }
}
```

### NFT Authentication

Use a minted NFT to authenticate a UTxO. Prevents datum hijacking:

```aiken
validator authenticated_vault {
  mint(redeemer: MintAction, policy_id: PolicyId, tx: Transaction) {
    when redeemer is {
      MintAuth -> {
        // Mint exactly 1 auth token
        let minted = assets.tokens(tx.mint, policy_id)
        dict.to_pairs(minted) == [Pair("AUTH", 1)]
      }
      BurnAuth -> {
        let minted = assets.tokens(tx.mint, policy_id)
        dict.to_pairs(minted) == [Pair("AUTH", -1)]
      }
    }
  }

  spend(datum: Option<VaultDatum>, _redeemer: Data, oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    // Must burn auth token to spend
    let own_policy = // derive from script hash
    let burns_auth = assets.quantity_of(tx.mint, own_policy, "AUTH") == -1
    let signed = list.has(tx.extra_signatories, d.owner)
    burns_auth? && signed?
  }
}
```

### Multi-Signature (M-of-N)

```aiken
type MultiSigDatum {
  signers: List<VerificationKeyHash>,
  threshold: Int,
}

validator multi_sig {
  spend(datum: Option<MultiSigDatum>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
    expect Some(d) = datum
    let valid_sigs =
      d.signers
        |> list.filter(fn(s) { list.has(tx.extra_signatories, s) })
        |> list.length
    valid_sigs >= d.threshold
  }
}
```

### Continuing Output (State Machine)

Validator that requires its own output to persist with updated state.

**Confirmed working patterns:**
- `transaction.find_input(tx.inputs, oref)` — finds the input being validated
- `InlineDatum(MyType { ... })` — Aiken auto-coerces custom types to `Data`
- `expect new_state: MyType = raw` — casts `Data` back to typed value
- `list.filter` on `tx.outputs` by `payment_credential` to find script outputs

```aiken
validator counter {
  spend(datum: Option<CounterState>, redeemer: CounterAction, oref: OutputReference, tx: Transaction) {
    expect Some(state) = datum
    expect Some(own_input) = transaction.find_input(tx.inputs, oref)
    let script_cred = own_input.output.address.payment_credential

    when redeemer is {
      Increment -> {
        // Find continuing outputs to same script
        let script_outputs =
          list.filter(tx.outputs, fn(o) {
            o.address.payment_credential == script_cred
          })
        // Exactly one continuing output
        expect [continuing_output] = script_outputs
        // Extract and type-check the new datum
        expect InlineDatum(raw) = continuing_output.datum
        expect new_state: CounterState = raw
        (new_state.count == state.count + 1)? &&
        (new_state.owner == state.owner)?
      }
      Reset -> list.has(tx.extra_signatories, state.owner)?
    }
  }
}
```
