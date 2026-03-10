# Governance Vote — SPO Voting Authorization

Validates that governance votes cast on behalf of a stake pool are authorized
by the pool operator. Uses the `vote` handler introduced in Plutus V3 / Conway.

## Types

```aiken
pub type VoteRedeemer {
  CastVote
}
```

## Governance Type Reference

```aiken
// From cardano/governance:
type Voter {
  ConstitutionalCommitteeMember(Credential)
  DelegateRepresentative(Credential)
  StakePool(VerificationKeyHash)
}

type Vote { No, Yes, Abstain }

type GovernanceActionId {
  transaction: TransactionId,
  proposal_procedure: Int,
}

// Transaction field:
votes: Pairs<Voter, Pairs<GovernanceActionId, Vote>>
```

## Validator

```aiken
// validators/governance_vote.ak

use aiken/collection/list
use cardano/governance.{GovernanceActionId, StakePool, Vote, Voter}
use cardano/transaction.{Transaction}

validator spo_vote(pool_operator: ByteArray) {
  vote(redeemer: VoteRedeemer, _voter: Voter, tx: Transaction) {
    when redeemer is {
      CastVote -> list.has(tx.extra_signatories, pool_operator)?
    }
  }
}
```

## Tests

```aiken
// validators/governance_vote.ak (continued)

const operator_key =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const wrong_key =
  #"1111111111111111111111111111111111111111111111111111111111111111"

const pool_hash =
  #"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"

const mock_gov_action_id =
  GovernanceActionId {
    transaction: #"0000000000000000000000000000000000000000000000000000000000000000",
    proposal_procedure: 0,
  }

fn mock_voter() -> Voter {
  StakePool(pool_hash)
}

test operator_signed_vote_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [operator_key],
      votes: [Pair(mock_voter(), [Pair(mock_gov_action_id, governance.Yes)])],
    }
  spo_vote.vote(operator_key, CastVote, mock_voter(), tx)
}

test unsigned_vote_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      votes: [Pair(mock_voter(), [Pair(mock_gov_action_id, governance.Yes)])],
    }
  spo_vote.vote(operator_key, CastVote, mock_voter(), tx)
}

test multiple_votes_in_transaction_succeeds() {
  let action_id_2 =
    GovernanceActionId {
      transaction: #"1111111111111111111111111111111111111111111111111111111111111111",
      proposal_procedure: 0,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [operator_key],
      votes: [
        Pair(
          mock_voter(),
          [
            Pair(mock_gov_action_id, governance.Yes),
            Pair(action_id_2, governance.No),
          ],
        ),
      ],
    }
  spo_vote.vote(operator_key, CastVote, mock_voter(), tx)
}

// Property: any vote type passes with operator signature
test prop_any_vote_type_with_operator_succeeds(
  vote_choice via fuzz.int_between(0, 2),
) {
  let chosen_vote: Vote =
    if vote_choice == 0 {
      governance.Yes
    } else if vote_choice == 1 {
      governance.No
    } else {
      governance.Abstain
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [operator_key],
      votes: [Pair(mock_voter(), [Pair(mock_gov_action_id, chosen_vote)])],
    }
  spo_vote.vote(operator_key, CastVote, mock_voter(), tx)
}
```

## Key Concepts Demonstrated

1. **`vote` handler signature** — `vote(redeemer, voter: Voter, tx: Transaction)`
2. **`Voter` type** — `StakePool(hash)`, `DelegateRepresentative(cred)`, `ConstitutionalCommitteeMember(cred)`
3. **`Vote` constructors** — accessed as `governance.Yes`, `governance.No`, `governance.Abstain`
4. **`GovernanceActionId`** — `{ transaction: TransactionId, proposal_procedure: Int }`
5. **`tx.votes` field** — `Pairs<Voter, Pairs<GovernanceActionId, Vote>>` (nested pairs)
6. **Parameterized validator test calling** — `validator_name.vote(param, redeemer, voter, tx)` — parameter first
7. **Import path** — `use cardano/governance.{GovernanceActionId, StakePool, Vote, Voter}`
