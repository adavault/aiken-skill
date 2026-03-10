# Dead Man's Switch — Proof-of-Life Inheritance

Owner must periodically "check in" or the beneficiary can claim the funds.
Combines the continuing output pattern (state machine) with time-based
validation. A novel inheritance pattern for Cardano.

## Types

```aiken
pub type SwitchDatum {
  owner: ByteArray,
  beneficiary: ByteArray,
  check_in_deadline: Int,      // POSIX time — must check in before this
  check_in_interval: Int,      // Milliseconds between required check-ins
}

pub type SwitchAction {
  CheckIn        // Owner extends the deadline
  Claim          // Beneficiary claims after deadline passed
  OwnerWithdraw  // Owner withdraws anytime
}
```

## Validator

```aiken
// validators/dead_mans_switch.ak

use aiken/collection/list
use aiken/interval.{Finite, IntervalBound}
use cardano/address
use cardano/assets
use cardano/transaction.{InlineDatum, Input, Output, OutputReference, Transaction}

validator dead_mans_switch {
  spend(
    datum: Option<SwitchDatum>,
    redeemer: SwitchAction,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      CheckIn -> {
        // Owner must sign
        expect list.has(tx.extra_signatories, d.owner)?

        // Extract current time from tx lower bound
        expect IntervalBound { bound_type: Finite(current_time), .. } =
          tx.validity_range.lower_bound

        // Find own input to get script credential
        expect Some(own_input) =
          transaction.find_input(tx.inputs, oref)
        let script_cred = own_input.output.address.payment_credential

        // Find the single continuing output to the same script
        let script_outputs =
          list.filter(tx.outputs, fn(o) {
            o.address.payment_credential == script_cred
          })
        expect [continuing_output] = script_outputs
        expect InlineDatum(raw) = continuing_output.datum
        expect new_datum: SwitchDatum = raw

        // New deadline must be current_time + check_in_interval
        let expected_deadline = current_time + d.check_in_interval

        and {
          (new_datum.owner == d.owner)?,
          (new_datum.beneficiary == d.beneficiary)?,
          (new_datum.check_in_interval == d.check_in_interval)?,
          (new_datum.check_in_deadline == expected_deadline)?,
        }
      }

      Claim -> {
        let signed_by_beneficiary =
          list.has(tx.extra_signatories, d.beneficiary)?
        let deadline_passed =
          interval.is_entirely_after(tx.validity_range, d.check_in_deadline)?
        signed_by_beneficiary && deadline_passed
      }

      OwnerWithdraw ->
        list.has(tx.extra_signatories, d.owner)?
    }
  }
}
```

## Tests

```aiken
// validators/dead_mans_switch.ak (continued)

const mock_owner =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_beneficiary =
  #"112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"

const mock_script_hash =
  #"dddddddddddddddddddddddddddddddddddddddddddddddddddddddd"

const initial_deadline = 1_700_000_000_000
const check_in_interval = 86_400_000  // 24 hours in ms

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> SwitchDatum {
  SwitchDatum {
    owner: mock_owner,
    beneficiary: mock_beneficiary,
    check_in_deadline: initial_deadline,
    check_in_interval: check_in_interval,
  }
}

fn mock_switch_input(d: SwitchDatum) -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: address.from_script(mock_script_hash),
      value: assets.from_lovelace(100_000_000),
      datum: InlineDatum(d),
      reference_script: None,
    },
  }
}

fn mock_switch_output(d: SwitchDatum) -> Output {
  Output {
    address: address.from_script(mock_script_hash),
    value: assets.from_lovelace(100_000_000),
    datum: InlineDatum(d),
    reference_script: None,
  }
}

test checkin_by_owner_succeeds() {
  let current_time = 1_699_999_000_000
  let d = mock_datum()
  let new_datum =
    SwitchDatum {
      ..d,
      check_in_deadline: current_time + check_in_interval,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_switch_input(d)],
      outputs: [mock_switch_output(new_datum)],
      extra_signatories: [mock_owner],
      validity_range: interval.after(current_time),
    }
  dead_mans_switch.spend(Some(d), CheckIn, mock_oref(), tx)
}

test checkin_by_non_owner_fails() fail {
  let current_time = 1_699_999_000_000
  let d = mock_datum()
  let new_datum =
    SwitchDatum {
      ..d,
      check_in_deadline: current_time + check_in_interval,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_switch_input(d)],
      outputs: [mock_switch_output(new_datum)],
      extra_signatories: [mock_beneficiary],
      validity_range: interval.after(current_time),
    }
  dead_mans_switch.spend(Some(d), CheckIn, mock_oref(), tx)
}

test claim_after_deadline_succeeds() {
  let d = mock_datum()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_beneficiary],
      validity_range: interval.after(initial_deadline + 1),
    }
  dead_mans_switch.spend(Some(d), Claim, mock_oref(), tx)
}

test claim_before_deadline_fails() fail {
  let d = mock_datum()
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_beneficiary],
      validity_range: interval.before(initial_deadline),
    }
  dead_mans_switch.spend(Some(d), Claim, mock_oref(), tx)
}

test owner_withdraw_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [mock_owner],
    }
  dead_mans_switch.spend(Some(mock_datum()), OwnerWithdraw, mock_oref(), tx)
}

// Property: owner always has full control
test prop_owner_always_can_withdraw(
  owner via fuzz.bytearray_fixed(28),
) {
  let d =
    SwitchDatum {
      owner: owner,
      beneficiary: mock_beneficiary,
      check_in_deadline: initial_deadline,
      check_in_interval: check_in_interval,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      extra_signatories: [owner],
    }
  dead_mans_switch.spend(Some(d), OwnerWithdraw, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Extracting lower bound** — `expect IntervalBound { bound_type: Finite(current_time), .. } = tx.validity_range.lower_bound`
2. **Continuing output with updated datum** — CheckIn produces output to same script with new deadline
3. **`and { }` block** — multi-condition check with individual `?` traces
4. **Record update syntax** — `SwitchDatum { ..d, check_in_deadline: new_value }`
5. **`interval.is_entirely_after`** — beneficiary can only claim when ENTIRE range is past deadline
6. **Owner always has control** — `OwnerWithdraw` has no time restriction
7. **`transaction.find_input`** — confirmed working function for finding own input

## Security Notes

- CheckIn validates ALL datum fields unchanged except `check_in_deadline` — prevents owner mutation
- New deadline computed from tx lower bound, not from old deadline — prevents "phantom check-ins"
- Beneficiary needs `is_entirely_after` (not just `after`) — prevents time-range straddling attacks
- Owner retains unconditional withdrawal — the switch is a safety net, not a prison
