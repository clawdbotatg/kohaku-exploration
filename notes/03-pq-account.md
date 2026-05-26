# 03 — The post-quantum account (`kohaku/packages/pq-account`)

A standalone-ish package: an **ERC-4337 smart account that requires two
signatures** — one classical, one post-quantum — for every UserOp. It's the
EF's hedge against "harvest now, decrypt later": even if a quantum computer
breaks ECDSA tomorrow, an attacker still can't forge the lattice signature, so
funds stay safe. It does **not** touch the Railgun/Privacy-Pools stack; it's a
parallel research deliverable in the same repo, built on the **ZKNOX** verifier
contracts.

## The account contract (verified verbatim)

`src/ZKNOX_ERC4337_account.sol` (just 99 lines) extends `BaseAccount` from the
canonical `account-abstraction` lib. Storage holds two public keys and two
**verifier contract addresses**:

```solidity
bytes    preQuantumPubKey;                  // classical (ECDSA k1 or r1)
bytes    postQuantumPubKey;                 // PQ (ML-DSA / Falcon)
address  preQuantumLogicContractAddress;    // an ISigVerifier
address  postQuantumLogicContractAddress;   // an ISigVerifier
```

`_validateSignature` decodes the UserOp signature as
`abi.decode(sig, (bytes, bytes))` → `(preQuantumSig, postQuantumSig)`, then calls
`isValid(...)`, which verifies **both** and ANDs the result:

```solidity
if (preQuantumCore.verify(preQPubKey, digest, preQuantumSig) != preQuantumCore.verify.selector) return false;
if (postQuantumCore.verify(postQPubKey, digest, postQuantumSig) != postQuantumCore.verify.selector) return false;
return true;
```

So validation succeeds only if *both* schemes accept the `userOpHash`. The
verifier-returns-its-own-selector convention is the ERC-7913 / ERC-1271 style
"magic value" pattern. Verifiers conform to `ISigVerifier { setKey(bytes) returns(bytes); verify(bytes pubKey, bytes32 digest, bytes sig) returns(bytes4); }`.
The constructor calls `setKey` on each verifier (for ML-DSA the large key lives in
a separate "PK contract"). `ZKNOX_PQFactory.sol` is the CREATE2 factory.

## The verifiers (`lib/`)

Pre-quantum (cheap, use precompiles):
- **ECDSA-k1** (secp256k1) via `ecrecover`. Signature `[r,s,v]` (65 bytes).
- **ECDSA-r1** (secp256r1 / P-256) via the P-256 verify precompile (the ERC-7913 /
  RIP-7212 direction). Signature `[r,s]` (64 bytes).

Post-quantum (expensive, real lattice math on-chain — this is the ZKNOX research):
- **ZKNOX_dilithium** — ML-DSA-44. Reconstructs the `A_hat` matrix by rejection
  sampling, NTT, modular reduction. Most expensive.
- **ZKNOX_ethdilithium** — an EVM-optimized ML-DSA variant.
- **ZKNOX_falcon** — Falcon-512 (FFT / gram-matrix checks).
- **ZKNOX_ethfalcon** — EVM-optimized Falcon. Cheapest PQ option.

"ETH" variants trade some standard-compliance for big gas savings via compact
encodings and precompute. **ZKNOX** is the umbrella research project for these
on-chain post-quantum verifiers.

## Scheme matrix and gas

8 combos = {k1, r1} × {MLDSA, MLDSAETH, Falcon, ETHFALCON}, each with its own
factory deployed on **Sepolia and Arbitrum Sepolia** (addresses in
`deployments/deployments.json`). Gas to validate (from `README.md`):

| pre ↓ / post → | MLDSA | MLDSAETH | Falcon | ETHFALCON |
|---|---|---|---|---|
| **ECDSA-k1** | 8.39M | 5.12M | 4.06M | **1.69M** |
| **ECDSA-r1** | 8.40M | 5.13M | 4.07M | 1.70M |

Cost is dominated by the PQ verify + the signature bytes carried in calldata
(4 gas/non-zero byte). The classical scheme barely matters (~1%). Practical sweet
spot today: **r1 + ETHFALCON** (~1.70M).

## Off-chain signing utilities (`js/`)

A reference signer stack so you can actually produce the dual signature:
- `software-signer/` — pure-JS keygen + sign:
  - ECDSA via ethers.
  - **ML-DSA-44 via `@noble/post-quantum`** (`ml_dsa44.keygen`/`.sign`), 2560-byte pubkey.
  - **Falcon-512 via a Falcon WASM module** (compiled C reference) with manual heap
    management; 1025-byte pubkey, ~1064-byte compact signature.
- `hardware-signer/` — Ledger (WebHID) for ECDSA + ML-DSA; Falcon is WASM-only.
- `utils_mldsa.js` / `utils.js` — encode pubkeys for on-chain storage: expand ML-DSA
  `A_hat` (rejection sampling with SHAKE128, q=8380417) and pack Falcon's 512
  coefficients into 32 `uint256` words (`nttCompact`).
- `createAccount.js`, `sendTransaction.js`, `userOperation.js` — deploy an account and
  build/sign/submit a UserOp: build → dummy-sign for gas estimate → sign with both
  signers → `abi.encode([ecdsaSig, pqSig])` → submit to a bundler.

## Test suite

Foundry, one `.t.sol` per combo (each: success, invalid-sig rejection, full execute
through the EntryPoint) plus `…_onchain_fixed_contracts.t.sol` that exercises the
live Arbitrum-Sepolia deployments. ML-DSA signing in tests shells out to a Python
reference signer (`lib/ETHDILITHIUM/pythonref`).

## Relationship to the rest of Kohaku

Standalone today — it implements `IBaseAccount` and the standard EntryPoint
(`0x0000000071727De22E5E9d8BAf0edAc6f37da032`), shares only ethers /
`@noble/post-quantum` with the rest. Conceptually it's the "the wallet's *account*
can also be quantum-safe" research track, separate from the "the wallet's *txns*
can be private" Railgun/PP track. Nothing wires a Railgun spend through a
pq-account yet.
