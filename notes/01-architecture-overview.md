# 01 — Architecture overview

## What Kohaku is, in one paragraph

Kohaku is the Ethereum Foundation's effort to make **on-chain privacy a
first-class wallet feature**. It is not one app — it's a layered stack:

1. A **privacy SDK** (`ethereum/kohaku`) that implements two privacy protocols
   (Railgun, Privacy Pools), a multi-backend RPC provider, a post-quantum
   account, and — crucially — a **plugin abstraction** that decouples protocol
   logic from any particular wallet.
2. A **reference wallet** built by forking Ambire: the browser extension
   (`kohaku-extension`) + its core logic library (`kohaku-commons`, which *is*
   `ambire-common`).
3. A **community CLI** (`kohaku-cli`) that consumes the published SDK.

## The seam: `Host` + plugins

The whole design pivots on one interface in `@kohaku-eth/plugins`
(`kohaku/packages/plugins/src/host/index.ts`):

```ts
type Host = {
  network: { fetch(input, init?): Promise<Response> };          // HTTP (ASP, relayers, indexers)
  storage: { get(key): string | null; set(key, value): void };  // persistent KV; SHOULD be encrypted
  keystore: { deriveAt(path: string): Hex };                     // BIP-32 derivation, deterministic
  provider: EthereumProvider;                                    // RPC access
};
```

A **plugin** is created from a Host via `CreatePluginFn = (host, params) => PluginInstance`
(`packages/plugins/src/base.ts:74`). The instance exposes a uniform API whose
*available methods are gated at the type level* by the protocol's declared
capabilities (`TxFeatureMap` in `base.ts:18`):

- `balance(assets?)` — always present
- `prepareShield` / `prepareShieldMulti` — **public** ops (user signs & sends a normal tx)
- `prepareTransfer` / `prepareTransferMulti` — **private** ops (proved, relayed)
- `prepareUnshield` / `prepareUnshieldMulti` — **private** ops

`prepareShield` returns a `PublicOperation` (raw `{to,data,value}` txns the user
broadcasts directly); `prepareTransfer`/`prepareUnshield` return a
`PrivateOperation` (a proof + payload handed to a relayer/bundler/broadcaster,
not signed as an ordinary tx). That public-vs-private split is the core mental
model — see note 06.

### Why this matters

Because protocol logic only ever touches the `Host`, the *same* Railgun and
Privacy Pools code runs in three very different environments. Each environment
is just a different `Host` implementation:

| Host implementation | storage | keystore | provider | network |
|---|---|---|---|---|
| **kohaku-cli** | AES-256-GCM JSON files on disk (`src/host/storage.ts`) | `derive-railgun-keys` (RG) or BIP-44 (PP) (`src/host/keystore.ts`) | ethers wrapped w/ chunked `eth_getLogs` (`src/host/chunked-get-logs.ts`) | Node global `fetch` |
| **kohaku-commons** (Privacy Pools) | Ambire's plugin storage | Ambire keystore → `HDNodeWallet.derivePath` (`hostFactory.ts`) | viem provider | Ambire fetch |
| **browser extension** | extension storage via background | derived keys from background service | colibri light client + Alchemy/HyperSync | browser fetch |

This is the single most important architectural fact about the project.

## Two technology lineages meet here

Kohaku is the **graft of two independent codebases**:

- **The privacy engine is new EF code.** `ethereum/kohaku` is a hybrid Rust +
  TypeScript monorepo. The heavy crypto (Railgun notes, Merkle trees, Groth16
  proving) is Rust compiled to WASM; TypeScript wraps it and supplies I/O.
- **The wallet shell is Ambire.** `kohaku-extension` is a fork of Ambire Wallet,
  and `kohaku-commons` is a fork of `ambire-common` (the package is still
  *named* `ambire-common`, v2.68). Ambire brings a mature, audited smart-account
  wallet (controllers, keystore, ERC-4337/7702, humanizer, recovery). Kohaku
  bolts privacy controllers onto it.

So when reading the extension/commons, constantly ask "is this inherited Ambire
machinery or a Kohaku privacy addition?" Note 04 separates them.

## End-to-end data flow (a shield, then an unshield)

**Shield (public → private)** — e.g. CLI `shield --protocol railgun`:

```
user/UI
  → plugin.prepareShield(asset)        (Railgun WASM builds a ShieldRequest, encrypts note for recipient viewing key)
  → returns PublicOperation { txns: [approve?, shield] }
  → user signs & broadcasts normally (ordinary EOA/account tx)
  → on-chain RailgunSmartWallet.shield() emits a Shield event
  → next plugin.balance() syncs the event, decrypts the note → it's now spendable
```

**Unshield (private → public)** — e.g. CLI `unshield --protocol railgun`:

```
user/UI
  → plugin.prepareUnshield(asset, recipient)
      • sync UTXO state (Subsquid + RPC)
      • select input notes (drain), build output notes
      • generate Groth16 proof (circom witness in WASM)
      • package as a private operation
  → returns PrivateOperation
  → plugin.broadcast(op)
      • Railgun: submit as an ERC-4337 UserOp via a Pimlico bundler,
        using an EIP-7702 delegated "Privacy Account", paid by a privacy paymaster
      • Privacy Pools: POST proof to a relayer (fastrelay.xyz) that submits on-chain
  → recipient EOA receives funds; nullifiers mark inputs spent
```

Note the asymmetry: **shields are cheap public txns the user signs; spends
(transfer/unshield) are ZK-proved and relayed** so the spender's address never
appears on-chain. Note 02 has the Railgun mechanics; note 06 the concepts.

## Repo-by-repo, where the depth lives

- **Privacy mechanics / crypto / proving** → `kohaku/crates/railgun`, `crates/crypto`, `crates/poseidon-rust` (note 02)
- **Plugin & provider abstractions** → `kohaku/packages/plugins`, `packages/provider` (note 02)
- **Privacy Pools protocol (pure TS)** → `kohaku/packages/privacy-pools` (note 02)
- **Post-quantum account** → `kohaku/packages/pq-account` (note 03)
- **Wallet UX, dapp connection, account model** → `kohaku-extension` + `kohaku-commons` (note 04)
- **Scriptable/agent wallet** → `kohaku-cli` (note 05)

## Status snapshot (May 2026)

Everything is **alpha** — and **not ready for mainnet**: a real-money round-trip
proved you can shield on mainnet but can't unshield (see
[07-mainnet-demo-and-unshield-blocker.md](07-mainnet-demo-and-unshield-blocker.md)).
Headline SDK packages are `0.0.x-alpha.*`. The extension
is **Sepolia-testnet only**, with some flows stubbed (notably Railgun *withdrawal*
in the extension UI — `useRailgunForm.ts:~426`). The CLI is a single-commit v0.0.1.
The pq-account contracts are deployed on Sepolia + Arbitrum Sepolia with a full
Foundry test suite. Tornado Cash support exists as a plugin example/docs stub only.

A telling version skew: the **CLI tracks newer SDK builds** (`@kohaku-eth/railgun`
`0.0.1-alpha.21`) than the **extension** (`0.0.1-alpha.8`), and uses the unified
`createRailgunPlugin(host, …)` entry point, whereas the extension drives Railgun
through an older `createRailgunAccount`/`Indexer` surface. The plugin/Host model
is the direction of travel; the extension hasn't fully migrated onto it for
Railgun yet. (See notes 04 and 05.)
