# 04 — The wallet: `kohaku-extension` + `kohaku-commons`

These two repos are the **reference product**, and both are **Ambire forks**.
`kohaku-commons` *is* `ambire-common` (package name unchanged, v2.68). The
extension consumes it (pulled in as `src/ambire-common`, a submodule) and adds
the browser-extension shell + privacy UI. Read them together; the golden rule is
**separate inherited-Ambire from Kohaku-added**.

---

## A. `kohaku-commons` (= `ambire-common`)

A **headless, framework-agnostic** TS library: the brains of the wallet, no UI.
Apps compile it from source (no published `dist`).

### The Ambire base machinery (inherited)

**Controller pattern** (`src/README.md` is the design doc): stateful, event-emitting
ES6 classes. A singleton `MainController` (`src/controllers/main/main.ts`)
constructs and orchestrates child controllers (Accounts, SelectedAccount, Keystore,
Networks, Providers, Portfolio, DefiPositions, SignAccountOp, Requests, Activity,
Transfer, SwapAndBridge, Dapps, AddressBook, EmailVault, …). Children
`emitUpdate()`; the parent watches; public methods return `void` (state flows
through events, not return values). Deliberately not Redux/MobX — chosen for
framework-independence (extension, future mobile, etc.).

**`AccountOp`** (`src/libs/accountOp/accountOp.ts`) is the universal transaction
abstraction: an account address + chain + a batch of `Call[]` + fee/activator calls
+ gas + signature + a `gasFeePayment` strategy + `meta` (which can carry a 7702
`delegation`, a paymaster, a swap request, etc.). One `AccountOp` can be broadcast
three ways (`src/libs/broadcast/`): direct EOA, ERC-4337 UserOp
(`src/libs/userOperation/`), or the Ambire relayer. Gas is estimated by
**deployless** simulation (`src/libs/deployless/` — run contracts via `eth_call`
with state overrides). The **humanizer** (`src/libs/humanizer/`) turns raw calls
into readable actions.

**Account types** (`src/libs/account/`): EOA, EOA7702, V1 (legacy QuickAcc), V2
(`AmbireAccount.sol` smart account with a privilege map). Contracts live in
`contracts/` (`AmbireAccount.sol`, factory, paymaster, `AmbireAccount7702.sol`,
DKIM recovery). Supports ERC-4337 / 7702 / 5792 / 7579. **Heavily audited**
(`audits/`: Code4rena 2023, Pashov, Hunter, Shieldify, …) — this is the mature part.

### The Kohaku privacy additions (new)

Three privacy controllers are constructed by `MainController`
(`main.ts:~516-557`) and given handles to keystore/accounts/portfolio/activity:

- **`controllers/railgun/railgun.ts`** — derives Railgun keys with the
  `derive-railgun-keys` npm package, holds local merkle-tree/notebook cache, builds
  deposit/withdraw `AccountOp`s, runs them through the *same* signing flow.
- **`controllers/privacyPools/privacyPools.ts`** (v2) and **`privacyPoolsV1.ts`** (v1).
  v1 wires the **`@kohaku-eth/privacy-pools`** SDK: `createPPv1Plugin`,
  `createPPv1Broadcaster`, `OxBowAspService`, `PrivacyPoolsV1_0xBow`.

The **bridge** is `controllers/privacyPools/hostFactory.ts` — it constructs a
`@kohaku-eth/plugins` **`Host`** out of Ambire's pieces: extract the seed from the
Ambire `KeystoreController`, build a `pluginKeystore.deriveAt(path)` from an
`HDNodeWallet`, wrap a viem provider, pass through storage. This is exactly the
note-01 seam: *ambire-common is just another Host implementation for Privacy Pools.*

Other Kohaku-added pieces:
- `libs/railgun/humanizer.ts`, `libs/privacyPools/humanizer.ts` — render "Private
  Transfer" / withdrawal actions in the activity feed.
- `services/provider/ColibriRpcProvider.ts` — an ethers `JsonRpcProvider` subclass
  that routes `send()` through CORPUS **colibri** for (optionally ZK-verified) RPC;
  chains 1/11155111/100/10200.
- `controllers/privacyPools/derivation.ts` — `getAppSecret()`: EIP-712-signature →
  HKDF-SHA256 → deterministic per-app secret.
- Signing is routed by a `SignAccountOpType` switch (`main.ts:~826`):
  `…_MAIN | _SWAP | _TRANSFER | _PRIVACY_POOLS | _PRIVACY_POOLS_V1 | _RAILGUN`,
  each privacy controller owning its own `SignAccountOpController`. **No smart-contract
  changes** — privacy ops are ordinary `AccountOp`s with privacy `meta`.

