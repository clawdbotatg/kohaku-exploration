# Kohaku deep-dive notes

Connected reference notes from reading the four repos in depth (May 2026).
Start with the architecture overview, then dive into whichever layer you need.

| # | Note | Covers |
|---|------|--------|
| — | [`../KOHAKU-OVERVIEW.md`](../KOHAKU-OVERVIEW.md) | One-page TLDR of the four repos and how they relate |
| 01 | [architecture-overview.md](01-architecture-overview.md) | The whole system; the `Host`/plugin seam that ties it together; who implements what |
| 02 | [sdk-core.md](02-sdk-core.md) | `ethereum/kohaku` — plugins, provider, Railgun (Rust→WASM), Privacy Pools, the WASM bridge |
| 03 | [pq-account.md](03-pq-account.md) | The post-quantum ERC-4337 account (dual-signature smart wallet) |
| 04 | [wallet-extension-and-commons.md](04-wallet-extension-and-commons.md) | The browser wallet (`kohaku-extension`) and its Ambire base (`kohaku-commons`) |
| 05 | [cli.md](05-cli.md) | `kohaku-cli` — the community terminal wallet, end-to-end flows |
| 06 | [concepts-and-glossary.md](06-concepts-and-glossary.md) | Railgun, Privacy Pools, POI, ASP, ERC-4337/7702, Poseidon/BabyJubJub, Groth16 — primers + glossary |

## The one thing to understand first

Everything in the SDK hangs off one tiny interface: **`Host`** in
`@kohaku-eth/plugins` (`kohaku/packages/plugins/src/host/index.ts`). A Host is
four services a wallet must provide — `network` (fetch), `storage` (KV),
`keystore` (BIP-32 `deriveAt(path)`), and `provider` (an `EthereumProvider`). A
**privacy protocol plugin** (Railgun, Privacy Pools) is constructed against a
Host and exposes a uniform, type-checked API: `balance()`, `prepareShield()`,
`prepareTransfer()`, `prepareUnshield()`.

So the same Railgun/Privacy-Pools logic runs unchanged in three different
"hosts": the browser extension, the CLI, and (for Privacy Pools) ambire-common.
Each just supplies storage/keys/RPC its own way. See note 01.

## Source layout (cloned, gitignored)

- `../kohaku/` — the SDK (Rust crates + TS packages)
- `../kohaku-extension/` — the browser wallet (Ambire fork)
- `../kohaku-commons/` — `ambire-common` fork (wallet core logic)
- `../kohaku-cli/` — the CLI (kassandraoftroy)

Citations in these notes are relative to each repo root (e.g.
`packages/plugins/src/host/index.ts` lives under `../kohaku/`).
