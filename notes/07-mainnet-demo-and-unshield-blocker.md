# 07 — Mainnet demo: shield works, unshield is BLOCKED (real-money finding)

> **VERDICT: Kohaku is NOT ready for mainnet.** You can shield real ETH but cannot
> unshield it with the current tooling — a one-way door. Don't use real funds on
> mainnet yet.

**Date:** 2026-05-26/27. **Tool:** `kohaku-cli` (kassandraoftroy), run against a
**local mainnet Ethereum node** (`192.168.68.62:8545`). **Real funds involved.**

## TL;DR

On Ethereum **mainnet**, kohaku-cli can **shield** real ETH into Railgun but
**cannot unshield it back out**. Its only Railgun-unshield path depends on
account-abstraction contracts that **are not deployed on mainnet** (only testnet).
The CLI nonetheless **defaults to mainnet** and happily takes the deposit — a
real footgun. Funds are not lost, but they're stranded from this tool's POV.

## What we did

1. Built kohaku-cli (had to run via `tsx`; the built `dist` crashes on an ESM
   bug in the published `@kohaku-eth/privacy-pools` → `maci-crypto` import).
2. Created a **mainnet** demo wallet (CLI defaults to chainId 1; `--testnet` is opt-in).
3. Funded deposit EOA `0x1845B0dae1705c8845bE6EabC6a6A2665fAC0189` with 0.011 ETH.
4. **Shield 0.01 ETH → Railgun: SUCCESS.**
   - tx `0x370b12be41b04d56536dcb19a48cae704a34931fd033b79801f7795d25ee50b6`
   - status `0x1`, block `25,183,142`, gas ~765k (~0.00004 ETH at 0.05 gwei)
   - routed through Railgun's real mainnet RelayAdapt `0xAc9f360Ae85469B27aEDdEaFC579Ef2d052aD405` (wraps ETH→WETH, shields)
5. Waited out the Railgun **Proof-of-Innocence** delay (~1h) — note became
   spendable right at ~62 min. (This part worked exactly as designed.)
6. **Unshield 0.008 ETH → fresh address: FAILED at the bundler:**
   ```
   Bundler error: RPC error -32602: Invalid EIP-7702 authorization:
   Delegate 0x304A1b31D6cC77616951579bD373A4bD8aeF4B4c has no code.
   ```

## Root cause (verified on-chain)

kohaku-cli's Railgun unshield is **only** implemented as a gasless ERC-4337 +
EIP-7702 flow (`configureRailgunForUnshield` → `setBundler(Bundler.pimlico)` +
`setDelegatingSigner`). The recipient EOA is 7702-delegated to a hardcoded
"Privacy Account" implementation, and a hardcoded privacy paymaster sponsors gas.
Both addresses are constants in the SDK (`crates/userop-kit/src/railgun.rs`):

| Contract | Address | Code on mainnet? |
|---|---|---|
| 7702 Privacy Account impl (`IMPL`) | `0x304A1b31D6cC77616951579bD373A4bD8aeF4B4c` | **`0x` — NOT deployed** |
| Privacy paymaster (`PAYMASTER`) | `0xBbbc86034C5371e098163A39eC1bb8B2f015bB74` | **`0x` — NOT deployed** |

(Verified via `eth_getCode` on the mainnet node — both return `0x`.) You cannot
EIP-7702-delegate to an address with no code, so the bundler rejects the op. These
contracts exist only on testnet, where the CLI author developed/tested the unshield
("fix: unshield via 4337 on railgun operational" — a Sepolia-era commit). There is
**no mainnet variant** and **no alternative unshield path** (no self-relay, no
classic Railgun broadcaster) in the CLI.

## The footgun

- kohaku-cli **defaults to mainnet** (`createWallet.ts`: `opts.testnet ? "11155111" : "1"`).
- Its README walkthrough uses **testnet**, but nothing stops you shielding **real**
  ETH on mainnet.
- **No preflight guard** checks that the unshield's 7702 delegate / paymaster are
  deployed on the active chain before accepting a shield. So you can move real money
  into a one-way door.

## Secondary issues found

- **Private key leaked to stdout:** the unshield flow `console.log`s the recipient's
  private key (debug line, "should not be in production"). Confirmed live.
- **Built `dist` is broken** (ESM/`maci-crypto` extension bug); only `tsx` works.

## Status of the funds

- **Safe, not lost.** 0.01 ETH (as WETH) sits in the Railgun pool under
  spending/viewing keys derived from the demo seed. The failed unshield spent **zero**
  on-chain (rejected pre-submission; deposit EOA dust unchanged at ~0.00096 ETH).
- **Recovery path — derivation compatibility CONFIRMED.** The **official Railgun wallet /
  `@railgun-community/wallet` SDK** unshields on mainnet via Railgun's own relayer/self-relay
  (no 7702) and can send to any address. The open question — does kohaku's `derive-railgun-keys`
  produce the *same* Railgun wallet as official? — is settled **yes**:
  - kohaku's derivation source (`derive-railgun-keys/dist/utils/bip32.js` + `railgun-node.js`)
    is **byte-identical** to Railgun's official key derivation: paths `m/44'/1984'/0'/0'/{i}'`
    (spending) + `m/420'/1984'/0'/0'/{i}'` (viewing), master key `HMAC-SHA512("babyjubjub seed", seed)`,
    hardened child `HMAC-SHA512(chainCode, 0x00‖chainKey‖index)` — same function names and constants
    as `@railgun-community/engine`.
  - The `@kohaku-eth/railgun` type for `spendingKeyPath`/`viewingKeyPath` **explicitly cites**
    `Railgun-Community/engine/.../key-derivation/wallet-node.ts#L17`.
  - Empirically, kohaku's keys already located + proved spendable notes in the **real** mainnet
    Railgun pool (post-POI), which only works with Railgun's real key/note scheme.
  - Demo wallet's Railgun address (index 0) holding the funds:
    `0zk1qy85e6lvyr84hw0t4ks5jv4w8y9py4ld8480duau42vvpyccjfnxkunpd9kxwatwqyn2yu40plq5ux9hd82c3tn43nhmjt2gkte8qp4rc6kwm4jtr3tmz9ajfze`
  - **Conclusion:** importing the same BIP-39 seed into the official Railgun wallet (railway.xyz)
    will show the 0.01 ETH and allow a normal unshield to any address.

## Lesson for this exploration

Before committing **real funds** through alpha tooling, verify the **entire
round-trip's on-chain dependencies exist on the target chain** — not just that the
protocol and the deposit path are live. Here the deposit (shield) used real, deployed
Railgun contracts, but the withdrawal (unshield) silently relied on testnet-only AA
contracts. That was checkable up front (the SDK hardcodes the addresses; we'd even
documented the 7702 unshield design in [02-sdk-core.md](02-sdk-core.md) §5) and was
not checked before depositing. Owned in the session writeup.