---

## B. `kohaku-extension`

A **Manifest V3** browser wallet (Chrome/Firefox/Safari), Expo + Webpack +
**React Native Web**, app version 5.18 — i.e. the Ambire extension, reskinned and
extended. Yarn, Node 22.

### Inherited extension machinery

- **Background service worker** (`src/web/extension-services/background/background.ts`)
  instantiates the ambire-common `MainController` with Kohaku config (railgun relayer,
  PP relayer + ASP URLs, HyperSync key, Pimlico bundler, Alchemy key, **`testnetMode: true`**),
  debounces controller updates to the UI, and schedules portfolio/defi/activity polling.
- **Inpage + content script** (`extension-services/inpage/EthereumProvider.ts`,
  `content-script/…`) inject `window.ethereum` (advertises `isMetaMask` + `isAmbire`
  for compat). dapp request path: inpage → content-script relay → background
  `handleProviderRequests` → approval popup → signed response. `eth_subscribe`/
  `eth_sign` are stubbed/rejected for safety.
- Keystore, hardware wallets (Ledger/Trezor/GridPlus), portfolio, dapp catalog, swap
  & bridge — all Ambire.

### Kohaku privacy additions (the part that's actually new)

Two UI feature modules + a sync context, layered as **side-channels** (separate
tabs), not as a replacement for the selected main account:

- **Railgun** — `src/web/modules/railgun/hooks/useRailgunForm.ts` +
  `src/web/contexts/railgunControllerStateContext/`. Tracks a derived zk-account
  (`derived:0`), boots its state from a bundled `sepolia-checkpoint.json`, then syncs
  logs via **colibri** light-client indexing + an RPC `logsProvider`, verifies the
  merkle root, caches the account back through the background. **Deposit/shield is
  implemented** (ERC-20 `approve` + `shield`, or `shieldNative`); **withdrawal is a
  TODO stub** (`useRailgunForm.ts:~426`). Notably this uses the **older
  `@kohaku-eth/railgun` 0.0.1-alpha.8** surface (`createRailgunAccount`,
  `createRailgunIndexer`, `Indexer`) — *not* the unified `createRailgunPlugin`/Host
  path the CLI uses. So the extension hasn't migrated Railgun onto the plugin model.
- **Privacy Pools v1** — `src/web/modules/PPv1/`. Deposit, **withdrawal (implemented:
  note selection by privacy score → batched ZK proofs → relayer)**, **ragequit**,
  and in-pool **transfer**. Built on `@0xbow/privacy-pools-core-sdk` directly (plus
  `@kohaku-eth/privacy-pools` types). ASP membership fetched from a server; proofs
  verified locally before relaying.

Privacy account models differ: **Railgun** is unilateral/self-custodial (client-side
sync, inherent privacy); **Privacy Pools** is opt-in with an **ASP review** step
(accounts carry a `reviewStatus`: pending/approved/declined/exited/spent, and you can
`ragequit` un-approved deposits). **Network is Sepolia-only** (`DEFAULT_CHAIN_ID =
11155111`, with UI copy "mainnet coming soon").

### Extension dependency cheat-sheet

```
@kohaku-eth/railgun        0.0.1-alpha.8   (Railgun account/indexer — older API)
@kohaku-eth/privacy-pools  0.0.2-alpha.4
@kohaku-eth/provider       0.1.0-alpha.5
@kohaku-eth/plugins        0.0.1-alpha.5
@0xbow/privacy-pools-core-sdk  0.1.22       (PPv1 proofs used directly)
@corpus-core/colibri-stateless ^1.1.1       (Railgun light-client indexing)
snarkjs                    ^0.7.5
+ ambire-common (submodule)
```

### Maturity markers

Functional: account creation, dapp connect, Railgun deposit, PP deposit/withdraw/
ragequit/transfer. Stubbed/incomplete: **Railgun withdrawal in UI**, WETH-wrap
checkbox, mnemonic import. Testnet-only, alpha SDK deps. A prebuilt demo (IPFS,
"March 2026") exists for testing.

---

## How extension ↔ commons ↔ SDK fit

```
React Native Web UI (extension)
   │  contexts/hooks bind to…
MainController + privacy controllers   (kohaku-commons = ambire-common)
   │  Railgun via derive-railgun-keys + cache;  Privacy Pools via hostFactory → Host
@kohaku-eth/{railgun, privacy-pools, provider, plugins}   (the SDK, note 02)
   │  Railgun: Rust→WASM proving;  PP: TS proving;  reads via colibri light client
Ethereum (Sepolia)  +  relayers (railgun, fastrelay)  +  Pimlico bundler  +  ASP server
```
