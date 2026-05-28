# Kohaku — What the repos are (TLDR)

> **In one line:** Kohaku is privacy plumbing for Ethereum wallets — it lets you move
> money on Ethereum without it being publicly traceable. The pattern is
> **shield → send privately → unshield**: deposit funds into a shared pool, then
> send/withdraw them with zero-knowledge proofs so the activity can't be linked back
> to you — all wrapped so wallets can drop it in.

## ⚠️ Mainnet readiness: NOT READY — verified the hard way (May 2026)

We ran a **real-money mainnet round-trip** through `kohaku-cli`. The **shield worked**
(0.01 ETH into Railgun on mainnet). The **unshield is structurally broken on mainnet**:
the CLI's only Railgun-unshield path depends on EIP-7702 account-abstraction contracts
(a "Privacy Account" implementation + a paymaster) that **are not deployed on Ethereum
mainnet** — verified `0x` (no code). Worse, `kohaku-cli` **defaults to mainnet** with
**no preflight guard**, so it will accept a real deposit into a one-way door. The EF
repos themselves carry a "work in progress — not ready for production" notice, and the
browser extension is **Sepolia-testnet only**.

**Bottom line: do not use Kohaku with real funds on mainnet yet.** Shielding works;
getting back out does not (via this tooling). Funds shielded this way *are* recoverable
through the official Railgun wallet — the key derivation is byte-compatible — but the
Kohaku tooling itself cannot complete the round trip on mainnet.
- Evidence + tx hashes: [notes/07-mainnet-demo-and-unshield-blocker.md](notes/07-mainnet-demo-and-unshield-blocker.md)
- ✅ Headless recovery playbook (worked, all obstacles + fixes): [notes/08-headless-mainnet-recovery.md](notes/08-headless-mainnet-recovery.md)

---

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

---

## How is this different from Tornado Cash?

Big picture: **Tornado Cash is a single mixer dapp; Kohaku is a wallet that builds
privacy in as a feature** — and the protocols it uses (Railgun, Privacy Pools) are the
"lessons-learned" next generation of the idea Tornado pioneered. They're technically
cousins (both use commitments + nullifiers + Merkle trees + zk-SNARKs), but differ on
almost everything around that core.

| | Tornado Cash | Kohaku |
|---|---|---|
| **What it is** | A standalone mixer (contracts + website) | Wallet tooling — privacy embedded *in* the wallet, plugin-based |
| **Amounts** | Fixed denominations only (0.1 / 1 / 10 / 100 ETH) | Arbitrary amounts; you hold a real private balance |
| **Capability** | Deposit → withdraw, nothing else | A full private account: shield, **send privately pool-to-pool**, partial spends, unshield |
| **Assets** | One pool per asset/denomination | Many ERC-20s, native ETH, etc. |
| **Compliance** | None — all funds commingle | The whole point of the redesign (see below) |
| **Posture** | Anonymous-ish, immutable, OFAC-sanctioned 2022 | Ethereum Foundation, explicitly compliance-aware |

**The one difference that explains all the others: dissociation from dirty money.**
Tornado's fatal flaw was that honest users and hackers shared one anonymity set with no
way to tell them apart — which is exactly why it was sanctioned and its devs prosecuted.
Kohaku's protocols bake in the fix:

- **Privacy Pools** (born from a Vitalik-coauthored paper reacting to Tornado) uses an
  **ASP** (Association Set Provider): you can *prove* your withdrawal comes from an
  approved/clean set of deposits, cryptographically separating yourself from illicit
  funds while staying private.
- **Railgun** has **POI** (Proof of Innocence): proving your funds don't descend from
  blocklisted deposits.

Tornado had no equivalent. That's the headline.

Two smaller notes:
- Both use **relayers** so your withdrawal isn't paid for by a linkable address — same
  idea, but Kohaku layers modern account abstraction (ERC-4337 + EIP-7702 + a paymaster)
  on top.
- Tellingly, Kohaku's own code treats **Tornado as just the minimal case** of its
  framework — there's a `packages/plugins/examples/tornado.ts` plugin that models Tornado
  in the same plugin interface, but with only shield/unshield and *no* private internal
  transfer. In Kohaku's worldview, Tornado is the simplest version of the general thing
  they're building.
