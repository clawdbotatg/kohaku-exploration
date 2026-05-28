# 08 — Headless mainnet recovery: how we got the ETH back out

> **Result:** **0.01073310 ETH landed at the recipient's external address** on
> mainnet (May 2026), recovered entirely **headlessly** via the official
> `@railgun-community/wallet` SDK after kohaku-cli's mainnet unshield blocker
> ([07-mainnet-demo-and-unshield-blocker.md](07-mainnet-demo-and-unshield-blocker.md)).
> Of the 0.011 ETH funded, ~0.00027 ETH (~2.4%) went to Railgun's 0.25%+0.25%
> fees and three real mainnet txs' gas. **Funds shielded via kohaku-cli on
> mainnet are recoverable; you just can't use kohaku-cli to do it.**

## Why recovery is possible at all

Because **kohaku's `derive-railgun-keys` is a byte-identical reimplementation of
RAILGUN's official key derivation** (note 07 has the source-level proof: same
paths `m/44'/1984'/0'/0'/{i}'` + `m/420'/.../{i}'`, same `HMAC-SHA512("babyjubjub
seed", seed)` master key, same hardened child derivation — and the kohaku type
literally cites `Railgun-Community/engine/.../wallet-node.ts#L17`). The same
BIP-39 mnemonic therefore produces the same RAILGUN `0zk` address in both
toolchains. The official engine sees the kohaku-shielded notes and can spend them.
Empirically confirmed during this recovery: the official SDK reported the exact
`0.009975 WETH` (= 0.01 − 0.25% Railgun shield fee) we shielded.

## The toolchain

- **`@railgun-community/wallet@10.8.6`** — the official SDK. Same protocol contracts
  kohaku uses (RailgunSmartWallet, RelayAdapt), but a different unshield path:
  no EIP-7702, no hardcoded "Privacy Account" delegate — just a proof + a tx to
  the real RelayAdapt that any EOA can self-relay.
- **Local mainnet Ethereum node** (`192.168.68.62:8545`) — all chain RPC reads and
  the tx submissions went here, not to a third party.
- **`ppoi.fdi.network`** — a live RAILGUN POI aggregator node, required by the
  engine on mainnet (see Obstacle 4 below).
