# Governance Propose — Treasury Withdrawal Guardrail

A governance guardrail script using the `propose` handler that only permits
TreasuryWithdrawal proposals with minimum deposit requirements.

## Types

```aiken
pub type ProposeRedeemer {
  Submit
}
```

## Governance Action Reference

```aiken
// From cardano/governance:
type GovernanceAction {
  ProtocolParameters { ancestor, new_parameters, guardrails }
  HardFork { ancestor, new_version }
  TreasuryWithdrawal { beneficiaries: Pairs<Credential, Lovelace>, guardrails: Option<ScriptHash> }
  NoConfidence { ancestor }
  ConstitutionalCommittee { ancestor, evicted_members, added_members, quorum }
  NewConstitution { ancestor, constitution }
  NicePoll
}

type ProposalProcedure {
  deposit: Lovelace,
  return_address: Credential,
  governance_action: GovernanceAction,
}
```

## Validator

```aiken
// validators/governance_propose.ak

use aiken/collection/list
use cardano/address.{Credential, VerificationKey}
use cardano/governance.{GovernanceAction, NicePoll, ProposalProcedure,
  TreasuryWithdrawal}
use cardano/transaction.{Transaction}

const min_deposit: Int = 100_000_000

fn is_treasury_withdrawal(action: GovernanceAction) -> Bool {
  when action is {
    TreasuryWithdrawal { .. } -> True
    _ -> False
  }
}

validator treasury_guardrail(authorized_proposer: ByteArray) {
  propose(
    _redeemer: ProposeRedeemer,
    proposal: ProposalProcedure,
    tx: Transaction,
  ) {
    let valid_action = is_treasury_withdrawal(proposal.governance_action)?
    let sufficient_deposit = (proposal.deposit >= min_deposit)?
    let signed = list.has(tx.extra_signatories, authorized_proposer)?
    valid_action && sufficient_deposit && signed
  }
}
```

## Tests

```aiken
// validators/governance_propose.ak (continued)

const mock_proposer =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_beneficiary_hash =
  #"1122334455667788990011223344556677889900112233445566778899001122"

fn mock_treasury_proposal(deposit: Int) -> ProposalProcedure {
  ProposalProcedure {
    deposit: deposit,
    return_address: VerificationKey(mock_proposer),
    governance_action: TreasuryWithdrawal {
      beneficiaries: [Pair(VerificationKey(mock_beneficiary_hash), 50_000_000)],
      guardrails: None,
    },
  }
}

fn mock_nicepoll_proposal(deposit: Int) -> ProposalProcedure {
  ProposalProcedure {
    deposit: deposit,
    return_address: VerificationKey(mock_proposer),
    governance_action: NicePoll,
  }
}

test valid_treasury_proposal_succeeds() {
  let proposal = mock_treasury_proposal(100_000_000)
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [mock_proposer] }
  treasury_guardrail.propose(mock_proposer, Submit, proposal, tx)
}

test non_treasury_action_fails() fail {
  let proposal = mock_nicepoll_proposal(100_000_000)
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [mock_proposer] }
  treasury_guardrail.propose(mock_proposer, Submit, proposal, tx)
}

test insufficient_deposit_fails() fail {
  let proposal = mock_treasury_proposal(99_999_999)
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [mock_proposer] }
  treasury_guardrail.propose(mock_proposer, Submit, proposal, tx)
}

test unsigned_proposal_fails() fail {
  let proposal = mock_treasury_proposal(100_000_000)
  treasury_guardrail.propose(mock_proposer, Submit, proposal, transaction.placeholder)
}

// Property: any sufficient deposit passes
test prop_valid_deposit_always_passes(
  deposit via fuzz.int_between(100_000_000, 1_000_000_000),
) {
  let proposal = mock_treasury_proposal(deposit)
  let tx =
    Transaction { ..transaction.placeholder, extra_signatories: [mock_proposer] }
  treasury_guardrail.propose(mock_proposer, Submit, proposal, tx)
}
```

## Key Concepts Demonstrated

1. **`propose` handler signature** — `propose(redeemer, proposal: ProposalProcedure, tx: Transaction)`
2. **`ProposalProcedure` fields** — `deposit`, `return_address: Credential`, `governance_action`
3. **`GovernanceAction` pattern matching** — `TreasuryWithdrawal { .. }` with wildcard catch-all
4. **`TreasuryWithdrawal` construction** — `beneficiaries: Pairs<Credential, Lovelace>` uses `[Pair(cred, amount)]`
5. **`NicePoll`** — simplest governance action (no fields), useful for testing
6. **`return_address` is `Credential`** — use `VerificationKey(hash)`, not a full `Address`
7. **`ProtocolParametersUpdate` is opaque** — cannot be constructed in tests, avoid in examples
