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
