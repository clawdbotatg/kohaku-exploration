# 05 — `kohaku-cli` (kassandraoftroy)

> ⚠️ **MAINNET WARNING:** the CLI defaults to mainnet and will shield **real ETH**,
> but its Railgun **unshield is broken on mainnet** — the EIP-7702 contracts it
> requires are testnet-only and undeployed on L1. Shielding on mainnet is a one-way
> door with this tool. Full writeup: [07-mainnet-demo-and-unshield-blocker.md](07-mainnet-demo-and-unshield-blocker.md).

A ~4k-LOC TypeScript **terminal wallet** by **kassandra.eth** — community, *not*
the EF org (MIT, copyright "kassandra.eth 2026"; single commit, v0.0.1). It's the
cleanest demonstration of the note-01 thesis: a thin host that **implements the
`Host` interface directly** and delegates *all* protocol logic to the published
`@kohaku-eth/*` packages. No forking, no reimplementation.

It also tracks the **newest** SDK builds (`@kohaku-eth/railgun 0.0.1-alpha.21`,
`privacy-pools 0.0.2-alpha.9`, `plugins 0.0.1-alpha.8`) and uses the unified
`createRailgunPlugin(host, …)` / `createPPv1Plugin(host, …)` entry points — so
it's the best place to see the intended plugin model in action.

Built with Commander.js + tsup; Node 22+. Stores everything under
`~/.kohaku-cli/<wallet>/`.

## Commands (`src/index.ts` → `src/commands/*`)

| Command | What it does |
|---|---|
| `create-wallet <name>` | Generate or `--import` a BIP-39 seed; on import, scan RPC to find the highest used HD index; encrypt seed to disk; tag mainnet/testnet. |
| `list-wallets` | List wallets + their network. |
| `next-fresh-address` | Derive & persist the next HD public account; print its address (a fresh unshield target). |
| `balances` | Aggregate **public** balances (ETH + default & discovered ERC-20s) and **private** balances for *both* Railgun and Privacy Pools. `--verbose` adds per-address + per-note detail. |
| `shield` | Public account → private protocol. `--protocol railgun\|privacy-pools`. Dry-run by default; `--broadcast` to send. Auto-handles ERC-20 approval. |
| `unshield` | Private balance → public address, via the protocol's relayer/bundler. `--to` or `--next`. |
| `see-decrypted-storage <public\|railgun\|privacy-pools>` | Debug: decrypt and print a storage file. |

Every command supports **`--non-interactive`** → no prompts/spinners, JSON to
stdout, BigInt serialized as strings (`src/utils/json-bigint.ts`). This is "agent
mode" — explicitly designed so scripts/agents can drive the wallet.

## The Host implementation (`src/host/`, `src/utils/plugins.ts`)

`makeHost({ rpc, walletDir, password, mnemonic, pluginId })` returns a `Host`:

- **storage** (`storage.ts`) — an in-memory map hydrated from an AES-256-GCM JSON
  file (`rg-storage.json` / `ppv1-storage.json`); every `set()` re-encrypts and
  rewrites the whole file. So plugin state (notes, proofs, sync cursor) is encrypted
  at rest with the wallet password.
- **keystore** (`keystore.ts`) — **two flavors**: `makeRailgunKeystore` uses
  `derive-railgun-keys`' `deriveRailgunKey(mnemonic, path)`; `makeKeystore` uses
  standard BIP-44 `Mnemonic.to0xPrivateKey`. (Railgun's spending/viewing keys live
  in a different derivation space than ordinary EOAs.)
- **provider** — ethers provider wrapped with `withTransactionCount` (adds
  `getTransactionCount`, needed by Railgun unshield simulation) and
  `withChunkedGetLogs`.
- **network** — Node global `fetch` (for the PP ASP service).

**`chunked-get-logs.ts`** is worth knowing: public RPCs cap `eth_getLogs` block
spans, but Railgun/PP sync from a deploy block forward. It splits any range into
windows (default 499 blocks, override `KOHAKU_GETLOGS_MAX_BLOCK_SPAN`) and
concatenates results, intercepting both the typed `getLogs` and raw `eth_getLogs`.

