# Off-Chain Integration with MeshJS

Building transactions that interact with Aiken smart contracts. Covers
MeshJS setup, transaction building, datum/redeemer encoding, signing,
and integration testing on preview/preprod testnets.

All patterns verified against MeshJS v1.9.x-beta and Aiken v1.1.21.

## Provider Setup

### Ogmios + Kupo (Node Infrastructure)

```typescript
import { KupoProvider, OgmiosProvider } from "@meshsdk/provider";

// KupoProvider implements IFetcher (UTxO queries)
const kupo = new KupoProvider("http://localhost:1442");

// OgmiosProvider implements IEvaluator + ISubmitter (tx evaluation + submission)
const ogmios = new OgmiosProvider("http://localhost:1337");
```

**Critical: Node.js 20 WebSocket polyfill.** OgmiosProvider uses the browser
WebSocket API. Node.js 20 lacks global WebSocket — you must polyfill:

```typescript
import WebSocket from "ws";
(globalThis as any).WebSocket = WebSocket;
```

Node.js 22+ has native WebSocket and does not need this.

### Kupo Targeted Indexing

Kupo's `--match "*"` indexes everything (~50GB+ on preview). Use targeted
credential matching to keep disk usage low (~50MB):

```bash
# Index only specific wallet/script credentials
kupo --match "credential_hash/*" --node-socket /ipc/node.socket ...
```

Add script addresses when deploying contracts — restart Kupo with additional
`--match` params.

## Wallet Setup

### AppWallet (Server-Side from CLI Keys)

```typescript
import { AppWallet } from "@meshsdk/wallet";

const wallet = new AppWallet({
  networkId: 0,          // 0 = testnet, 1 = mainnet
  fetcher: kupo,
  submitter: ogmios,
  key: {
    type: "cli",
    payment: signingKeyHex,  // 64 hex chars (32 bytes), with or without 5820 prefix
  },
});
await wallet.init();
```

### Loading CLI Signing Keys

cardano-cli produces TextEnvelope format with CBOR-wrapped keys:

```typescript
function loadSigningKey(path: string): string {
  const raw = readFileSync(path, "utf-8");
  const envelope = JSON.parse(raw);
  const cborHex: string = envelope.cborHex;
  // Strip CBOR prefix: 5820 = 32-byte bytestring
  return cborHex.startsWith("5820") ? cborHex.slice(4) : cborHex;
}
```

### Enterprise vs Base Address

**Gotcha:** `wallet.getPaymentAddress()` returns a base address (with stake
component). If the funded wallet uses an enterprise address (no stake key),
UTxO lookups will fail because the addresses don't match.

```typescript
// BAD — returns base address with stake key
const addr = wallet.getPaymentAddress();

// GOOD — returns enterprise address (no stake key)
const addr = wallet.getEnterpriseAddress();
```

### Extracting Key Hash

`toBytes()` returns a hex string (not Uint8Array). The first 2 hex chars are
the address header byte, followed by 56 hex chars (28 bytes) of payment key
hash:

```typescript
const usedAddr = wallet.getUsedAddress(0, 0, "enterprise");
const addrHex = usedAddr.toBytes() as unknown as string;
const keyHashHex = addrHex.slice(2, 58);  // 28-byte payment key hash
```

## Parameterized Scripts

### Loading and Applying Parameters

Use `MintingBlueprint` (or the appropriate blueprint class) with `"JSON"` format:

```typescript
import { MintingBlueprint } from "@meshsdk/core";

const blueprint = new MintingBlueprint("V3");
blueprint.paramScript(
  compiledCode,                              // from plutus.json
  [{ bytes: keyHashHex }, { int: feeAmount }],  // parameters
  "JSON"                                     // MUST be "JSON", not default
);

const policyId = blueprint.hash;
const scriptCbor = blueprint.cbor;
```

**Critical:** The default format (Mesh type) causes "Cannot convert undefined
to a BigInt" errors. Always pass `"JSON"` as the third argument.

### Blueprint Extraction

```typescript
function loadCompiledCode(blueprintPath: string, title: string): string {
  const blueprint = JSON.parse(readFileSync(blueprintPath, "utf-8"));
  const validator = blueprint.validators.find(
    (v: { title: string }) => v.title === title
  );
  if (!validator) throw new Error(`${title} not found in blueprint`);
  return validator.compiledCode;
}

// Title format: "module.validator_name.handler"
const code = loadCompiledCode("contract/plutus.json", "notary.notary.mint");
```

