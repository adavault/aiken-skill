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

**Confirmed working** — See [oracle-feed.md](../examples/oracle-feed.md).

**Mitigation:**
- Authenticate reference inputs by checking for known NFTs
- Never trust data from reference inputs unless authenticated

```aiken
// BAD — trusts any reference input at the oracle address
expect Some(oracle_ref) =
  list.find(tx.reference_inputs, fn(i) { i.output.address == oracle_address })
expect InlineDatum(raw) = oracle_ref.output.datum
expect price: Int = raw  // Attacker supplies fake price

// GOOD — verify oracle NFT is present (confirmed working)
expect Some(ref_input) = list.at(tx.reference_inputs, redeemer.oracle_index)
let has_nft =
  (assets.quantity_of(ref_input.output.value, oracle_policy, oracle_token) == 1)?
expect True = has_nft
expect InlineDatum(raw) = ref_input.output.datum
expect oracle: OracleDatum = raw
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

### 8. Other Token Name (Same Policy)

**What:** A minting policy validates the minting of one specific token name but
doesn't restrict minting of *other* token names under the same policy ID. An
attacker mints additional tokens under your policy, potentially creating fake
authentication tokens.

**Example:** Policy checks `dict.to_pairs(minted) == [Pair("AUTH", 1)]` for
minting, but this only validates when minting AUTH. The attacker submits a
separate transaction minting "FAKE" under the same policy — the minting
policy's redeemer doesn't cover this case.

**Mitigation:**
- Validate the **entire** `assets.tokens(tx.mint, policy_id)` dict, not just one entry
- Ensure the minting policy handles ALL possible token names
- Use `expect [Pair(name, qty)] = dict.to_pairs(minted)` to enforce exactly one token name

```aiken
// BAD — only checks for expected token name
mint(redeemer: Action, policy_id: PolicyId, tx: Transaction) {
  when redeemer is {
    MintAuth -> assets.quantity_of(tx.mint, policy_id, "AUTH") == 1
    // Attacker mints "FAKE" — no redeemer case handles it
  }
}

// GOOD — validates entire token dict
mint(redeemer: Action, policy_id: PolicyId, tx: Transaction) {
  let minted = assets.tokens(tx.mint, policy_id)
  when redeemer is {
    MintAuth -> dict.to_pairs(minted) == [Pair("AUTH", 1)]
    BurnAuth -> dict.to_pairs(minted) == [Pair("AUTH", -1)]
  }
}
```

### 9. Unbounded Datum Growth

**What:** A continuing output pattern (state machine) allows datum fields to
grow without limit. Eventually the datum exceeds Cardano's 16KB transaction
size limit, making the UTxO permanently unspendable — funds are locked forever.

**Example:** A validator that appends to a `List<ByteArray>` in the datum on
each state transition. After enough transitions, the datum cannot fit in a
transaction.

**Mitigation:**
- Set hard limits on datum collection sizes in the validator
- Use merkle roots instead of storing full lists
- Prune old entries on each transition
- Consider off-chain storage with on-chain commitment hashes

```aiken
// BAD — datum grows without limit
type State {
  history: List<Action>,  // Grows every transition
  count: Int,
}

// GOOD — bounded history, old entries pruned
type State {
  recent_history: List<Action>,  // Capped at N entries
  count: Int,
}

// In validator:
let new_history = [action, ..state.recent_history] |> list.take(10)
```

### 10. UTxO Contention

**What:** When multiple users need to interact with the same script UTxO
(e.g., a shared pool or state machine), transactions conflict because only one
can consume a given UTxO per block. This creates a bottleneck where most
transactions fail.

**Impact:** Protocol stalls under load. Real problem for DEXes and lending protocols.

**Mitigation:**
- **Batching via withdraw-zero** — collect individual orders as separate UTxOs,
  batch-process them in a single transaction (see [withdraw-zero](../examples/withdraw-zero.md))
- **UTxO fan-out** — split state across multiple UTxOs that can be consumed independently
- **Off-chain ordering** — use a batcher service to sequence transactions
- **Reference inputs** — read shared state without consuming it (where possible)

### 11. Insufficient Staking Credential Control

**What:** A validator checks the payment credential of output addresses but
ignores the staking credential. An attacker can redirect staking rewards by
supplying outputs with the correct payment credential but their own staking key.

**Example:** Continuing output validator checks
`output.address.payment_credential == script_cred` but doesn't verify
`output.address.stake_credential`. Attacker earns staking rewards from protocol
ADA by setting their stake key on the output address.

**Mitigation:**
- Validate the **full address** (payment + staking) for protocol-owned outputs
- Use `address.from_script(hash) |> address.with_delegation_script(stake_hash)`
  to construct expected addresses with staking credentials
- For user-facing outputs, staking credential is user's choice (no check needed)

```aiken
// BAD — only checks payment credential
let correct_addr =
  (output.address.payment_credential == script_cred)?

// GOOD — checks full address including staking credential
let expected_address =
  address.from_script(script_hash)
    |> address.with_delegation_script(stake_script_hash)
let correct_addr =
  (output.address == expected_address)?
```

## Security Checklist

For every validator, verify:

- [ ] **Signatures:** All required signatories are checked
- [ ] **Time:** Validity range is constrained appropriately
- [ ] **Value:** Input/output values are conserved or explicitly accounted for
- [ ] **Datum:** Datum content is validated, not just present
- [ ] **Datum size:** Datum cannot grow unbounded across transitions
- [ ] **Redeemer:** All redeemer variants are handled exhaustively
- [ ] **Mint field:** Unexpected minting/burning is blocked
- [ ] **Token names:** All token names under your policy are validated
- [ ] **Double satisfaction:** Inputs are linked to specific outputs
- [ ] **Reference inputs:** Authenticated by NFT or similar
- [ ] **Continuing outputs:** State transitions are validated
- [ ] **Script address:** Outputs to script use correct address (payment + staking)
- [ ] **Staking credentials:** Protocol-owned outputs have correct staking key
- [ ] **Token composition:** No unexpected tokens in values
- [ ] **Computation bounds:** No unbounded iteration over tx fields
- [ ] **Contention:** Shared state doesn't create single-UTxO bottleneck

## Resources

- [Cardano CTF](https://github.com/vacuumlabs/cardano-ctf) — 25 challenges teaching real exploit patterns (original 11 + banking series 14)
- [Vacuumlabs vulnerability series](https://medium.com/@vacuumlabs_auditing) — 6-part deep dive on most-audited vulnerabilities
- [MLabs common vulnerabilities](https://www.mlabs.city/blog/common-plutus-security-vulnerabilities) — 11-category vulnerability taxonomy with severity classification
- [Plutonomicon vulnerabilities](https://plutonomicon.github.io/plutonomicon/vulnerabilities) — eUTxO-specific attack patterns including oracle and concurrency attacks
- [CIP-52](https://cips.cardano.org/cip/CIP-0052) — Cardano audit best practices standard
- [Auditing guide](auditing.md) — Structured audit methodology for reviewing Aiken validators
