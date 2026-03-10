# Security Patterns

## eUTxO Security Model

The eUTxO model eliminates several classes of vulnerabilities common in
account-based chains:

- **Reentrancy:** Impossible. No mutable global state.
- **Front-running:** Mitigated. Transaction execution is deterministic.
- **Gas estimation attacks:** Impossible. Fees are precisely predictable.
- **Unexpected state changes:** Impossible. Validators reason locally.

However, eUTxO introduces its own attack vectors that every validator must guard
against.

## Attack Vectors

### 1. Double Satisfaction

**What:** A single output satisfies the validation conditions of multiple script
inputs in the same transaction. An attacker creates a transaction consuming two
locked UTxOs but only providing one valid output.

**Example:** Two UTxOs each locked with "must produce output of equal value to
script address." Attacker consumes both, produces one output equal to the larger
amount, satisfying both validators.

**Mitigation:**
- Use authentication NFTs — each UTxO carries a unique token that must be
  present in its corresponding output
- Link inputs to outputs explicitly via redeemer (UTxO indexing pattern)

```aiken
// BAD — vulnerable to double satisfaction
spend(datum: Option<Datum>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
  // Just checks "some output" goes to script — could be shared
  list.any(tx.outputs, fn(o) { o.address == script_address })
}

// GOOD — links specific input to specific output via NFT
spend(datum: Option<Datum>, _redeemer: Data, oref: OutputReference, tx: Transaction) {
  expect Some(d) = datum
  // Must burn THIS UTxO's unique auth token
  assets.quantity_of(tx.mint, auth_policy, d.auth_token_name) == -1
}
```

### 2. Datum Hijacking

**What:** An attacker creates a UTxO at your script address with a crafted datum
that bypasses validation. Since anyone can send to a script address, the
validator must verify datum provenance.

**Example:** Vault expects datum with owner's key hash. Attacker sends a UTxO to
the vault address with their own key hash as owner, then claims it.

**Mitigation:**
- Authenticate UTxOs with minted NFTs — the minting policy validates the
  initial datum
- Verify datum content against trusted reference data
- Use parameterized validators where the owner is baked in

```aiken
// BAD — trusts whatever datum is at the script address
spend(datum: Option<Datum>, _redeemer: Data, _oref: OutputReference, tx: Transaction) {
  expect Some(d) = datum
  list.has(tx.extra_signatories, d.owner)
  // Attacker creates UTxO with their own key as owner
}

// GOOD — minting policy validates initial datum at creation time
mint(redeemer: MintAction, policy_id: PolicyId, tx: Transaction) {
  when redeemer is {
    Create { owner } -> {
      // Verify the output datum matches the authenticated owner
      let script_outputs = find_script_outputs(tx.outputs, script_hash)
      expect [output] = script_outputs
      expect InlineDatum(raw) = output.datum
      expect d: Datum = raw
      d.owner == owner && list.has(tx.extra_signatories, owner)
    }
  }
}
```

### 3. Unbounded Computation / Denial of Service

**What:** A validator performs computation proportional to the size of
user-controlled data (e.g., iterating over all inputs or outputs without bound).
An attacker crafts a transaction with many irrelevant inputs to exhaust the
execution budget.

**Mitigation:**
- Use redeemer indices to point directly to relevant inputs/outputs
- Avoid iterating over `tx.inputs` or `tx.outputs` without bounds
- Use the withdraw-zero trick to validate once per transaction

```aiken
// BAD — O(n) in number of outputs
let total_to_script =
  tx.outputs
    |> list.filter(fn(o) { o.address == script_address })
    |> list.foldl(0, fn(o, acc) { acc + assets.lovelace_of(o.value) })

// GOOD — redeemer tells us exactly where to look
type SpendRedeemer {
  Claim { output_index: Int }
}

spend(datum: Option<Datum>, redeemer: SpendRedeemer, _oref: OutputReference, tx: Transaction) {
  expect Claim { output_index } = redeemer
  expect Some(output) = list.at(tx.outputs, output_index)
  // Direct access, O(1)
  validate_output(output)
}
```

### 4. Reference Input Manipulation

**What:** Reference inputs are read-only inputs that don't require validation by
their locking script. An attacker can supply fake reference inputs with
manipulated data.

**Mitigation:**
- Authenticate reference inputs by checking for known NFTs
- Never trust data from reference inputs unless authenticated

```aiken
// BAD — trusts any reference input at the oracle address
expect Some(oracle_ref) =
  list.find(tx.reference_inputs, fn(i) { i.output.address == oracle_address })
expect InlineDatum(raw) = oracle_ref.output.datum
expect price: Int = raw  // Attacker supplies fake price

// GOOD — verify oracle NFT is present
expect Some(oracle_ref) =
  list.find(tx.reference_inputs, fn(i) {
    assets.quantity_of(i.output.value, oracle_policy, oracle_token) == 1
  })
```

### 5. Missing Redeemer Validation

**What:** Validator accepts any redeemer without checking which action is being
performed. An attacker can use an unintended redeemer to bypass checks.

**Mitigation:**
- Always pattern match on the redeemer
- Use typed redeemers (Aiken enforces this at the type level)
- Ensure exhaustive handling of all variants

### 6. Token Dust / Value Injection

**What:** Attacker sends worthless tokens to a script UTxO, bloating it and
potentially confusing value calculations.

**Mitigation:**
- Check exact value composition, not just ADA amount
- Reject outputs containing unexpected tokens
- Use `assets.without_lovelace()` to inspect non-ADA tokens

### 7. Missing Mint Field Check

**What:** An attacker includes unexpected minting/burning in a transaction that
interacts with your validator. If your validator doesn't check `tx.mint`, it
won't notice unauthorized token operations.

**Mitigation:**
- Always check `tx.mint` is empty or contains only expected operations
- For validators that use authentication tokens, verify exact mint quantities

## Security Checklist

For every validator, verify:

- [ ] **Signatures:** All required signatories are checked
- [ ] **Time:** Validity range is constrained appropriately
- [ ] **Value:** Input/output values are conserved or explicitly accounted for
- [ ] **Datum:** Datum content is validated, not just present
- [ ] **Redeemer:** All redeemer variants are handled exhaustively
- [ ] **Mint field:** Unexpected minting/burning is blocked
- [ ] **Double satisfaction:** Inputs are linked to specific outputs
- [ ] **Reference inputs:** Authenticated by NFT or similar
- [ ] **Continuing outputs:** State transitions are validated
- [ ] **Script address:** Outputs to script use correct address
- [ ] **Token composition:** No unexpected tokens in values
- [ ] **Computation bounds:** No unbounded iteration over tx fields

## Resources

- [Cardano CTF](https://github.com/vacuumlabs/cardano-ctf) — 25 challenges teaching real exploit patterns
- [Vacuumlabs vulnerability series](https://medium.com/vacuumlabs) — 6-part deep dive
- [CIP-52](https://cips.cardano.org/cip/CIP-0052) — Audit best practices
