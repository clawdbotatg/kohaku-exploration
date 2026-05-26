# 06 — Concepts & glossary

Background needed to read the rest of these notes. Two privacy protocols, two
account-abstraction primitives, and the ZK plumbing under both.

## The core idea: shield → transact → unshield

All of Kohaku's privacy works on a **pool** model:

- **Shield (deposit):** move a public asset *into* a shared pool contract. This is a
  normal, public, signed transaction — everyone sees you deposited. Your funds become
  a **commitment** (a hash) in the pool's Merkle tree; the secret behind it is yours.
- **Transact / transfer (inside the pool):** move value between pool positions
  *privately*, proven with zero-knowledge. No link between sender and receiver
  on-chain.
- **Unshield (withdraw):** move value *out* of the pool to a public address, proven
  with ZK so the withdrawal isn't linkable to your original deposit.

Privacy comes from the **anonymity set**: the more people in the pool, the less any
one withdrawal can be tied to a specific deposit. Shields are cheap and public;
spends/unshields are ZK-proved and usually **relayed** (so the gas-payer isn't you,
which would otherwise re-link your identity).

## Railgun

A **UTXO-based** shielded pool (think Zcash-style notes on Ethereum).

- **Note / UTXO:** an encrypted record of (asset, value, owner) sitting as a
  commitment leaf in a Merkle tree. Encrypted to the owner's **viewing key**.
- **Commitment:** `Poseidon(notePublicKey, assetHash, value)` — the public leaf.
- **Nullifier:** a one-way tag `Poseidon(nullifyingKey, leafIndex)` published when a
  note is spent, so it can't be spent twice — but it can't be linked back to which
  commitment it nullified.
- **Spending vs viewing keys:** viewing key decrypts incoming notes (and can be shared
  for auditing); spending key authorizes spends (BabyJubJub signature checked *inside*
  the circuit).
- **0zk address:** a Railgun address (`0zk1…`) encoding spending+viewing pubkeys.
- A spend proves, in zero knowledge: "I know notes that exist in the tree (Merkle
  membership), I'm authorized (signature), here are their nullifiers, and here are the
  new output commitments that conserve value." Circuit sizes are named `NxM`
  (N inputs × M outputs).

## POI / PPOI — Proof of Innocence

Railgun's **compliance layer**. To discourage the pool being a haven for hacked
funds, a spender can additionally prove their note descends only from shields that
are *not* on a blocklist — without revealing which note. Status per note:
`Valid / ShieldBlocked / ProofSubmitted / Missing`. Implemented with an extra
transaction-ID Merkle tree + per-list membership proofs. ("PPOI" = Private Proof of
Innocence; the docs use both.)

## Privacy Pools

The Ameen-Soleimani / Buterin-et-al "Privacy Pools" design (here, the **0xBow** v1
deployment). Same shield/withdraw shape, but the headline feature is the **ASP**:

- **ASP — Association Set Provider:** publishes an *association set* — the set of
  deposits considered "clean". When you withdraw, you prove membership in *both* the
  full deposit tree *and* the ASP's approved set. This lets honest users
  cryptographically dissociate from illicit deposits (regulatory-friendly privacy).
- **Review status:** a deposit is pending/approved/declined by the ASP. Kohaku surfaces
  this in the UI.
- **Ragequit:** if your deposit is *not* approved (or you just want out), you can
  withdraw it yourself back to the original depositor, bypassing the relayer/anonymity
  benefits — an escape hatch.
- **Relayer:** withdrawals are POSTed to a relayer (e.g. `fastrelay.xyz`) that submits
  the proof on-chain and takes a fee, so your withdrawal isn't paid by a linkable EOA.
- **Largest-note limitation (in Kohaku):** the current integration can't merge notes
  within one withdrawal proof, so a single unshield is capped at your biggest single
  note.

Kohaku's `@kohaku-eth/privacy-pools` is **pure TypeScript** (proving via `maci-crypto`
+ a circuits package + `@zk-kit/lean-imt`), whereas Railgun's core is **Rust→WASM**.