`createProtocolPlugin(protocol, host, chainId)`:
- railgun → `createRailgunPlugin(host, { rpcBatchSize: 450 })`
- privacy-pools → `createPPv1Plugin(host, { entrypoint, broadcasterUrl: fastrelay.xyz, aspServiceFactory: OxBowAspService(sepolia), initialState })`

## Seed & key management (`src/utils/`)

- `mnemonic.ts` — `Mnemonic.generate/validate` (from `derive-railgun-keys`).
- `aes-storage.ts` — AES-256-GCM with a PBKDF2-HMAC-SHA256 key (310k iterations,
  OWASP 2023), per-file random salt + per-write random IV, GCM tag. Envelope
  `{v,salt,iv,tag,ciphertext}`. Encrypts: the seed (`.encrypted-seed.json`), the
  public-accounts cache, and both plugin stores.
- `public-accounts.ts` — derives EOAs at BIP-44 indices, caches address + (encrypted)
  private key + balances; `addNextAccounts`/`peekNextAccounts` for fresh addresses.

## Shield flow (end to end)

`shield --protocol railgun --from 0 --token eth --amount-formatted 1 --broadcast`:

1. resolve wallet/password/mnemonic/chainId; validate token (PP ERC-20 must be
   whitelisted).
2. `makeHost(...)` + `createProtocolPlugin(...)`.
3. build `AssetAmount` (Railgun ETH → `{__type:'native'}`; PP ETH → `{__type:'erc20',
   contract: ETH_AS_ERC20 (0xee…ee)}`).
4. `plugin.prepareShield(asset)` → `{to,data,value}` txn(s).
5. if ERC-20 and allowance < amount, prepend an `approve` txn.
6. **dry-run** (default): print the txns (JSON or table) and stop.
7. **`--broadcast`**: create an ethers `Wallet` from the sender key, send approval
   (wait), simulate the shield (`gasLimit 2_000_000`), confirm, `sendTransaction`,
   wait. Output the hash. (These are ordinary signed txns — shields are *public*.)

## Unshield flow (end to end)

`unshield --protocol railgun --next --token eth --amount-formatted 0.5 --broadcast`:

1. resolve recipient: `--next` (derive/persist a fresh public account) or `--to`.
2. **Railgun requires the recipient's private key** — it's set as the EIP-7702
   *delegating signer*: `configureRailgunForUnshield(plugin, recipientPriv,
   pimlicoBundlerUrl)` calls `plugin.setBundler(Bundler.pimlico(url))` +
   `plugin.setDelegatingSigner(Signer.privateKey(priv))`. (PP allows an arbitrary
   recipient with no key.)
3. PP only: `plugin.sync()` first.
4. compute a **max-amount hint**: PP = the *largest single note* (the protocol can't
   merge notes in one withdrawal proof); Railgun = the *summed* balance (it can).
5. build asset (Railgun ETH → `railgunNativeEthAssetAmount` = native+WETH so it
   unwraps to the recipient).
6. `plugin.prepareUnshield(asset, recipient)` → a `PrivateOperation` (proof + payload).
7. dry-run prints the prepared op. `--broadcast`:
   - **Railgun** → `plugin.broadcast(op)` (submits the 4337 UserOp via the Pimlico
     bundler set in step 2).
   - **Privacy Pools** → `createPPv1Broadcaster(host, {broadcasterUrl: fastrelay.xyz})`
     then `.broadcast(op)` (relayer submits on-chain).

No CLI-level signing of the spend — the proof *is* the authorization; the relayer/
bundler pays gas and submits.

## Balances flow

For each protocol: spin up a Host+plugin, call `plugin.balance()`. Public side:
iterate stored accounts, `getBalance` + ERC-20 `balanceOf` over a merged token list
(per-chain defaults + `--tokensList` + tokens seen in private balances). `--verbose`
adds PP per-note detail (label, balance, asset, approved, precommitment).

## Why it's a good reference

If you want to understand the SDK without the Ambire baggage, read kohaku-cli: it
shows precisely what a wallet must provide (a Host) and what it gets back (uniform
shield/transfer/unshield/balance), with the two protocols' real-world quirks
visible (Railgun's 7702+bundler+paymaster unshield; PP's relayer + largest-note +
ASP).
