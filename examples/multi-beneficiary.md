# Multi-Beneficiary — Percentage-Based Fund Splitting

An inheritance vault that splits funds to multiple heirs based on basis-point
shares. Owner can withdraw anytime. Beneficiaries can only claim their
individual shares after the unlock time.

## Types

```aiken
pub type InheritanceDatum {
  owner: ByteArray,
  beneficiaries: List<Pair<ByteArray, Int>>,  // (key_hash, share_bps)
  unlock_after: Int,                           // POSIX time
}

pub type InheritanceRedeemer {
  OwnerWithdraw
  ClaimShare { beneficiary_index: Int, output_index: Int }
}
```

## Validator

```aiken
// validators/multi_beneficiary.ak

use aiken/collection/list
use aiken/interval
use cardano/address
use cardano/assets
use cardano/transaction.{Input, Output, OutputReference, Transaction}

validator multi_beneficiary {
  spend(
    datum: Option<InheritanceDatum>,
    redeemer: InheritanceRedeemer,
    oref: OutputReference,
    tx: Transaction,
  ) {
    expect Some(d) = datum

    when redeemer is {
      OwnerWithdraw -> list.has(tx.extra_signatories, d.owner)?

      ClaimShare { beneficiary_index, output_index } -> {
        // Time must be after unlock_after
        let time_ok =
          interval.is_entirely_after(tx.validity_range, d.unlock_after)?

        // Look up the beneficiary at the given index
        expect Some(beneficiary_pair) =
          list.at(d.beneficiaries, beneficiary_index)
        let Pair(beneficiary_key, share_bps) = beneficiary_pair

        // Beneficiary must have signed
        let signed = list.has(tx.extra_signatories, beneficiary_key)?

        // Get the input value (our own UTxO)
        expect Some(own_input) =
          transaction.find_input(tx.inputs, oref)
        let input_lovelace = assets.lovelace_of(own_input.output.value)

        // Calculate minimum share
        let min_share = input_lovelace * share_bps / 10000

        // Look up output at output_index
        expect Some(output) = list.at(tx.outputs, output_index)

        // Verify output goes to the beneficiary
        let correct_address =
          (output.address == address.from_verification_key(beneficiary_key))?

        // Verify output value >= their share
        let sufficient_value =
          (assets.lovelace_of(output.value) >= min_share)?

        time_ok && signed && correct_address && sufficient_value
      }
    }
  }
}
```

## Tests

