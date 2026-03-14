# Aiken Smart Contract Skill

A [Claude Code skill](https://code.claude.com/docs/en/skills) for writing, testing, and auditing [Aiken](https://aiken-lang.org) smart contracts on Cardano.

Following the [Agent Skills](https://agentskills.io) open standard — works with Claude Code, Cursor, Gemini CLI, VS Code Copilot, and 30+ other AI coding tools.

## Summary

| Metric | Count |
|--------|-------|
| Reference docs | 10 |
| Example validators | 27 |
| Security categories | 11 |
| Audit methodology phases | 5 |
| Unit tests validated | 249 |
| E2E tested on preview | 61 operations across 21 contracts |

This skill gives your AI coding assistant deep, compiler-validated knowledge of Aiken smart contract development — from basic spend validators through CIP-113 programmable tokens and multi-validator governance patterns. Every example compiles and every test passes against Aiken v1.1.21 with Plutus V3.

## Features

### Reference Documentation (10 files)

- **Language** — types, syntax, modules, PlutusData encoding, Plutus V3 specifics
- **Validators** — spend, mint, withdraw, vote, publish, propose — all 6 handler types
- **Testing** — unit tests, property-based testing with fuzz library, scenario testing
- **Security** — 11 attack vectors: double satisfaction, datum hijacking, reference input manipulation, unbounded computation, and more
- **Auditing** — 5-phase methodology: understand, scan, parameterized analysis, multi-validator interaction, test coverage
- **Standard Library** — transaction, assets, interval, list, dict, crypto, bytearray
- **Patterns** — withdraw-zero trick, UTxO indexing, TVMP, validity range normalisation, 4 migration strategies
- **Gotchas** — 20+ learnings covering type system, values/assets, certificates, governance, testing, compiler behaviour
- **Off-chain** — MeshJS integration guide with transaction building, UTxO management, multi-signer patterns
- **CIP-113** — Programmable token protocol: sorted linked list registry, multi-validator mint, transfer lifecycle

### Example Validators (27 files, 8 phases)

Progressive difficulty from hello-world to CIP-113 multi-validator patterns:

| Phase | Examples | Concepts |
|-------|----------|----------|
| 1. Fundamentals | hello-world, vesting, gift-card | Spend, time-locks, mint+spend dual handler |
| 2. Intermediate | escrow, multi-sig, state-machine, nft-vault | Multi-party, continuing outputs, state transitions |
| 3. Advanced | withdraw-zero, dead-mans-switch, marketplace, multi-beneficiary | Withdraw trick, inactivity triggers, royalties |
| 4. Patterns | notary, tvmp, utxo-indexer, validity-range, pool-restriction | One-shot mint, tx-level validation, UTxO ordering |
| 5. Governance | governance-vote, governance-publish, governance-propose | All 3 governance handler types |
| 6. Advanced Mint | dao-vote, oracle-feed, tagged-output | DAO tokens, oracle data, tagged outputs |
| 7. Multi-handler | multi-sig (parameterized) | Multi-purpose validators sharing script hash |
| 8. CIP Extensions | reference-input, reference-scripts, cip68-metadata, merkle-proof, dutch-auction | CIP-31, CIP-33, CIP-68, Merkle proofs, auctions |

### Security Coverage (11 categories)

- Double satisfaction
- Datum hijacking
- Reference input manipulation
- Unbounded computation / budget exhaustion
- Missing authorization checks
- Token name confusion
- Validity range exploitation
- Redeemer manipulation
- Continuing output validation gaps
- Multi-validator interaction attacks
- Governance-specific vectors

### Audit Methodology (5 phases)

1. **Understand** — types, purpose, state transitions, actors, trust boundaries
2. **Vulnerability scan** — systematic check against all 11 security categories
3. **Parameterized analysis** — parameter validation, upgrade paths, migration safety
4. **Multi-validator interaction** — cross-script dependencies, shared state, policy coordination
5. **Test coverage** — gap analysis, edge cases, property-based test recommendations

## Validation

All examples are compiler-validated with 249 passing tests against Aiken v1.1.21 (Plutus V3, stdlib v3.0.0).

End-to-end validation on Cardano preview testnet via two companion projects:

- **[cardano-notary](https://github.com/ADAvault/cardano-notary)** — 61/61 E2E operations across 21 contracts, covering all major patterns (mint, spend, time-lock, multi-sig, state machine, withdraw-zero, marketplace, CIP-68, Merkle proofs, reference scripts)
- **[programmable-tokens](https://github.com/ADAvault/programmable-tokens)** — CIP-113 full lifecycle: protocol params, sorted registry, stake registration, multi-validator mint (4 scripts in 1 tx), programmable transfer (3 scripts in 1 tx), multi-holder round-trip

## Installation

### Project-Level (Recommended)

```bash
# From your Aiken project root
mkdir -p .claude/skills
git clone https://github.com/ADAvault/cardano-skill.git .claude/skills/aiken-smart-contract
```

### Personal (All Projects)

```bash
git clone https://github.com/ADAvault/cardano-skill.git ~/.claude/skills/aiken-smart-contract
```

## Usage

The skill activates automatically when your conversation involves Aiken, validators, smart contracts, or Cardano on-chain development.

You can also invoke it explicitly:

```
/aiken-smart-contract
```

## Structure

```
cardano-skill/
├── SKILL.md                        # Core skill definition — workflow, signatures, decision tree
├── reference/
│   ├── language.md                 # Types, syntax, modules, PlutusData encoding
│   ├── validators.md               # All 6 handler types, architectures, recipes
│   ├── testing.md                  # Unit, property-based, scenario testing
│   ├── security.md                 # 11 attack vectors and mitigations
│   ├── auditing.md                 # 5-phase audit methodology
│   ├── stdlib.md                   # Standard library reference
│   ├── patterns.md                 # Advanced patterns + 4 migration strategies
│   ├── gotchas.md                  # 20+ compiler and runtime learnings
│   ├── offchain.md                 # MeshJS off-chain integration guide
│   └── cip113.md                   # CIP-113 programmable token protocol
└── examples/
    ├── hello-world.md              # Simplest spend validator
    ├── vesting.md                  # Time-locked spending
    ├── gift-card.md                # Mint+spend dual handler with NFT
    ├── escrow.md                   # Multi-party escrow
    ├── multi-sig.md                # M-of-N multi-signature
    ├── state-machine.md            # Continuing output state transitions
    ├── nft-vault.md                # NFT-gated vault
    ├── withdraw-zero.md            # Withdraw-zero trick pattern
    ├── dead-mans-switch.md         # Inactivity-triggered release
    ├── marketplace.md              # NFT marketplace with royalties
    ├── multi-beneficiary.md        # Multiple payment outputs
    ├── notary.md                   # Proof-of-existence (one-shot mint)
    ├── tvmp.md                     # Transaction-level minting policy
    ├── utxo-indexer.md             # UTxO input/output ordering
    ├── validity-range.md           # Time-based validation
    ├── pool-restriction.md         # Pool-specific delegation checks
    ├── governance-vote.md          # Governance vote handler
    ├── governance-publish.md       # Governance publish handler
    ├── governance-propose.md       # Governance propose handler
    ├── dao-vote.md                 # DAO governance token voting
    ├── oracle-feed.md              # Oracle data feed
    ├── tagged-output.md            # Tagged output routing
    ├── reference-input.md          # CIP-31 reference inputs
    ├── reference-scripts.md        # CIP-33 reference scripts
    ├── cip68-metadata.md           # CIP-68 on-chain metadata
    ├── merkle-proof.md             # Merkle proof verification
    └── dutch-auction.md            # Dutch auction with price decay
```

## Requirements

- [Aiken](https://aiken-lang.org) v1.1.x
- An AI coding assistant that supports [Agent Skills](https://agentskills.io)

## Production References

Built with patterns from established Cardano projects:

- Minswap, SundaeSwap, Lenfi — DEX and lending protocols
- SpaceBudz Nebula — NFT marketplace
- Anastasia Labs — Smart contract library
- Logical Mechanism — Advanced patterns
- MeshJS — Off-chain transaction building

## Contributing

PRs welcome — especially from SPOs, Aiken developers, and Cardano builders. Areas of interest:

- New validator patterns or examples
- Security findings or additional attack vectors
- Corrections to existing reference material
- Off-chain integration patterns for other frameworks

## Key Findings from Validation

Compiling all examples and running E2E tests against real infrastructure uncovered patterns not documented elsewhere:

- **`tx.inputs` are sorted lexicographically** — redeemer indices (UTxO indexer pattern) must match sorted order, not insertion order
- **`invalidBefore` needs 180s safety margin** — ledger tip can lag 60-200s behind wall clock on preview (Poisson block distribution)
- **Conway registration is permissionless** — `registerStakeCertificate()` needs no script witness, redeemer, or collateral
- **Multi-handler validators share script hash** — policy ID = spend script hash for the same compiled code
- **Aiken `else` handler blocks unmatched purposes** — can't deregister a staking cred if the script only has a `withdraw` handler

All findings are documented in [gotchas.md](reference/gotchas.md) and in each example's Key Concepts section.

## Sources

Built from:

- **Aiken documentation and standard library** (v1.1.21, stdlib v3.0.0)
- **Production Cardano projects** — Minswap, SundaeSwap, Lenfi, SpaceBudz Nebula, Anastasia Labs
- **MeshJS SDK** — off-chain transaction building and testing patterns
- **Cardano Improvement Proposals** — CIP-31, CIP-33, CIP-68, CIP-113
- **Compiler validation** — 249 tests passing against Aiken v1.1.21 with Plutus V3
- **Preview testnet deployment** — 61 E2E operations across 21 contracts via [cardano-notary](https://github.com/ADAvault/cardano-notary) and [programmable-tokens](https://github.com/ADAvault/programmable-tokens)

## Companion Skill

See also: [midnight-skill](https://github.com/ADAvault/midnight-skill) — the equivalent skill for Compact smart contracts on Midnight (privacy, ZK proofs, shielded state).

## License

MIT — see [LICENSE](LICENSE).

## Credits

Built by [ADAvault](https://adavault.com) — Cardano stake pool operators since epoch 208.
