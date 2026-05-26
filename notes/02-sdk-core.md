# 02 — The SDK core (`ethereum/kohaku`)

A hybrid **Rust (Cargo workspace) + TypeScript (pnpm workspace)** monorepo.
Rust does the cryptography and proving and compiles to WASM; TypeScript wraps
the WASM and supplies all I/O (storage, RPC, fetch) back across the boundary.

```
crates/                         packages/
  common          (utils)         plugins        (@kohaku-eth/plugins — the Host/plugin contract)
  crypto          (primitives)    provider       (@kohaku-eth/provider — multi-backend RPC)
  poseidon-rust   (hash)          privacy-pools  (@kohaku-eth/privacy-pools — pure TS)
  eip-1193-provider               pq-account     (post-quantum 4337 — see note 03)
  railgun         (protocol)
  railgun-ts      (WASM bindings → @kohaku-eth/railgun)
  userop-kit      (ERC-4337)
  userop-kit-ts   (WASM bindings → @kohaku-eth/userop-kit-ts)
```

Dependency graph (TS side):

```
plugins ──► provider
privacy-pools ──► plugins, provider   (+ reduxjs, maci-crypto, @zk-kit/lean-imt, circuits)
railgun (@kohaku-eth/railgun) ──► plugins, provider   (+ railgun-ts WASM)
```

---

## 1. `@kohaku-eth/plugins` — the contract

Already covered in note 01. Key files:

- `src/host/index.ts` — `Host`, `Network`, `Storage`, `Keystore` types (verified verbatim).
- `src/base.ts` — the type-level capability machinery. `PluginInstance<TAccountId, C>`
  is `instanceId()` + a `Transact` object whose methods are **`Pick`ed** from
  `TxFeatureMap` by the protocol's declared `features` flags. This means a plugin
  that doesn't declare `prepareTransfer` literally won't have that method at the
  type level — capabilities are enforced by the compiler.
- `src/shared.ts` — `AssetId` is a discriminated union: `{__type:'native'}` |
  `{__type:'erc20', contract}` | `{__type:'erc721', contract, tokenId}`.
  `AssetAmount` pairs an asset with an amount. `PrivateOperation` vs
  `PublicOperation` are the two operation kinds.
- `src/host/mnemonic-keystore.ts` — reference `Keystore` from a mnemonic (scure bip32/bip39).
- `src/host/memory-storage.ts` — in-memory `Storage` (tests only; not encrypted).
- `src/broadcaster/base.ts` — `Broadcaster<TPrivateOp, TResult>` = `{ broadcast(op) }`,
  the thing that actually submits a private operation.
- `examples/railgun.ts`, `examples/tornado.ts` — show full vs. minimal capability sets.

`AssetAmounts` has four flavors (`input`/`internal`/`output`/`read`) so the
types can distinguish "what you put in to shield" from "what's valid inside the
pool" from "what you get out" from "what a balance read returns" (which can be
tagged, e.g. `pending`).

---

## 2. `@kohaku-eth/provider` — one RPC interface, many backends

`src/provider.ts` defines `EthereumProvider` (getChainId/BlockNumber/Balance/Code,
`getLogs`, `getTransactionReceipt`, `call`, `estimateGas`, `request`) and a
`TxSigner`. Backends, each behind the same interface:

- `src/ethers/` — wraps ethers v6 `JsonRpcProvider`.
- `src/viem/` — wraps a viem `PublicClient` (already EIP-1193).
- `src/raw/` — adapts any `{ request(method, params) }` EIP-1193 object; includes a
  `waitForTransaction` poll loop and hex→bigint normalization.
- `src/helios/` — **a16z Helios** WASM light client (`@a16z/helios`). Falls back to a
  raw RPC for `eth_getLogs` (Helios has a small block-range window); `bypassLogs`
  routes logs straight to RPC.
- `src/colibri/` — **CORPUS `@corpus-core/colibri-stateless`** WASM stateless light
  client (dynamic import). Supports a `zk_proof` mode and a `fallback_provider`.

The light-client backends are the privacy story for *reads*: you can fetch chain
state without trusting (or doxxing yourself to) a single RPC. ambire-common wraps
this further in a `ColibriRpcProvider` (note 04).

---

## 3. Railgun — the Rust protocol core (`crates/railgun`)

Railgun is a **UTXO-style shielded pool**. The Rust crate is the heart of the SDK.

### Note / UTXO model (`src/note/utxo.rs`)

