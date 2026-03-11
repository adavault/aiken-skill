# Auditing Methodology

A structured approach for reviewing Aiken smart contracts. Follow this
methodology when asked to audit, review, or check a validator for security
issues.

## Audit Process

### Phase 1: Understand the Contract

Before looking for bugs, understand what the contract is supposed to do.

1. **Read the types** — datum, redeemer, and any custom types. These define the
   contract's state model and allowed actions.
2. **Identify the validator purpose** — spend, mint, withdraw, governance, or
   multi-purpose. Each has different trust assumptions.
3. **Map the state transitions** — what inputs produce what outputs? Draw the
   state machine if applicable.
4. **Identify all actors** — who interacts with this contract? (owner, buyer,
   beneficiary, admin, anyone)
5. **Identify trust boundaries** — what data comes from trusted sources vs.
   user-supplied?

### Phase 2: Systematic Vulnerability Scan

Check each vulnerability category from the [security patterns](security.md).
For each, ask the specific question and examine the relevant code.

#### Authorization Checks

| Check | Question | What to look for |
|-------|----------|-----------------|
| Signatures | Who must sign for each redeemer? | `list.has(tx.extra_signatories, ...)` for every privileged action |
| Redeemer exhaustiveness | Are all variants handled? | `when redeemer is { ... }` — missing variants, catch-all `_` patterns |
| Time constraints | Is the validity range checked? | `interval.is_entirely_after` / `interval.is_entirely_before` |
| Missing authorization | Can anyone trigger sensitive actions? | Redeemers without signature or time checks |

#### Value Conservation

| Check | Question | What to look for |
|-------|----------|-----------------|
| Output values | Are output values verified? | `assets.lovelace_of(output.value) >= expected` |
| Token composition | Are unexpected tokens rejected? | `assets.without_lovelace(value)` checks |
| Mint field | Is `tx.mint` validated? | Ensure no unexpected minting/burning |
| Token names | Are all token names under the policy handled? | `dict.to_pairs(assets.tokens(tx.mint, policy_id))` |

#### Input/Output Linking

| Check | Question | What to look for |
|-------|----------|-----------------|
| Double satisfaction | Is each input linked to a specific output? | NFT auth tokens, redeemer indices, or unique identifiers |
| Continuing outputs | Is the output datum validated? | `expect InlineDatum(raw) = output.datum` with type checks |
| Output address | Is the full address checked (payment + staking)? | `output.address == expected_address` including stake credential |
| Output count | Is the number of script outputs constrained? | `expect [single_output] = script_outputs` |

#### Data Integrity

| Check | Question | What to look for |
|-------|----------|-----------------|
| Datum hijacking | Can an attacker create fake UTxOs at the script address? | NFT authentication on script UTxOs |
| Datum mutation | Are immutable datum fields protected in state transitions? | Field-by-field equality checks on continuing outputs |
| Datum growth | Can the datum grow without bound? | Lists or maps in datum that grow on each transition |
| Reference inputs | Are reference inputs authenticated? | NFT check before trusting reference input data |

#### Scalability & Liveness

| Check | Question | What to look for |
|-------|----------|-----------------|
| Unbounded computation | Does validation cost scale with user-controlled data? | `list.filter`/`list.foldl` over `tx.inputs`/`tx.outputs` without indices |
| UTxO contention | Is there a single shared UTxO that multiple users compete for? | State machine patterns without batching |
| Stuck states | Can the contract reach a state where no redeemer can succeed? | Missing "emergency" or "admin" redeemers |
| Minimum ADA | Can outputs be created below the minimum ADA threshold? | Value calculations that might produce sub-minimum outputs |

### Phase 3: Parameterized Validator Analysis

If the validator takes parameters:

1. **Parameter trust** — who supplies the parameters? Are they validated at
   deployment time or trusted?
2. **Zero/empty parameters** — what happens with empty byte arrays, zero
   integers, empty lists?
3. **Hash sensitivity** — different parameters produce different script
   addresses. Are all required parameters included?
4. **Parameter encoding** — CBOR encoding of parameters must match what the
   off-chain code supplies. Mismatches produce wrong script addresses silently.
   See [offchain.md](offchain.md) for MeshJS encoding patterns and common pitfalls.

### Phase 4: Multi-Validator Interaction

For contracts with multiple validators (e.g., mint + spend):

1. **Cross-validator invariants** — does the minting policy enforce rules that
   the spend validator relies on?
2. **Shared script hash** — multi-purpose validators share a hash. Can an
   attacker exploit the mint handler to bypass the spend handler?
3. **Reference scripts** — if using reference scripts, can an attacker
   substitute a different script?
4. **Withdrawal delegation** — if using withdraw-zero for batching, is the
   staking validator properly checked?

### Phase 5: Test Coverage Assessment

Review the test suite:

1. **Happy path coverage** — does every redeemer variant have at least one
   passing test?
2. **Failure path coverage** — is every validation check tested with a
   `fail` test that proves it actually rejects?
3. **Boundary conditions** — are edge cases tested? (zero values, empty lists,
   exact deadlines, minimum thresholds)
4. **Property-based tests** — are there fuzz tests for critical invariants?
   (e.g., "owner can always withdraw", "stranger can never claim")
