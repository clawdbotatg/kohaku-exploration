# Kohaku — What the repos are (TLDR)

Kohaku is the Ethereum Foundation's **privacy-first wallet tooling** effort: a stack of
libraries plus a reference wallet that bake on-chain privacy (Railgun, Privacy Pools)
and post-quantum account abstraction directly into Ethereum wallets.

There are two layers here:
- The **SDK / brains** (`ethereum/kohaku`) — protocol logic, mostly Rust compiled to WASM + TS.
- The **product / UI** (`ethereum/kohaku-extension` + `ethereum/kohaku-commons`) — a browser
  wallet, built by forking **Ambire Wallet** and wiring the SDK into it.

Plus one community tool (`kassandraoftroy/kohaku-cli`) that consumes the published SDK.

---

## The four repos at a glance

| Repo | Owner | What it is | Language | Status |
|------|-------|------------|----------|--------|
| [`kohaku`](https://github.com/ethereum/kohaku) | ethereum (EF) | The privacy **SDK** — Railgun, Privacy Pools, provider abstraction, post-quantum 4337 account, plugin system | Rust→WASM + TypeScript | Alpha, active |
| [`kohaku-extension`](https://github.com/ethereum/kohaku-extension) | ethereum (EF) | The **reference browser wallet** (MetaMask-style) that consumes the SDK. Fork of Ambire Wallet | TypeScript / React Native Web | Prototype, Sepolia-only |
| [`kohaku-commons`](https://github.com/ethereum/kohaku-commons) | ethereum (EF) | Shared **wallet business logic** behind the extension. Fork of `ambire-common` (still named that) | TypeScript (+ Solidity) | Production-grade (Ambire base) |
| [`kohaku-cli`](https://github.com/kassandraoftroy/kohaku-cli) | kassandraoftroy (community) | A **terminal wallet** for shield/unshield, built on the published SDK | TypeScript (Node) | Early, single-commit |

---

## 1. `ethereum/kohaku` — the SDK (the brains)

Privacy-first tooling monorepo. **Hybrid Rust (Cargo workspace, compiled to WASM via
`wasm-bindgen`/`tsify`) + TypeScript (pnpm workspace).** This is where the actual privacy
protocols and crypto live. The headline npm packages:

- **`@kohaku-eth/railgun`** — Railgun privacy protocol SDK. Shield/unshield ERC-20s, manage
  UTXOs, generate Groth16 zk-SNARK proofs (via `ark-*`/circom), POI (Proof of Innocence),
  broadcast through 4337 bundlers. Rust core (`crates/railgun`) + WASM bindings (`crates/railgun-ts`).
- **`@kohaku-eth/privacy-pools`** — Privacy Pools v1/v2 lib. Commitments/nullifiers (Poseidon),
  ASP (compliance) integration, viem-based.
- **`@kohaku-eth/provider`** — one provider interface over **ethers, viem, helios** (a16z light
  client), and **colibri** (light client). Lets the wallet swap RPC backends, including trustless ones.
- **`@kohaku-eth/pq-account`** — **post-quantum ERC-4337 account**. Verifies a classic sig
  (ECDSA/P256) *and* a PQ sig (ML-DSA/Falcon). Solidity, deployed on Sepolia + Arbitrum Sepolia.
- **`@kohaku-eth/plugins`** — plugin interface so each privacy protocol plugs into a host that
  supplies network/storage/keystore/provider. (This is the seam the extension and CLI hook into.)

Other crates: `crypto`, `poseidon-rust`, `eip-1193-provider`, `userop-kit` (ERC-4337 UserOp
builder/bundler client), `common`. Tornado Cash support is documented but WIP.

## 2. `ethereum/kohaku-extension` — the wallet (the product)

A **Manifest V3 browser extension wallet** (Chrome/Firefox/Safari), explicitly a **fork of
Ambire Wallet**, whose goal is "to embed onchain privacy directly into ethereum wallets." Built
with Expo + Webpack + React Native Web. It depends on the SDK packages
(`@kohaku-eth/railgun`, `privacy-pools`, `provider`, `plugins`) and adds the UI: account
creation, dApp connection (injects `window.ethereum`), hardware wallets, and privacy flows
(Railgun deposits, Privacy Pools v1 deposit/transfer/ragequit). Testnet-only, several flows
still stubbed (e.g. Railgun withdrawal). Pulls `kohaku-commons` in as a submodule (`src/ambire-common`).

## 3. `ethereum/kohaku-commons` — the wallet's foundation

A **fork of `ambire-common`** (package name is literally `ambire-common`, v2.x) — the
framework-agnostic core business logic behind Ambire wallets: account/keystore management,
transaction building, ERC-4337/7702 support, portfolio, humanizer, controllers, plus the
`AmbireAccount` Solidity contracts. It is **not** a Kohaku-invented library; it's the mature,
audited Ambire base that the Kohaku extension is built on top of. Consumed by
`kohaku-extension` (not by the core `kohaku` SDK).

## 4. `kassandraoftroy/kohaku-cli` — a community terminal wallet

A **command-line wallet** by **kassandra.eth** (community, *not* the EF org) for moving funds
between public accounts and private balances on **Railgun and Privacy Pools**. It **consumes the
published `@kohaku-eth/*` npm packages** (railgun, privacy-pools, plugins, provider) — no forking
or reimplementation; it implements the plugin `Host` (network/storage/keystore/provider) and
delegates all protocol logic to the official libs. Adds: BIP-39 seed mgmt with on-disk AES
encryption, HD account derivation, balance scanning, and a `--non-interactive` JSON mode for
scripts/agents. Early stage (v0.0.1, single commit).

---

## How they relate

```
            ┌─────────────────────────────────────────────┐
            │  ethereum/kohaku  (the privacy SDK)          │
            │  railgun · privacy-pools · provider ·        │
            │  pq-account · plugins   (Rust→WASM + TS)     │
            └───────────────┬─────────────────┬───────────┘
        published @kohaku-eth/* on npm        │
                            │                 │
        ┌───────────────────┴──────┐   ┌──────┴────────────────────────┐
        │ ethereum/kohaku-extension│   │ kassandraoftroy/kohaku-cli     │
        │ reference browser wallet │   │ community terminal wallet      │
        │ (fork of Ambire Wallet)  │   │ (consumes the published libs)  │
        └───────────────┬──────────┘   └────────────────────────────────┘
                        │ submodule / dependency
            ┌───────────┴──────────────┐
            │ ethereum/kohaku-commons   │
            │ = ambire-common fork      │
            │ (core wallet logic)       │
            └───────────────────────────┘
```

**One-liners:**
- `kohaku` = the privacy engine + SDK (EF).
- `kohaku-extension` = the official reference browser wallet using that SDK (EF, Ambire fork).
- `kohaku-commons` = the Ambire-derived wallet logic the extension stands on (EF).
- `kohaku-cli` = a third-party CLI wallet that plugs into the same published SDK (community).