## Plutus Data Encoding (JSON Format)

MeshJS supports three data encoding formats: Mesh, JSON, and CBOR. **Always
use JSON format** for datum and redeemer values to avoid BigInt serialization
errors.

### Constructors

```typescript
// Aiken: ConStr0 (first constructor)
{ constructor: 0, fields: [...] }

// Aiken: ConStr1 (second constructor)
{ constructor: 1, fields: [...] }

// Nested constructors
{
  constructor: 0,
  fields: [
    { constructor: 0, fields: [{ bytes: "abcd" }, { int: 42 }] }
  ]
}
```

**Critical:** The JSON key is `constructor`, NOT `alternative`. The
`alternative` key is for the Mesh type format. Using `alternative` with
`"JSON"` format produces "Malformed Plutus data json" errors.

### Primitive Types

```typescript
// ByteArray (hex-encoded)
{ bytes: "abcdef0123456789" }

// Integer
{ int: 42 }

// List
{ list: [{ int: 1 }, { int: 2 }, { int: 3 }] }

// Map (key-value pairs)
{ map: [{ k: { bytes: "key" }, v: { int: 100 } }] }
```

### Option Encoding

Aiken's `Option<T>` maps to Plutus constructors:

```typescript
// None = ConStr1 with no fields
{ constructor: 1, fields: [] }

// Some(value) = ConStr0 wrapping the value
{ constructor: 0, fields: [{ bytes: "the_value_hex" }] }
```

### String to Hex

Aiken `ByteArray` values that represent text must be hex-encoded:

```typescript
function strToHex(s: string): string {
  return Buffer.from(s).toString("hex");
}

// Example: "SHA-256" → "5348412d323536"
{ bytes: strToHex("SHA-256") }
```

### Complete Datum Example

For this Aiken type:

```aiken
pub type NotaryDatum {
  document_hash: ByteArray,
  hash_algorithm: ByteArray,
  uri: Option<ByteArray>,
  notarizer: ByteArray,
  description: Option<ByteArray>,
}
```

The off-chain encoding is:

```typescript
const datum = {
  constructor: 0,
  fields: [
    { bytes: documentHashHex },
    { bytes: strToHex("SHA-256") },
    { constructor: 1, fields: [] },                              // uri: None
    { bytes: notarizerKeyHash },
    { constructor: 0, fields: [{ bytes: strToHex("some text") }] }, // description: Some
  ],
};
```

## Transaction Building

### MeshTxBuilder Setup

```typescript
import { MeshTxBuilder } from "@meshsdk/core";

const txBuilder = new MeshTxBuilder({
  fetcher: kupo,       // IFetcher — UTxO queries
  submitter: ogmios,   // ISubmitter — tx submission
  evaluator: ogmios,   // IEvaluator — Plutus script evaluation
});
```

### Minting Transaction

```typescript
await txBuilder
  // Input (consume UTxO for one-shot uniqueness)
  .txIn(txHash, txIndex)
  // Mint with Plutus V3
  .mintPlutusScriptV3()
  .mint("1", policyId, tokenName)
  .mintingScript(scriptCbor)
  .mintRedeemerValue(redeemer, "JSON")   // ← JSON format
  // Output with inline datum
  .txOut(walletAddress, [
    { unit: "lovelace", quantity: "2000000" },
    { unit: policyId + tokenName, quantity: "1" },
  ])
  .txOutInlineDatumValue(datum, "JSON")  // ← JSON format
  // Collateral (required for all Plutus scripts)
  .txInCollateral(collateralTxHash, collateralTxIndex)
  // Required signer
  .requiredSignerHash(keyHashHex)
  // Change
  .changeAddress(walletAddress)
  // Signing key
  .signingKey(signingKeyHex)
  .complete();
```

### Burning Transaction

```typescript
const burnRedeemer = { constructor: 1, fields: [] };  // Burn variant

await txBuilder
  .txIn(nftTxHash, nftTxIndex)
  .mintPlutusScriptV3()
  .mint("-1", policyId, tokenName)           // quantity = "-1"
  .mintingScript(scriptCbor)
  .mintRedeemerValue(burnRedeemer, "JSON")
  .txInCollateral(collateralTxHash, collateralTxIndex)
  .requiredSignerHash(keyHashHex)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  .complete();
```

### Spending Transaction (Spend Validator)