## ERC-4337 — account abstraction

Smart-contract wallets without consensus changes. A **UserOperation** (not a normal
tx) is sent to a **bundler** (e.g. Pimlico/alto), which calls a singleton
**EntryPoint** that invokes your account's `validateUserOp` then executes the call. A
**paymaster** can sponsor gas (or be repaid in tokens). Kohaku uses this for:
(a) the privacy unshield path, and (b) the post-quantum account (note 03).

## EIP-7702 — set EOA code

Lets a plain EOA temporarily *delegate* to contract code by signing an
**authorization**. Railgun uses it so an ordinary key can act as a "Privacy Account"
implementation for one UserOp — enabling the privacy paymaster to sponsor gas and be
repaid by an in-pool **fee note**, instead of the spender paying gas from a doxxing
address. (Also see EIP-7579 modular accounts and EIP-5792 `wallet_sendCalls`, both
supported by the Ambire base.)

## ZK plumbing

- **Groth16:** the succinct proof system Railgun uses. Tiny proofs, fast on-chain
  verify, but needs a per-circuit trusted setup → **proving key** + **matrices**
  artifacts (Kohaku downloads these at runtime from a GitHub artifacts repo).
- **circom / ark-circom:** circuits are written in circom; `ark-circom` (Arkworks)
  loads the compiled circuit WASM to compute the **witness** (the full set of signal
  values), which Groth16 then proves.
- **BN254 (alt-bn128):** the elliptic curve everything runs over (it has cheap EVM
  precompiles for pairing checks).
- **Poseidon:** a hash designed to be cheap *inside* a ZK circuit (few constraints),
  unlike keccak/SHA. Used for commitments, nullifiers, Merkle nodes.
- **BabyJubJub:** an elliptic curve embedded in BN254's scalar field, so signatures
  over it can be verified *inside* a BN254 circuit. Railgun spend authorization uses
  EdDSA-style BabyJubJub signatures with a Poseidon challenge.
- **Merkle tree (depth 16 in Railgun):** commitments are leaves; a spend proves
  membership via a sibling path; the contract checks the proof's root matches the
  on-chain root.

## Post-quantum terms (note 03)

- **ML-DSA (Dilithium):** NIST lattice signature standard. Large keys/sigs.
- **Falcon:** another lattice signature; smaller signatures, cheaper to verify on-chain.
- **ZKNOX:** the research project providing on-chain Solidity verifiers for these.
- **"ETH" variants:** EVM-gas-optimized encodings of the above.
- **secp256k1 (k1) / secp256r1 / P-256 (r1):** the two classical ECDSA curves; r1 has a
  verify precompile (RIP-7212 / the ERC-7913 direction).
- **Harvest-now-decrypt-later:** the threat model — record signatures today, forge them
  once quantum computers exist. The dual-signature account defeats it by also requiring
  a lattice signature.

## Glossary quick-reference

| Term | One-liner |
|---|---|
| Host | The 4 services (network/storage/keystore/provider) a wallet gives a Kohaku plugin |
| Plugin | A privacy protocol exposing `balance/prepareShield/prepareTransfer/prepareUnshield` |
| Public operation | Raw txns the user signs & broadcasts (shields) |
| Private operation | A ZK proof + payload handed to a relayer/bundler (transfers, unshields) |
| Shield / Unshield | Deposit into / withdraw out of a privacy pool |
| Commitment / Nullifier | Pool leaf hash / one-time spend tag |
| Viewing / Spending key | Decrypt incoming notes / authorize spends |
| ASP | Privacy Pools' approved-deposits set provider |
| POI / PPOI | Railgun's proof your funds aren't blocklisted |
| Relayer / Bundler | Third party that submits your private op so you don't pay gas from a linkable EOA |
| Subsquid / colibri / Helios / HyperSync | Indexers & light clients for reading chain state |
| ambire-common | The forked Ambire wallet-logic lib = `kohaku-commons` |