```aiken
// validators/multi_beneficiary.ak (continued)

const mock_owner =
  #"aabbccddee112233445566778899001122334455667788990011223344556677"

const mock_heir_a =
  #"112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"

const mock_heir_b =
  #"ffeeddccbbaa99887766554433221100ffeeddccbbaa99887766554433221100"

const mock_heir_c =
  #"00112233445566778899aabbccddeeff00112233445566778899aabbccddee"

const unlock_time = 1_700_000_000_000
const vault_lovelace = 100_000_000  // 100 ADA

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

fn mock_datum() -> InheritanceDatum {
  InheritanceDatum {
    owner: mock_owner,
    beneficiaries: [
      Pair(mock_heir_a, 5000),   // 50%
      Pair(mock_heir_b, 3000),   // 30%
      Pair(mock_heir_c, 2000),   // 20%
    ],
    unlock_after: unlock_time,
  }
}

fn mock_input() -> Input {
  Input {
    output_reference: mock_oref(),
    output: Output {
      address: address.from_script(
        #"cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc",
      ),
      value: assets.from_lovelace(vault_lovelace),
      datum: transaction.NoDatum,
      reference_script: None,
    },
  }
}

fn beneficiary_output(key: ByteArray, lovelace: Int) -> Output {
  Output {
    address: address.from_verification_key(key),
    value: assets.from_lovelace(lovelace),
    datum: transaction.NoDatum,
    reference_script: None,
  }
}

test owner_withdraw_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      extra_signatories: [mock_owner],
    }
  multi_beneficiary.spend(Some(mock_datum()), OwnerWithdraw, mock_oref(), tx)
}

test beneficiary_claims_correct_share_after_unlock() {
  // Heir A has 5000 bps = 50% of 100 ADA = 50 ADA
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      outputs: [beneficiary_output(mock_heir_a, 50_000_000)],
      extra_signatories: [mock_heir_a],
      validity_range: interval.after(unlock_time + 1),
    }
  multi_beneficiary.spend(
    Some(mock_datum()),
    ClaimShare { beneficiary_index: 0, output_index: 0 },
    mock_oref(),
    tx,
  )
}

test beneficiary_claim_before_unlock_fails() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      outputs: [beneficiary_output(mock_heir_a, 50_000_000)],
      extra_signatories: [mock_heir_a],
      validity_range: interval.before(unlock_time),
    }
  multi_beneficiary.spend(
    Some(mock_datum()),
    ClaimShare { beneficiary_index: 0, output_index: 0 },
    mock_oref(),
    tx,
  )
}

test beneficiary_claims_wrong_amount_fails() fail {
  // Heir A entitled to 50 ADA but output only has 30 ADA
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      outputs: [beneficiary_output(mock_heir_a, 30_000_000)],
      extra_signatories: [mock_heir_a],
      validity_range: interval.after(unlock_time + 1),
    }
  multi_beneficiary.spend(
    Some(mock_datum()),
    ClaimShare { beneficiary_index: 0, output_index: 0 },
    mock_oref(),
    tx,
  )
}

test wrong_beneficiary_signing_fails() fail {
  // Heir B signs but tries to claim Heir A's share (index 0)
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      outputs: [beneficiary_output(mock_heir_a, 50_000_000)],
      extra_signatories: [mock_heir_b],
      validity_range: interval.after(unlock_time + 1),
    }
  multi_beneficiary.spend(
    Some(mock_datum()),
    ClaimShare { beneficiary_index: 0, output_index: 0 },
    mock_oref(),
    tx,
  )
}

test second_beneficiary_claims_correct_share() {
  // Heir B has 3000 bps = 30% of 100 ADA = 30 ADA
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      outputs: [
        beneficiary_output(mock_heir_a, 50_000_000),
        beneficiary_output(mock_heir_b, 30_000_000),
      ],
      extra_signatories: [mock_heir_b],
      validity_range: interval.after(unlock_time + 1),
    }
  multi_beneficiary.spend(
    Some(mock_datum()),
    ClaimShare { beneficiary_index: 1, output_index: 1 },
    mock_oref(),
    tx,
  )
}

// Property: owner always authorized regardless of time
test prop_owner_always_authorized(
  params via fuzz.both(fuzz.bytearray_fixed(28), fuzz.int()),
) {
  let (owner, time) = params
  let datum =
    InheritanceDatum {
      owner: owner,
      beneficiaries: [Pair(mock_heir_a, 5000), Pair(mock_heir_b, 5000)],
      unlock_after: unlock_time,
    }
  let tx =
    Transaction {
      ..transaction.placeholder,
      inputs: [mock_input()],
      extra_signatories: [owner],
      validity_range: interval.after(time),
    }
  multi_beneficiary.spend(Some(datum), OwnerWithdraw, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **Basis-point shares** — `share_bps / 10000` for percentage calculation (5000 = 50%)
2. **`Pair` in datum** — `List<Pair<ByteArray, Int>>` for beneficiary-to-share mapping
3. **`let Pair(key, bps) = pair_value`** — destructuring Pairs
4. **`list.at(list, index)`** — O(1) lookup by redeemer-provided index
5. **`address.from_verification_key(key)`** — construct address for output comparison
6. **`transaction.find_input(tx.inputs, oref)`** — get own input value for share calculation
7. **Index-based output linking** — `output_index` in redeemer points to beneficiary's output
8. **Owner unconditional access** — no time restriction on `OwnerWithdraw`

## Security Notes

- Each beneficiary claims independently — no single transaction distributes all shares
- Share calculated from actual input value, not datum-stored amount — prevents stale data attacks
- Output address verified against beneficiary key — prevents funds going to wrong address
- `>= min_share` allows overpayment (generous heirs) but blocks underpayment
- Consider adding NFT authentication to prevent datum hijacking on the vault UTxO