```typescript
import { SpendingBlueprint } from "@meshsdk/core";

// Non-parameterized spend validator
const blueprint = new SpendingBlueprint("V3", 0, "");
blueprint.noParamScript(compiledCode);

// Parameterized spend validator (same params as minting counterpart)
const blueprint = new SpendingBlueprint("V3", 0, "");
blueprint.paramScript(compiledCode, params, "JSON");

const scriptHash = blueprint.hash;
const scriptCbor = blueprint.cbor;
const scriptAddress = blueprint.address;
```

Build the spending transaction:

```typescript
await txBuilder
  // Spend the script UTxO — order matters: spendingPlutusScriptV3 first
  .spendingPlutusScriptV3()
  .txIn(lockedUtxo.input.txHash, lockedUtxo.input.outputIndex)
  .txInScript(scriptCbor)
  .txInInlineDatumPresent()           // reads datum from the UTxO
  .txInRedeemerValue(redeemer, "JSON")
  // Collateral
  .txInCollateral(collateralTxHash, collateralTxIndex)
  // Required signer (if validator checks tx.extra_signatories)
  .requiredSignerHash(keyHashHex)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  .complete();
```

### Combined Mint + Spend Transaction

Some patterns require both minting and spending in the same transaction (e.g.,
gift card redeem: burn the NFT + unlock the ADA). Chain both handlers:

```typescript
await txBuilder
  // 1. Spend the script UTxO (unlock ADA)
  .spendingPlutusScriptV3()
  .txIn(lockedUtxo.input.txHash, lockedUtxo.input.outputIndex)
  .txInScript(spendScriptCbor)
  .txInInlineDatumPresent()
  .txInRedeemerValue(spendRedeemer, "JSON")
  // 2. Burn the NFT
  .mintPlutusScriptV3()
  .mint("-1", policyId, tokenName)
  .mintingScript(mintScriptCbor)
  .mintRedeemerValue(burnRedeemer, "JSON")
  // Shared requirements
  .txInCollateral(collateralTxHash, collateralTxIndex)
  .requiredSignerHash(keyHashHex)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  .complete();
```

### Time-Locked Transaction (Validity Range)

For validators that check `interval.is_entirely_after(tx.validity_range, deadline)`,
set `invalidBefore` to a slot after the deadline:

```typescript
// Preview testnet: slot 0 = 2022-10-25T00:00:00Z = 1666656000 Unix seconds
// 1 second per slot
const PREVIEW_SLOT_ZERO_UNIX = 1666656000;

function posixMsToSlot(posixMs: number): number {
  return Math.floor(posixMs / 1000) - PREVIEW_SLOT_ZERO_UNIX;
}

const deadlineSlot = posixMsToSlot(deadlineMs);
const invalidBeforeSlot = deadlineSlot + 1;    // ensures "entirely after"
const invalidHereafterSlot = currentSlot + 900; // 15 min TTL

await txBuilder
  .spendingPlutusScriptV3()
  // ... script input chain ...
  .invalidBefore(invalidBeforeSlot)
  .invalidHereafter(invalidHereafterSlot)
  // ... collateral, signer, change, signing ...
  .complete();
```

**Slot conversion varies by network:**
- Preview: slot 0 = 1666656000 (Oct 2022), 1s/slot
- Preprod: slot 0 = 1654041600 (Jun 2022), 1s/slot
- Mainnet: use `ogmios.querySystemStart()` and account for Byron/Shelley eras

### Continuing Output (State Machine / Check-In Pattern)

When spending a script UTxO and sending a new output back to the same script
address with updated datum (state transitions, check-ins, counter increments):

```typescript
await txBuilder
  .spendingPlutusScriptV3()
  .txIn(lockedUtxo.input.txHash, lockedUtxo.input.outputIndex)
  .txInScript(scriptCbor)
  .txInInlineDatumPresent()
  .txInRedeemerValue(redeemer, "JSON")
  // Continuing output: same script address, updated datum
  .txOut(scriptAddress, [
    { unit: "lovelace", quantity: lockAmount },
  ])
  .txOutInlineDatumValue(newDatum, "JSON")
  .txInCollateral(collateralTxHash, collateralTxIndex)
  .requiredSignerHash(keyHashHex)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  // CRITICAL: selectUtxosFrom required for fee coverage
  .selectUtxosFrom(walletUtxos)
  .complete();
```

**Why `selectUtxosFrom(walletUtxos)` is needed:** When all the ADA from the
script input goes to the continuing output, there's nothing left for fees.
The tx builder needs wallet UTxOs to cover fees. Without this, you get
"Insufficient lovelace" or the tx builder silently fails.

