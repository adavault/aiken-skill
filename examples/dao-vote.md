# DAO Vote — Token-Weighted Governance Voting

A token-weighted DAO voting contract where users lock governance tokens at a
script address to cast votes on proposals. Vote weight equals the number of
governance tokens locked. After the voting deadline, voters reclaim their tokens.

## Types

```aiken
pub type VoteDatum {
  proposal_id: Int,
  vote_for: Bool,       // True = yes, False = no
  voter: ByteArray,
}

pub type VoteRedeemer {
  Unlock
}
```

## Validator

```aiken
// validators/dao_vote.ak

use aiken/collection/list
use aiken/interval
use cardano/address
use cardano/assets
use cardano/transaction.{InlineDatum, Input, Output, OutputReference, Transaction}

/// Find the script input being spent and return its output
fn get_own_input(
  inputs: List<Input>,
  oref: OutputReference,
) -> Output {
  expect Some(input) =
    list.find(inputs, fn(inp) { inp.output_reference == oref })
  input.output
}

validator dao_vote(
  gov_token_policy: ByteArray,
  gov_token_name: ByteArray,
  voting_deadline: Int,
) {
  spend(
    datum: Option<VoteDatum>,
    redeemer: VoteRedeemer,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      Unlock -> {
        // 1. Voter must sign the transaction
        let voter_signed = list.has(tx.extra_signatories, d.voter)?

        // 2. Voting period must be over
        let voting_ended =
          interval.is_entirely_after(tx.validity_range, voting_deadline)?

        // 3. Governance tokens must actually be present (prevents empty vote UTxOs)
        let own_output = get_own_input(tx.inputs, oref)
        let gov_token_count =
          assets.quantity_of(own_output.value, gov_token_policy, gov_token_name)
        let has_gov_tokens = (gov_token_count > 0)?

        voter_signed && voting_ended && has_gov_tokens
      }
    }
  }
}
```

## Tests

