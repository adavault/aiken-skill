# Validity Range Normalisation

Cardano validity ranges use complex interval types with open/closed bounds and
positive/negative infinity. This pattern normalises them to a clean enum for
easier reasoning in validators.

**Key learning:** `Interval` in Aiken stdlib is NOT generic — use `Interval`
not `Interval<Int>`. Access bounds via `range.lower_bound.bound_type` and
`range.upper_bound.bound_type`.

## Types

```aiken
pub type NormalisedRange {
  ClosedRange { from: Int, to: Int }
  FromNegInf { to: Int }
  ToPosInf { from: Int }
  Always
}

pub type TimeRedeemer {
  MustBeAfter { deadline: Int }
  MustBeBefore { deadline: Int }
  MustBeBetween { start: Int, end: Int }
}
```

## Normalisation Function

```aiken
use aiken/interval.{Finite, Interval, IntervalBound, NegativeInfinity,
  PositiveInfinity}

fn normalise(range: Interval) -> NormalisedRange {
  when (range.lower_bound.bound_type, range.upper_bound.bound_type) is {
    (Finite(lo), Finite(hi)) -> ClosedRange { from: lo, to: hi }
    (NegativeInfinity, Finite(hi)) -> FromNegInf { to: hi }
    (Finite(lo), PositiveInfinity) -> ToPosInf { from: lo }
    (NegativeInfinity, PositiveInfinity) -> Always
    _ -> fail @"invalid range bounds"
  }
}
```

## Validator

```aiken
// validators/validity_range.ak

use aiken/collection/list
use aiken/fuzz
use aiken/interval.{Finite, Interval, IntervalBound, NegativeInfinity,
  PositiveInfinity}
use cardano/transaction.{OutputReference, Transaction}

validator time_check {
  spend(
    _datum: Option<Data>,
    redeemer: TimeRedeemer,
    _oref: OutputReference,
    tx: Transaction,
  ) {
    let range = normalise(tx.validity_range)

    when redeemer is {
      MustBeAfter { deadline } ->
        when range is {
          ToPosInf { from } -> (from > deadline)?
          ClosedRange { from, .. } -> (from > deadline)?
          _ -> fail @"range must have a lower bound"
        }
      MustBeBefore { deadline } ->
        when range is {
          FromNegInf { to } -> (to < deadline)?
          ClosedRange { to, .. } -> (to < deadline)?
          _ -> fail @"range must have an upper bound"
        }
      MustBeBetween { start, end } ->
        when range is {
          ClosedRange { from, to } ->
            (from >= start)? && (to <= end)?
          _ -> fail @"range must be closed"
        }
    }
  }
}
```

## Tests

```aiken
// validators/validity_range.ak (continued)

fn mock_oref() -> OutputReference {
  OutputReference {
    transaction_id: #"0000000000000000000000000000000000000000000000000000000000000000",
    output_index: 0,
  }
}

// -- Normalisation Unit Tests --

test normalise_after() {
  let range = interval.after(1000)
  when normalise(range) is {
    ToPosInf { from } -> from == 1000
    _ -> False
  }
}

test normalise_before() {
  let range = interval.before(2000)
  when normalise(range) is {
    FromNegInf { to } -> to == 2000
    _ -> False
  }
}

test normalise_between() {
  let range = interval.between(1000, 2000)
  when normalise(range) is {
    ClosedRange { from, to } -> from == 1000 && to == 2000
    _ -> False
  }
}

test normalise_everything() {
  let range =
    Interval {
      lower_bound: IntervalBound {
        bound_type: NegativeInfinity,
        is_inclusive: True,
      },
      upper_bound: IntervalBound {
        bound_type: PositiveInfinity,
        is_inclusive: True,
      },
    }
  when normalise(range) is {
    Always -> True
    _ -> False
  }
}

// -- Validator Tests --

test must_be_after_with_valid_range() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.after(2000),
    }
  time_check.spend(None, MustBeAfter { deadline: 1000 }, mock_oref(), tx)
}

test must_be_after_fails_when_before() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.after(500),
    }
  time_check.spend(None, MustBeAfter { deadline: 1000 }, mock_oref(), tx)
}

test must_be_before_with_valid_range() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.before(500),
    }
  time_check.spend(None, MustBeBefore { deadline: 1000 }, mock_oref(), tx)
}

test must_be_before_fails_when_after() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.before(2000),
    }
  time_check.spend(None, MustBeBefore { deadline: 1000 }, mock_oref(), tx)
}

test must_be_between_succeeds() {
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.between(100, 900),
    }
  time_check.spend(
    None,
    MustBeBetween { start: 0, end: 1000 },
    mock_oref(),
    tx,
  )
}

test must_be_between_fails_outside() fail {
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.between(100, 2000),
    }
  time_check.spend(
    None,
    MustBeBetween { start: 0, end: 1000 },
    mock_oref(),
    tx,
  )
}

// -- Property-Based Test --

test prop_after_always_valid_when_greater(
  params via fuzz.both(
    fuzz.int_between(1000, 1_000_000),
    fuzz.int_between(1, 999),
  ),
) {
  let (range_start, deadline) = params
  let tx =
    Transaction {
      ..transaction.placeholder,
      validity_range: interval.after(range_start),
    }
  time_check.spend(None, MustBeAfter { deadline }, mock_oref(), tx)
}
```

## Key Concepts Demonstrated

1. **`Interval` is not generic** — use `Interval` not `Interval<Int>`
2. **Bound access** — `range.lower_bound.bound_type` yields `Finite(n)`, `NegativeInfinity`, or `PositiveInfinity`
3. **Explicit import of constructors** — `use aiken/interval.{Finite, NegativeInfinity, PositiveInfinity}`
4. **Tuple pattern matching** — `when (lower, upper) is { (Finite(lo), Finite(hi)) -> ... }`
5. **`interval.after(t)`** — creates `[t, +inf)` range (ToPosInf)
6. **`interval.before(t)`** — creates `(-inf, t]` range (FromNegInf)
7. **`interval.between(a, b)`** — creates `[a, b]` range (ClosedRange)
8. **Manual `IntervalBound` construction** — needed for testing `Always` case