### Withdrawal Transaction (Withdraw-Zero Pattern)

The withdraw-zero pattern uses a withdrawal validator as a transaction-level
check. Register the staking credential, then include a zero-withdrawal in
transactions that need the check.

```typescript
import { WithdrawalBlueprint } from "@meshsdk/core";

// Setup: derive script hash and reward address
const withdrawBlueprint = new WithdrawalBlueprint("V3", networkId);
withdrawBlueprint.noParamScript(compiledCode);

const withdrawScriptHash = withdrawBlueprint.hash;
const withdrawScriptCbor = withdrawBlueprint.cbor;
const rewardAddress = withdrawBlueprint.address;  // bech32 stake_test1...
```

**Registration is permissionless in Conway:** No script witness, redeemer, or
collateral needed:

```typescript
await txBuilder
  .registerStakeCertificate(rewardAddress)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  .selectUtxosFrom(utxos)
  .complete();
```

**Including the withdraw-zero in a transaction:**

```typescript
await txBuilder
  // Normal spend from the guarded script
  .spendingPlutusScriptV3()
  .txIn(lockedUtxo.input.txHash, lockedUtxo.input.outputIndex)
  .txInScript(spendScriptCbor)
  .txInInlineDatumPresent()
  .txInRedeemerValue(spendRedeemer, "JSON")
  // Withdraw-zero: triggers the withdrawal validator
  .withdrawalPlutusScriptV3()
  .withdrawal(rewardAddress, "0")
  .withdrawalScript(withdrawScriptCbor)
  .withdrawalRedeemerValue(withdrawRedeemer, "JSON")
  .txInCollateral(collateralTxHash, collateralTxIndex)
  .requiredSignerHash(keyHashHex)
  .changeAddress(walletAddress)
  .signingKey(signingKeyHex)
  .selectUtxosFrom(walletUtxos)
  .complete();
```

**Deregistration limitation:** Aiken auto-generates a default `else` handler
that fails for all unmatched purposes. If the script only has a `withdraw`
handler, the `else` handler blocks deregistration (which is a `publish`
action). Production fix: add an explicit `else` handler or leave the
credential registered.

### Signing and Submission

**Critical:** `complete()` only builds the transaction. You must call
`completeSigning()` separately to add vkey witnesses:

```typescript
await txBuilder
  .txIn(...)
  // ... build chain ...
  .signingKey(signingKeyHex)
  .complete();

// THIS IS REQUIRED — complete() does NOT sign
txBuilder.completeSigning();

const signedTx = txBuilder.txHex;
const txHash = await ogmios.submitTx(signedTx);
```

Without `completeSigning()`, submission fails with "Some signatures are
missing" from Ogmios.

## Collateral

Plutus script transactions require collateral — ADA locked as insurance
against script execution failure. The ledger collects it only if the script
fails.

### Rules

1. **Collateral must be pure ADA** — UTxOs containing native tokens are
   rejected with "unsuitableCollateralValue"
2. **Use a separate UTxO** when the main input carries tokens
3. **The same UTxO can serve as both input and collateral** if it contains
   only ADA

### Finding Pure-ADA Collateral

```typescript
const utxos = await kupo.fetchAddressUTxOs(walletAddress);

// Find a UTxO with only lovelace (no native tokens)
const collateral = utxos.find((u) =>
  u.output.amount.length === 1 && u.output.amount[0].unit === "lovelace"
);

if (!collateral) {
  throw new Error("No pure-ADA UTxO available for collateral");
}

txBuilder.txInCollateral(collateral.input.txHash, collateral.input.outputIndex);
```

## Token Name Derivation (One-Shot NFTs)

Aiken's `cbor.serialise(OutputReference) |> crypto.blake2b_256` produces a
unique 32-byte token name. The off-chain code must produce the exact same
bytes.

### OutputReference CBOR Encoding

```
D879 = tag 121 (ConStr0)
9F   = indefinite-length array start
5820 = 32-byte bytestring prefix
<32 bytes tx hash>
<CBOR integer for output index>
FF   = break (end array)
```

### CBOR Integer Encoding

| Range     | Encoding                              |
|-----------|---------------------------------------|
| 0–23      | Single byte (value itself)            |
| 24–255    | `0x18` + 1 byte                       |
| 256–65535 | `0x19` + 2 bytes (big-endian)         |