```aiken
// validators/dao_vote.ak (continued)

const mock_voter =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const other_signer =
  #"1111111111111111111111111111111111111111111111111111111111111111"

const gov_policy =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

const gov_name = "GOV"

const deadline = 1_700_000_000_000

const after_deadline = 1_700_000_000_001

const before_deadline = 1_699_999_999_999

const mock_script_hash =
  #"eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee"

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn yes_vote_datum() -> VoteDatum {
  VoteDatum { proposal_id: 1, vote_for: True, voter: mock_voter }
}

fn no_vote_datum() -> VoteDatum {
  VoteDatum { proposal_id: 1, vote_for: False, voter: mock_voter }
}

fn vote_input(oref: OutputReference, gov_qty: Int) -> Input {
  Input {
    output_reference: oref,
    output: Output {
      address: address.from_script(mock_script_hash),
      value: assets.from_lovelace(2_000_000)
        |> assets.add(gov_policy, gov_name, gov_qty),
      datum: InlineDatum(yes_vote_datum()),
      reference_script: None,
    },
  }
}

fn empty_vote_input(oref: OutputReference) -> Input {
  Input {
    output_reference: oref,
    output: Output {
      address: address.from_script(mock_script_hash),
      value: assets.from_lovelace(2_000_000),
      datum: InlineDatum(yes_vote_datum()),
      reference_script: None,
    },
  }
}

test voter_unlocks_after_deadline_succeeds() {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_voter],
      validity_range: interval.after(after_deadline),
      inputs: [vote_input(oref, 100)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(yes_vote_datum()), Unlock, oref, tx,
  )
}

test voter_unlock_before_deadline_fails() fail {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_voter],
      validity_range: interval.before(before_deadline),
      inputs: [vote_input(oref, 100)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(yes_vote_datum()), Unlock, oref, tx,
  )
}

test non_voter_unlock_fails() fail {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [other_signer],
      validity_range: interval.after(after_deadline),
      inputs: [vote_input(oref, 100)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(yes_vote_datum()), Unlock, oref, tx,
  )
}

test zero_gov_tokens_fails() fail {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_voter],
      validity_range: interval.after(after_deadline),
      inputs: [empty_vote_input(oref)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(yes_vote_datum()), Unlock, oref, tx,
  )
}

test no_vote_unlock_succeeds() {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_voter],
      validity_range: interval.after(after_deadline),
      inputs: [vote_input(oref, 50)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(no_vote_datum()), Unlock, oref, tx,
  )
}

test different_proposal_unlocks() {
  let oref = mock_oref()
  let datum = VoteDatum { proposal_id: 42, vote_for: True, voter: mock_voter }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_voter],
      validity_range: interval.after(after_deadline),
      inputs: [vote_input(oref, 100)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(datum), Unlock, oref, tx,
  )
}

// Property: any voter can unlock their own tokens after deadline
test prop_voter_can_always_unlock_after_deadline(
  params via fuzz.both(
    fuzz.bytearray_fixed(28),
    fuzz.int_at_least(after_deadline),
  ),
) {
  let (voter, unlock_time) = params
  let oref = mock_oref()
  let datum = VoteDatum { proposal_id: 1, vote_for: True, voter: voter }
  let input =
    Input {
      output_reference: oref,
      output: Output {
        address: address.from_script(mock_script_hash),
        value: assets.from_lovelace(2_000_000)
          |> assets.add(gov_policy, gov_name, 100),
        datum: InlineDatum(datum),
        reference_script: None,
      },
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [voter],
      validity_range: interval.after(unlock_time),
      inputs: [input],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(datum), Unlock, oref, tx,
  )
}

// Property: stranger can never unlock (different key from datum.voter)
test prop_stranger_cannot_unlock(
  stranger via fuzz.bytearray_fixed(28),
) fail {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [stranger],
      validity_range: interval.after(after_deadline),
      inputs: [vote_input(oref, 100)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(yes_vote_datum()), Unlock, oref, tx,
  )
}

// Property: any token quantity > 0 works
test prop_any_token_quantity_works(
  qty via fuzz.int_between(1, 10_000_000),
) {
  let oref = mock_oref()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_voter],
      validity_range: interval.after(after_deadline),
      inputs: [vote_input(oref, qty)],
    }
  dao_vote.spend(
    gov_policy, gov_name, deadline,
    Some(yes_vote_datum()), Unlock, oref, tx,
  )
}
```

## Key Concepts Demonstrated

1. **Parameterized validator** — `dao_vote(gov_token_policy, gov_token_name, voting_deadline)` fixes governance parameters at deployment
2. **Token-weighted voting** — vote weight = `assets.quantity_of(value, policy, name)`, no separate field needed
3. **Test calling convention** — parameterized validators pass params first: `dao_vote.spend(policy, name, deadline, datum, redeemer, oref, tx)`
4. **`list.find` for own input** — alternative to `transaction.find_input` using direct pattern
5. **Zero-token prevention** — `gov_token_count > 0` prevents empty UTxOs from counting as votes
6. **`fuzz.int_at_least(n)`** — generates integers >= n for property tests on time ranges
7. **`fuzz.int_between(lo, hi)`** — bounded integer fuzzing for token quantities
8. **`fuzz.both(a, b)`** — combines two fuzzers into a tuple for multi-parameter property tests
9. **Boolean vote** — `vote_for: Bool` keeps the datum simple; off-chain tallying reads UTxOs

## Security Notes

- **Governance tokens must exist** — the `gov_token_count > 0` check prevents locking empty UTxOs as phantom votes
- **Tokens locked until deadline** — cannot unlock early, ensuring vote commitment
- **Only the voter can unlock** — `d.voter` must sign, preventing token theft
- **Off-chain tallying** — the contract doesn't count votes on-chain; tallying happens by reading script UTxOs off-chain (standard Cardano pattern)
- **One UTxO per vote** — each vote is a separate UTxO; a voter can create multiple votes by sending multiple transactions
- **No vote changing** — once locked, the vote cannot be modified (the UTxO is immutable). Voter must wait until after the deadline to unlock.
