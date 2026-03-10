# Aiken Smart Contract Skill

A [Claude Code skill](https://code.claude.com/docs/en/skills) for writing and testing [Aiken](https://aiken-lang.org) smart contracts on Cardano.

Following the [Agent Skills](https://agentskills.io) open standard — works with Claude Code, Cursor, Gemini CLI, VS Code Copilot, and 30+ other AI coding tools.

> **Status:** Early development. Being built and validated against known Cardano smart contract patterns as part of the [ADAvault](https://adavault.com) R2.1 smart contract development effort.

## What It Does

Gives your AI coding assistant deep knowledge of:

- **Aiken language** — syntax, types, modules, idioms, PlutusData encoding
- **Validator patterns** — spend, mint, withdraw, multi-purpose, parameterized, state machines
- **Testing** — unit tests, property-based testing with fuzz library, scenario testing
- **Security** — double satisfaction, datum hijacking, reference input manipulation, and other eUTxO-specific attack vectors
- **Standard library** — key modules (transaction, assets, interval, list, dict, crypto)
- **Design patterns** — withdraw-zero trick, UTxO indexing, TVMP, validity range normalisation

## Installation

### Project-Level (Recommended)

Copy into your project's `.claude/skills/` directory:

```bash
# From your Aiken project root
mkdir -p .claude/skills
cp -r /path/to/aiken-skill .claude/skills/aiken-smart-contract
```

Or clone directly:

```bash
mkdir -p .claude/skills
git clone https://github.com/adavault/aiken-skill.git .claude/skills/aiken-smart-contract
```

### Personal (All Projects)

```bash
cp -r /path/to/aiken-skill ~/.claude/skills/aiken-smart-contract
```

## Usage

The skill activates automatically when your conversation involves Aiken, validators, smart contracts, or Cardano on-chain development.

You can also invoke it explicitly:

```
/aiken-smart-contract
```

## Structure

```
├── SKILL.md                     # Core skill — workflow, signatures, patterns
├── reference/
│   ├── language.md              # Types, syntax, modules, encoding
│   ├── validators.md            # Validator architectures and recipes
│   ├── testing.md               # Unit, property-based, scenario testing
│   ├── security.md              # Attack vectors and mitigations
│   ├── stdlib.md                # Standard library reference
│   └── patterns.md              # Advanced design patterns
└── examples/
    ├── hello-world.md           # Simplest spend validator
    ├── vesting.md               # Time-locked spending
    └── gift-card.md             # Mint+spend dual handler with NFT
```

## Requirements

- [Aiken](https://aiken-lang.org) v1.1.x
- An AI coding assistant that supports [Agent Skills](https://agentskills.io)

## Contributing

This skill is being built iteratively. We're validating it against known Cardano smart contract patterns and refining as we develop real contracts.

If you're an SPO, Aiken developer, or Cardano builder and want to contribute patterns, examples, or corrections — PRs welcome.

## License

MIT — see [LICENSE](LICENSE).

## Credits

Built by [ADAvault](https://adavault.com) — Cardano stake pool operators since epoch 208.