### TypeScript Implementation

```typescript
import blake from "blakejs";

function deriveTokenName(txHashHex: string, outputIndex: number): string {
  const parts: Buffer[] = [];
  parts.push(Buffer.from("d879", "hex")); // ConStr0 tag
  parts.push(Buffer.from("9f", "hex"));   // indefinite array
  parts.push(Buffer.from("5820", "hex")); // 32-byte bytestring prefix
  parts.push(Buffer.from(txHashHex, "hex")); // tx hash
  // CBOR integer encoding
  if (outputIndex <= 23) {
    parts.push(Buffer.from([outputIndex]));
  } else if (outputIndex <= 255) {
    parts.push(Buffer.from([0x18, outputIndex]));
  } else {
    parts.push(Buffer.from([0x19, (outputIndex >> 8) & 0xff, outputIndex & 0xff]));
  }
  parts.push(Buffer.from("ff", "hex")); // break
  const cbor = Buffer.concat(parts);
  return Buffer.from(blake.blake2b(cbor, null, 32)).toString("hex");
}
```

**Note:** Node.js 20 lacks native blake2b256 (`crypto.createHash('blake2b256')`
throws "Digest method not supported"). Use the `blakejs` npm package.

## Integration Testing on Preview Testnet

### Prerequisites

1. **Cardano preview node** with Ogmios + Kupo running
2. **Funded preview wallet** — get tADA from the Cardano testnet faucet
3. **SSH tunnels** if providers are on a remote host:
   ```bash
   ssh -N -L 1337:localhost:1337 -L 1442:localhost:1442 cardano@your-node
   ```

### Test Harness Structure

```
my-contract/
├── contract/
│   └── plutus.json      # Compiled Aiken blueprint
├── test/
│   ├── config.ts        # Provider config, key loading, blueprint parsing
│   ├── integration.ts   # End-to-end tests on preview
│   └── keys/            # Signing keys (gitignored)
├── tsconfig.json
└── package.json
```

### Minimal package.json

```json
{
  "type": "module",
  "scripts": {
    "test": "tsx test/integration.ts",
    "test:verify": "tsx test/integration.ts verify",
    "test:burn": "tsx test/integration.ts burn"
  },
  "dependencies": {
    "@meshsdk/core": "^1.9.0-beta.101",
    "@meshsdk/core-cst": "^1.9.0-beta.101",
    "blakejs": "^1.2.1",
    "ws": "^8.0.0"
  },
  "devDependencies": {
    "tsx": "^4.0.0",
    "typescript": "^5.0.0"
  }
}
```

### Test Flow

A complete integration test exercises the full lifecycle:

1. **Setup** — connect providers, load wallet, parameterize script
2. **Deploy** — mint token with inline datum, submit to chain
3. **Verify** — query the minted UTxO, read datum back
4. **Burn** — revoke by burning the token

Save state (tx hash, token name, policy ID) to a JSON file between steps so
verify and burn can run independently.

### Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `WebSocket is not defined` | Node.js 20 lacks WebSocket | `import WebSocket from "ws"; (globalThis as any).WebSocket = WebSocket` |
| `Cannot convert undefined to a BigInt` | Mesh type datum/redeemer format | Use `"JSON"` format for all Plutus data |
| `Malformed Plutus data json` | `alternative` key in JSON format | Use `constructor` key, not `alternative` |
| `Some signatures are missing` | `completeSigning()` not called | Call `txBuilder.completeSigning()` after `complete()` |
| `unsuitableCollateralValue` | Collateral UTxO has native tokens | Use a pure-ADA UTxO for `.txInCollateral()` |
| `No UTxOs found` | Wrong address type | Use `getEnterpriseAddress()` for enterprise wallets |
| `Digest method not supported` | Node.js 20 no blake2b256 | Use `blakejs` npm package |
| `ConflictingOptionsException` | Changed Kupo `--match` on existing DB | Use HTTP API `PUT /v1/patterns/` or nuke DB and restart |
| UTxO not found after submit | Kupo hasn't indexed the new block yet | Wait ~30s for next block, or query Kupo API directly to verify |
| `unknown UTxO references` (3117) | UTxO consumed by another tx in same block | Wait for block confirmation (~30s), retry with fresh UTxO set |
| `submitted too early` | `invalidBefore` slot ahead of ledger tip | Use `currentSlot - 60` safety margin for clock skew |
| `Insufficient lovelace` on continuing output | No ADA left for fees after script output | Add `.selectUtxosFrom(walletUtxos)` for fee coverage |