- **`snarkjs`** — the groth16 prover (the SDK doesn't bundle one; see Obstacle 6).
- **`memdown`** — in-memory `AbstractLevelDOWN` DB, sufficient for a one-shot
  recovery and avoids native-compile fragility.
- **`ethers v6`** — derives the demo EOA key from the same seed for self-relay
  signing and hop-2 forwarding.

## Strategy: safe 2-hop recovery

Direct-to-external-recipient unshield is technically possible (the proof binds
the recipient, not the sender), but with real money I didn't want to bet on the
recipient-binding semantics under `sendWithPublicWallet: true`. So:

1. **Hop 1 — official Railgun unshield**: `publicWalletAddress` = signer =
   recipient = the demo EOA (the *standard* self-relay pattern; zero ambiguity).
   Gas paid by the demo EOA's existing dust.
2. **Hop 2 — plain ETH send**: demo EOA → external recipient. Trivial 21k-gas tx.

The trade-off: hop 2 publicly links the demo EOA to the external recipient.
Privacy was already moot for *recovery* (the deposit EOA's dust paid hop-1 gas
anyway, linking deposit to the unshield tx). The user controls all the addresses.

## The obstacles, in order

This is the meat. Each was a real wall; recording the fix saves the next person
a day.

### 1. Built `dist` of kohaku-cli crashes on an ESM/CJS bug
Importing `@kohaku-eth/privacy-pools` crashes at runtime: `ERR_MODULE_NOT_FOUND
… maci-crypto/build/ts/hashing` — Node's strict ESM resolver rejects the
extensionless import. **Fix:** run the CLI via `tsx` (more forgiving resolver):
`npx tsx src/index.ts …`.

### 2. `npm install @railgun-community/wallet` hung indefinitely
`npm@10`'s strict peer-dependency resolver thrashes on `shared-models`'
`ethers@6.14.3` peer constraint. Ran ~1h40m and wrote *nothing* to `node_modules`.
**Fix:** `npm install … --legacy-peer-deps --no-audit --no-fund` with `ethers@6.14.3`
pinned exactly. Installed in <2 min.

### 3. `loadProvider` rejected my fallback config with "Invalid for chain 1"
RAILGUN's `createFallbackProviderFromJsonConfig` (in `shared-models`)
**requires total provider weight ≥ 2** for the quorum. With a single provider
at `weight: 1`, the sum was 1 → reject. **Fix:** single provider, `weight: 2`.

```js
const fallback = {
  chainId: 1,
  providers: [{ provider: RPC, priority: 1, weight: 2, stallTimeout: 5000 }],
};
```

### 4. Mainnet requires a POI node URL, and the documented one is dead
`loadProvider` then threw: "This network requires Proof Of Innocence. Pass
'poiNodeURL' to startRailgunEngine…". The official RAILGUN docs only list a
*test* aggregator (`https://ppoi-agg.horsewithsixlegs.xyz`) and say "ask the
community for production nodes." That URL is now **NXDOMAIN** — confirmed via
`curl` (HTTP 000) and via a 2026 community skill which explicitly notes the
takedown.

**Fix:** GitHub code-search across the RAILGUN community + downstream wallets
surfaced a current live alternative — **`https://ppoi.fdi.network`** (HTTP 200,
serving the POI JSON-RPC). Passed as `poiNodeURLs` to `startRailgunEngine`:

```js
await startRailgunEngine("kohakurecover", memdown(), false, artifactStore,
  false, false, ["https://ppoi.fdi.network"]);
```

### 5. The engine wants `AbstractLevelDOWN`, not modern `level`
`startRailgunEngine`'s `db` parameter expects the legacy `abstract-leveldown`
interface (not `abstract-level`). The modern `level@8+` doesn't satisfy it.
`leveldown` is native (compile risk). **Fix:** `memdown@6.1.1` — pure JS,
implements `abstract-leveldown`. In-memory means a re-scan each run, which is
acceptable for a one-shot recovery.

### 6. "Requires groth16 full prover implementation"
The SDK doesn't bundle `snarkjs` — the app supplies the groth16 implementation.
**Fix:** `npm i snarkjs`, then immediately after engine start:

```js
import * as snarkjs from "snarkjs";
import { getProver } from "@railgun-community/wallet";
getProver().setSnarkJSGroth16(snarkjs.groth16);
```

### 7. "Timed out downloading artifact files for 01x01 circuit"
Circuit artifacts (`.wasm`, `.zkey`) are downloaded on demand from RAILGUN's
artifact CDN; the first attempt timed out at ~5%. **Fix:** retry. The
`ArtifactStore` caches to disk, so partial state resumes. Second run went
5 → 20 → 33 → 45 → 99 → 100%.

### Bonus: the seed never leaks
The mnemonic is encrypted in `~/.kohaku-cli/demo/`. To avoid putting it in
argv, on disk, or any transcript, both the read-only check and the unshield
read it from **stdin**, piped from a tiny kohaku-side helper that decrypts the
keystore in-memory and writes ONLY the raw mnemonic to stdout. The pipe
connects two processes; neither displays the seed.

```bash
npx tsx kohaku-cli/print-seed.ts \
  | node --max-old-space-size=8192 rg-recover/unshield.mjs
```

`unshield.mjs` then derives the demo EOA from the same seed via `ethers.HDNodeWallet.fromPhrase()`
and **asserts the derived address equals the known demo EOA** before signing — so
a wrong seed/path can't accidentally sign with someone else's key.

## The recovery flow in code (shape only)

```
startRailgunEngine("…", memdown(), false, artifactStore, false, false, POI_NODES)
getProver().setSnarkJSGroth16(snarkjs.groth16)
loadProvider({chainId:1, providers:[{provider: localRPC, weight:2, …}]}, "Ethereum")
const info   = await createRailgunWallet(encKey, mnemonic, {Ethereum: SHIELD_BLOCK})
await refreshBalances(chain, [info.id])
await refreshReceivePOIsForWallet(TXID, "Ethereum", info.id)
await refreshBalances(chain, [info.id])
const spendable = await balanceForERC20Token(TXID, wallet, "Ethereum", WETH, true)
// gas estimate -> generate proof -> populate -> sign with demo EOA -> send
```

## On-chain receipts

| Step | Tx | Block | What happened |
|---|---|---|---|
| Shield (via kohaku-cli) | `0x370b12be41b04d56536dcb19a48cae704a34931fd033b79801f7795d25ee50b6` | 25,183,142 | 0.01 ETH → Railgun pool (via RelayAdapt) |
| Unshield (via official SDK, headless) | `0x02c6c7f8707e77e51352f17ad7f3a3a50dc39a6a6e12230dc176f0c76dd815a5` | 25,190,827 | 0.009975 WETH unshielded + unwrapped → demo EOA |
| Forward (plain send) | `0xf4d179ec178ac428375cdcfc15fe9df8c2e59a16bff30c7805131037020500ab` | 25,190,834 | demo EOA → recipient (external wallet) |

**Demo wallet's RAILGUN address** (publicly derivable from the seed; provided here
for the record): `0zk1qy85e6lvyr84hw0t4ks5jv4w8y9py4ld8480duau42vvpyccjfnxhrv7j6fe3z53lun2yu40plq5ux9hd82c3tn43nhmjt2gkte8qp4rc6kwm4jtr3tmzzsa6wt`

## Cost breakdown

| Item | ETH |
|---|---|
| Original deposit | 0.011000 |
| Shield gas (kohaku-cli) | ~0.000038 |
| RAILGUN shield fee (0.25%) | 0.000025 |
| RAILGUN unshield fee (0.25%) | ~0.000025 |
| Unshield gas (proof verify on-chain, ~532k gas) | ~0.000096 |
| Forward gas (21k) | ~0.000004 |
| **Final at recipient** | **0.01073310** |

~2.4% total overhead — most of which is the protocol fees (0.5%) and the
heavy on-chain proof verification of the unshield tx.

## Takeaways for anyone in the same hole

1. **The funds aren't stuck** — if you have the seed, the official Railgun
   wallet (railway.xyz, 5 minutes) is the easiest path, and the headless
   `@railgun-community/wallet` SDK route works too (this doc).
2. **POI node URLs go stale.** Don't trust the docs' example; search current
   community wallets for a live one.
3. **Total provider weight must be ≥ 2.** Bare-minimum config = one provider,
   weight 2.
4. **The SDK doesn't bundle a prover.** Install `snarkjs` and wire it via
   `getProver().setSnarkJSGroth16(snarkjs.groth16)` right after engine start.
5. **Artifact downloads will time out at least once.** Retry.
6. **For self-relay, set `publicWalletAddress` = signer = recipient.** It's
   the documented standard pattern; don't get clever with real money.
7. **The verdict stands:** kohaku-cli (and the broader Kohaku stack) is
   **NOT READY FOR MAINNET**. You can shield real ETH with kohaku-cli; you
   *can't* unshield with it. The fix has to come from kohaku — either deploy
   the EIP-7702 contracts on L1, or add a non-7702 unshield path, *and* add a
   preflight guard so the CLI refuses to shield on a chain whose unshield
   infra isn't deployed.