A shielded note (`UtxoNote`) carries `tree_number`, `leaf_index`, spending &
viewing pubkeys, 16 bytes of `random`, `value`, `asset`, `memo`, plus derived:

- **commitment hash** (the Merkle leaf): `Poseidon(NPK, asset_hash, value)`
- **note public key (NPK)**: `Poseidon(MasterPublicKey(spending, nullifying), random)`
- **nullifier**: `Poseidon(nullifying_key, leaf_index)` — revealed when spending, prevents double-spend, unlinkable to the commitment
- **blinded commitment**: extra privacy layer used by POI

Notes are encrypted to the recipient's **viewing key** (AES-GCM for shields;
ECDH-derived shared key for transacts), so only the holder can decrypt value/random.

### State sync (`src/indexer/`)

- `utxo_indexer.rs` maintains in-memory Merkle trees + per-account balances. Sync
  = fetch events → route to handlers (Shield/Transact/Nullified/Legacy) → insert
  leaves → rebuild trees → verify root against on-chain → persist snapshot to the
  `Host` storage. ("Tabular state" in the TODO refers to refactoring this state
  into table form for persistence.)
- `txid_indexer.rs` maintains a parallel Merkle tree of railgun transaction IDs,
  needed for POI.
- Two syncers (`syncer/`): **Subsquid** (GraphQL, batches ~20k events — fast) and
  **RPC** (scans logs via the EIP-1193 callback — trustless fallback). They can be
  **chained** (`UtxoSyncer.chained([subsquid, rpc])`).

### Merkle tree (`src/merkle_tree/`)

Depth-16 (65 536 leaves/tree), **Poseidon** as the hash, zero-leaf =
`keccak256("Railgun") mod SNARK_PRIME`. Supports batched `insert_leaf_raw` then
`rebuild`, and `generate_proof` (sibling path) for circuit inputs.

### Spending: build → prove → broadcast

- `transact/shield_builder.rs` — builds the *public* shield txn (no proof): encrypt
  note to recipient viewing key, call `RailgunSmartWallet.shield(...)`. Native ETH is
  wrapped to WETH via a RelayAdapt contract.
- `transact/transaction_builder.rs` — builds *private* transfer/unshield ops. Groups
  intents by `(signer, asset)`, selects input notes from one tree, creates output +
  change notes.