## Kupo Runtime Pattern Management

When deploying new contracts, register their script credentials with Kupo via
the HTTP API (no restart needed):

```bash
# Get a recent slot for rollback_to (must be within 36h safe zone)
SLOT=$(curl -s http://localhost:1442/v1/checkpoints \
  | python3 -c "import json,sys; print(json.load(sys.stdin)[-1]['slot_no'])")

# Register the script credential
curl -X PUT "http://localhost:1442/v1/patterns/${SCRIPT_HASH}/*" \
  -H "Content-Type: application/json" \
  -d "{\"rollback_to\":{\"slot_no\":${SLOT}}}"
```

**Gotchas:**
- `slot_no: 0` is beyond the safe zone — use a recent checkpoint slot
- Kupo may return 503 "too busy" during re-indexing — wait 60s and retry
- Multiple rapid pattern registrations can overwhelm Kupo — space them out
- Verify registration: `GET /v1/patterns` should include your credential

## Complex Datum Encoding Patterns

### List of Pairs (Aiken `List<Pair<K, V>>`)

Aiken's `List<Pair<ByteArray, Int>>` compiles to a Plutus map type. Use
the map encoding in JSON format:

```typescript
// Aiken: beneficiaries: List<Pair<ByteArray, Int>>
// JSON encoding:
{
  map: [
    { k: { bytes: "aabb..." }, v: { int: 5000 } },
    { k: { bytes: "ccdd..." }, v: { int: 3000 } },
  ]
}
```

### Nested List (Aiken `List<ByteArray>`)

```typescript
// Aiken: signers: List<ByteArray>
{ list: [{ bytes: "aabb..." }, { bytes: "ccdd..." }] }

// Aiken: allowed_pools: List<ByteArray>
{ list: [{ bytes: poolHash1 }, { bytes: poolHash2 }] }
```

### Bool Encoding

```typescript
// Aiken True = ConStr1, False = ConStr0
{ constructor: 1, fields: [] }  // True
{ constructor: 0, fields: [] }  // False
```

## E2E Test Patterns (Verified on Preview Testnet)

The following patterns have been validated end-to-end across 16 contracts
with 45 on-chain operations on preview testnet.

### Contract Pattern Coverage

| Pattern | Contract | Operations | Key Off-Chain Technique |
|---------|----------|-----------|------------------------|
| Simple mint+burn | Notary | notarize, verify, burn | MintingBlueprint, one-shot minting |
| Time-locked spend | Vesting | lock, withdraw, claim | invalidBefore/invalidHereafter slots |
| One-shot mint+spend | Gift Card | create, check, redeem | Combined mint+spend tx |
| Multi-redeemer spend | Escrow | lock, complete, cancel, refund | Different redeemer constructors |
| Multi-sig spend | Multi-Sig | lock, approve | List datum encoding for signers |
| Continuing output | State Machine | lock, increment, reset | selectUtxosFrom for fees |
| Parameterized multi-handler | NFT Vault | create, check, close | MintingBlueprint + SpendingBlueprint same params |
| Withdraw-zero | Withdraw Zero | register, lock, batch-spend | WithdrawalBlueprint, permissionless registration |
| Time + continuing output | Dead Man's Switch | lock, withdraw, checkin | invalidBefore safety margin, datum update |
| Output payment check | Marketplace | list, buy, cancel | Explicit txOut to seller address |
| Pair encoding + time | Multi-Beneficiary | lock, withdraw, claim | Map encoding, percentage calculation |
| Message redeemer | Hello World | lock, unlock | ByteArray redeemer encoding |
| Tx-level mint validation | TVMP | mint-receipt | Non-parameterized MintingBlueprint |
| Index-based I/O linking | UTxO Indexer | lock, update | Sorted input index in redeemer |
| Validity range normalisation | Validity Range | lock, spend-after, lock-again, spend-before | invalidBefore/invalidHereafter with safety margin |
| Pool whitelist enforcement | Pool Restriction | lock, unlock | List datum, certificate checking |

### UTxO Contention

When running multiple tests against the same wallet, UTxOs consumed by one
transaction may not be confirmed when the next transaction tries to use them.
This causes "unknown UTxO references" (error code 3117).

**Prevention:** Run tests sequentially with ~30s between operations that
depend on previous tx confirmation. Never submit two transactions from the
same wallet in the same block unless they use completely disjoint UTxO sets.