5. **Missing tests** — for each validation condition in the validator, is there
   a corresponding test that would fail if that condition were removed?

## Severity Classification

Based on MLabs' taxonomy, classify findings by impact:

| Severity | Definition | Examples |
|----------|------------|---------|
| **Critical** | Direct loss of funds or permanent lock | Bypassed authorization, unspendable outputs |
| **High** | Protocol disruption or indirect fund loss | UTxO contention DoS, staking reward theft |
| **Medium** | Degraded security or limited exploit window | Missing time checks, unbounded datum growth |
| **Low** | Best practice violation, no direct exploit | Unused redeemer variants, suboptimal computation |
| **Informational** | Code quality, documentation, style | Missing traces, unclear variable names |

## Audit Report Format

When presenting findings, use this structure per issue:

```
### [SEVERITY] Issue Title

**Category:** (e.g., Double Satisfaction, Datum Hijacking, etc.)
**Location:** file.ak:line_number
**Description:** What the vulnerability is and why it matters.
**Exploitation:** How an attacker would exploit this.
**Recommendation:** Specific code change to fix it.
```

## CIP-52 Compliance Checklist

For contracts heading toward professional audit or mainnet deployment,
verify readiness against [CIP-52](https://cips.cardano.org/cip/CIP-0052):

### Documentation
- [ ] **Specification document** — precise description of contract behavior, not just code comments
- [ ] **Transaction formats** — documented for every interaction (which inputs, outputs, signatories, validity range)
- [ ] **State machine diagram** — if applicable, all states and transitions documented
- [ ] **Threat model** — identified actors, trust assumptions, and known risks
- [ ] **Intended properties** — what the contract must guarantee (e.g., "only owner can withdraw")

### Code Quality
- [ ] **Reproducible build** — `aiken.toml` pins stdlib version, build produces identical `plutus.json`
- [ ] **Source code matches deployment** — plutus.json hash verified against on-chain script hash
- [ ] **No dead code** — all functions and types are used
- [ ] **Explicit error traces** — `?` operator or `trace` on every validation condition for debugging

### Testing
- [ ] **Unit tests** — every redeemer variant, happy + failure paths
- [ ] **Property-based tests** — critical invariants fuzzed (authorization, value conservation)
- [ ] **Boundary tests** — zero, one, max values; exact deadline; minimum ADA
- [ ] **Coverage metric** — every `if`/`when`/`expect` branch exercised
- [ ] **DoS scenario tests** — large input/output counts, maximum datum sizes

### Deployment Readiness
- [ ] **Parameterization validated** — parameters produce expected script addresses
- [ ] **Off-chain code reviewed** — transaction building matches on-chain expectations
- [ ] **Upgrade path documented** — how to migrate if a vulnerability is found post-deployment
- [ ] **Monitoring plan** — how to detect anomalous transactions against the contract

## Tooling

### Current (2025)

- **Aiken's test runner** — executes on the real CEK machine; test results are
  production-accurate. Property-based testing via `aiken/fuzz` library.
- **`aiken check`** — type checking + test execution. Always run before deployment.
- **`aiken build`** — generates `plutus.json` blueprint with script hashes.

### Emerging

- **SmartCodeVerifier** (IOG) — formal verification tool combining Lean4 proof
  system with SMT solvers (Z3). Translates annotated Aiken code to formal proofs
  automatically. Supports verification against actual UPLC bytecode. Can
  integrate into CI/CD pipelines. Rolling deployment throughout 2025.
- **Plu-Stan** (IOG) — static analyzer for PlutusTx (not directly for Aiken,
  but applicable to the compiled UPLC output). Rule categories: security
  patterns and performance optimizations.

### Audit Firms with Cardano/Aiken Experience

From CIP-52 Appendix D and ecosystem track record:
- Tweag, WellTyped — formal methods expertise
- Anastasia Labs — Aiken design patterns library, audited Lenfi
- TxPipe — infrastructure + audit, co-audited Lenfi
- MLabs — published vulnerability taxonomy, Plutus/Aiken experience
- Vacuumlabs — published 6-part vulnerability series, built Cardano CTF
- Runtime Verification — K framework formal verification
- CertiK — broad smart contract auditing

## Learning Resources

Ordered by difficulty:

1. **[Aiken examples in this skill](../SKILL.md)** — 23 compiler-validated
   examples covering all handler types, with security notes on each
2. **[Cardano CTF](https://github.com/vacuumlabs/cardano-ctf)** — 25 hands-on
   challenges. Original series (11) teaches fundamentals; banking series (14)
   covers production-grade exploits
3. **[MLabs vulnerability guide](https://www.mlabs.city/blog/common-plutus-security-vulnerabilities)** —
   11-category taxonomy with property-test-impact structure
4. **[Plutonomicon](https://plutonomicon.github.io/plutonomicon/vulnerabilities)** —
   eUTxO-specific attack patterns including oracle and concurrency attacks
5. **[Vacuumlabs auditing blog](https://medium.com/@vacuumlabs_auditing)** —
   real-world findings from professional audits
6. **[CIP-52](https://cips.cardano.org/cip/CIP-0052)** — the formal audit
   standard for Cardano smart contracts