- `circuit/groth16_prover.rs` — picks circuit `railgun/{nIn:02}x{nOut:02}` (e.g.
  `02x02`), loads proving key + matrices + circuit WASM, computes the witness
  (`circuit/witness.rs` via **ark-circom**'s WASM witness calculator), runs
  `Groth16::create_proof…`, and verifies locally before returning.
- `circuit/remote_artifact_loader.rs` — fetches `.wasm`/`_proving_key.bin`/`_matrices.bin`
  on demand from a GitHub repo (`Robert-MacWha/privacy-protocol-artifacts`). So
  circuit artifacts are **downloaded at runtime**, not bundled.
- `circuit/inputs/transact_inputs.rs` — maps protocol state → circuit signals.
  Public: `merkleRoot`, `boundParamsHash`, `nullifiers`, `commitmentsOut`. Private:
  token, pubkey, BabyJubJub `signature`, input values, Merkle `pathElements`,
  nullifying key, output NPKs/values.

### POI — Proof of Innocence (`src/poi/`)

A compliance layer: a note's `PoiStatus` is `Valid` / `ShieldBlocked` /
`ProofSubmitted` / `Missing`. Spending with POI enabled adds extra circuit inputs
(`circuit/inputs/poi_inputs.rs`): a TXID-tree membership proof plus per-list
membership proofs that the note descends from "innocent" (non-blocklisted)
shields. `withPoi()` on the builder turns this on. (See the TODO: they want to
switch the `spendable` check to read POI status rather than mere tree inclusion.)

### Keys & address (`src/account/`)

`RailgunSigner` exposes a viewing key (decrypt / ECDH) and a spending key
(BabyJubJub signatures over the transact challenge). Addresses are `0zk1…`
bech32-style, derived from spending+viewing pubkeys + chain id.

---

## 4. Cryptographic primitives (`crates/crypto`, `crates/poseidon-rust`)

- **Poseidon** (`poseidon-rust/src/lib.rs`): ZK-friendly hash over BN254 Fr, params
  for arity t=2..17. Used for Merkle hashing, commitments, nullifiers, and signature
  challenges. `crypto/src/poseidon.rs` wraps it for U256 ↔ Fr.
- **BabyJubJub** (`crypto/src/babyjubjub.rs`): twisted Edwards curve for in-circuit
  signatures. EdDSA-style: `R = r·B8`, challenge `hm = Poseidon(R.x,R.y,pk.x,pk.y,msg)`,
  `s = r + hm·scalar`. Output `(r8_x, r8_y, s)` — three field elements the circuit checks.
- Also MiMC, Pedersen, Merkle helpers.

Everything is BN254-based so it slots into Groth16 circuits cheaply.

---

## 5. ERC-4337 (`crates/userop-kit`) + the Railgun unshield path

`user_operation.rs` is a full ERC-4337 `UserOperation` (EntryPoint 0.7 & 0.8):
sender, nonce, factory, callData, the gas triple, paymaster fields, an EIP-7702
`authorization`, signature. A `Bundler` trait (Pimlico/alto impl) does fee
suggestion, gas estimation, send, and receipt polling.

**The clever bit — how Railgun pays for an unshield privately** (`railgun.rs` +
`crates/railgun/src/provider.rs::prepare_userop`):

- The spender doesn't pay gas from a doxxing EOA. Instead an EOA signs an **EIP-7702
  authorization delegating to a fixed "Privacy Account" implementation**
  (`IMPL = 0x304a…4b4c`). The UserOp's calldata is `executeCall(fee_calldata, tail_calls)`.
- A **privacy paymaster** (`0xBbbc…bB74`) sponsors gas and is repaid by a Railgun
  **fee note**: the transaction includes an internal transfer of a fee asset (wrapped
  base token) to the paymaster's Railgun address, and `paymaster_data` carries the
  fee note's `(random, asset, value)` commitment so the paymaster can verify it.
- `prepare_userop` runs a **fee-convergence loop** (≤5 iterations): build op with a
  guessed fee → ask the bundler to estimate gas → recompute fee = `gas·maxFee·1.1` →
  repeat until within ~1%. Then it returns a `SignableUserOperation`.
- `tail_calls` let you append extra actions (e.g. unwrap WETH→ETH for the recipient).

This is why the CLI's Railgun unshield needs the recipient's private key as a
"delegating signer" and a Pimlico bundler URL (note 05).

---

## 6. The Rust ↔ TypeScript WASM bridge

This is what makes the whole thing usable from JS, and it's bidirectional.

**Rust → TS (exporting types/functions):** `wasm-bindgen` exposes Rust structs as JS
classes (e.g. `JsRailgunSigner` → `RailgunSigner`, `JsRailgunBuilder` →
`RailgunBuilder`), and **`tsify`** auto-generates the `.d.ts` types. `u128`/U256
become `BigInt`; `serde-wasm-bindgen` serializes structs across the boundary.
Built with `wasm-pack build --target web` → `pkg/index_bg.wasm`, loaded lazily by
`crates/railgun-ts/sdk/lib.ts`.

**TS → Rust (callbacks):** this is the key inversion — Rust calls *back* into JS for
all I/O. Two `extern "C"` interfaces are declared in Rust with
`#[wasm_bindgen(typescript_custom_section)]`:

- `Database` (`crates/railgun-ts/src/database.rs`) — `get/set/delete` mapped to the
  Host's `storage`. TS side: `crates/railgun-ts/sdk/database.ts` prefixes keys with
  `chainId:` and proxies to `host.storage`.
- `Eip1193Provider` (`crates/eip-1193-provider/src/js.rs`) — `getChainId`, `getLogs`,
  `ethCall`, `estimateGas`, etc. TS side: `crates/railgun-ts/sdk/ethereum-provider.ts`
  adapts the Host's `EthereumProvider` into it.

Async crosses the boundary via `wasm-bindgen-futures` (Rust `async` extern fn ↔ JS
`Promise`). A `build.rs` sets `cfg_aliases` (`native`/`wasm`/`js`) so the same crate
compiles natively (alloy HTTP client, real tokio) or to WASM (JS callbacks,
single-threaded). The `?Send` async-trait variant is used because WASM is
single-threaded.

So: **Rust owns the Merkle trees, note crypto, and Groth16 proving; JS owns storage,
RPC, and fetch; they meet through `Database` + `Eip1193Provider` callbacks that the
`Host` ultimately backs.**

### The TS Railgun SDK (`crates/railgun-ts/sdk/`)

`plugin.ts::createRailgunPlugin(host, config)` is the unified entry point used by
the CLI:
1. derive spending+viewing keys via `host.keystore.deriveAt(RailgunSigner.…Path(i))`
2. wrap `host.provider` → `Eip1193Provider`, `host.storage` → `Database`
3. `new RailgunBuilder(chain, provider).withDatabase(db).withUtxoSyncer(chained([subsquid, rpc])).withPoi().build()`
4. register the signer; expose `balance/prepareShield/prepareTransfer/prepareUnshield/broadcast`

`signer-pool.ts::drain()` greedily selects UTXOs across one or more signers to
cover a requested amount. `broadcast()` converts the proved builder into a 4337
UserOp and sends it via the configured bundler.

---

## 7. Privacy Pools — pure TypeScript (`packages/privacy-pools`)

Unlike Railgun, Privacy Pools is implemented **entirely in TS** (no Rust/WASM),
leaning on `maci-crypto` (Poseidon), `@zk-kit/lean-imt` (incremental Merkle tree),
and a circuits package for proving.

- `src/plugin/base.ts::PrivacyPoolsV1Protocol` implements the plugin interface.
  Construction wires a `SecretManager`, a Redux-backed `StateManager`, a
  `RelayerClient`, and a `DataService`.
- **Secrets** (`src/account/keys.ts`): custom BIP-32 path
  `m/28784'/1'/{account}'/{type}'/{deposit}'/{index}'` where type 0 = nullifier,
  1 = salt; secrets are bound to `chainId + entrypoint` via Poseidon. A note's
  `precommitment = Poseidon(nullifier, salt)`.
- **State** (`src/state/state-manager.ts`): a Redux store keyed
  `privacy-pool-state-{chainId}-{entrypoint}`, persisted into `host.storage` after
  every sync. Thunks: `sync`, `withdraw`, `ragequit`, `quote`.
- **API**: `prepareShield` → a public deposit txn (ERC-20 approve or ETH transfer to
  the Entrypoint). `prepareUnshield` → syncs, gets a relayer quote, greedily selects
  notes, generates the withdrawal proof, returns a `PrivateOperation` with
  `{context, relayData, proof, withdrawalPayload}` + the relayer quote. `ragequit`
  → emergency self-withdrawal bypassing relayers (for un-approved notes). `notes()`
  → list commitments.
- **ASP** (Association Set Provider) services: `IPFSAspService`, `OxBowAspService` —
  fetch the approved/blocked membership sets used to prove your funds are "clean".
- `src/plugin/broadcaster.ts` POSTs the proof to a relayer (`/quote`, `/request`).

Config (`src/config.ts`) ships mainnet + Sepolia entrypoint addresses and deploy
blocks.

---

## 8. Docs (`docs/`)

A Vocs site. `getting-started.mdx` frames the wallet-integration model; `privacy.mdx`
flags the real-world privacy pitfalls (address linkability, RPC centralization,
token discovery). The `railgun/` pages explain shield/unshield/accounts/txs/ppoi;
`privacy-pools/` and `tornado/` are WIP. Worth reading `privacy.mdx` — it's the
project's honest threat-model statement.

---

## Key files index

| Concern | File |
|---|---|
| Host/plugin contract | `packages/plugins/src/host/index.ts`, `src/base.ts`, `src/shared.ts` |
| Provider backends | `packages/provider/src/{ethers,viem,raw,helios,colibri}/` |
| Railgun note model | `crates/railgun/src/note/utxo.rs` |
| Railgun sync | `crates/railgun/src/indexer/{utxo_indexer,txid_indexer,syncer/*}.rs` |
| Railgun Merkle | `crates/railgun/src/merkle_tree/*.rs` |
| Railgun proving | `crates/railgun/src/circuit/{groth16_prover,witness,remote_artifact_loader,inputs/*}.rs` |
| Railgun POI | `crates/railgun/src/poi/*.rs` |
| 4337 + 7702 | `crates/userop-kit/src/{user_operation,railgun}.rs`, `crates/railgun/src/provider.rs` |
| Crypto | `crates/crypto/src/{poseidon,babyjubjub}.rs`, `crates/poseidon-rust/src/lib.rs` |
| WASM bridge | `crates/railgun-ts/src/*.rs`, `crates/eip-1193-provider/src/js.rs`, `crates/railgun-ts/sdk/*.ts` |
| Privacy Pools | `packages/privacy-pools/src/{plugin,state,account,relayer}/*.ts` |
