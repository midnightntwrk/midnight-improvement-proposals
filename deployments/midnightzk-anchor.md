# midnightzk-anchor — Mainnet Deployment Authorization Application

> **DRAFT v2.0 — V2 design (owner-gated in-circuit authentication), revised per reviewer feedback.**
> On-chain state remains bounded-by-design (no growable storage). In addition, every state-mutating
> circuit is gated by an in-circuit owner check. The platform's Gateway layer (per-tenant API keys)
> is retained as an off-chain operational boundary, not as the primary on-chain authorization.

---

## 1. Summary

軒傑科技 (Hsuan-Jie Technology), a Taiwan-registered technology company, operates **MidnightZK** — a multi-tenant Privacy Proving Platform (PPP) that anchors data hashes onto Midnight on behalf of authenticated enterprise tenants. The applicant is the Co-founder of Hsuan-Jie Technology (軒傑科技).

The `midnightzk-anchor` contract publishes 32-byte cryptographic commitments contributed by authenticated tenants through the platform's M2M Gateway, producing tamper-evident on-chain records of existence and time. The platform serves enterprise use cases including:

- **Regulatory compliance** — EU PPWR digital passport for paper-bottle producers
- **Supply chain provenance** — TWCBAM tea/wine label traceability
- **Audit trails** — ConformPass compliance attestation
- **AI training data provenance** — SSATT.AI dataset commitment

We request Mainnet deployment authorization for the design described below.

---

## 1.1 Risk rubric

| Category | Score | Summary |
|---|---|---|
| Privacy-at-risk | 1 | Only 32-byte data hashes reach the chain. Raw tenant payloads, PII, and source data never leave the tenant's M2M boundary; the platform forwards hashes only. |
| Value-at-risk | 1 | Contract holds no on-chain assets — no tokens, no escrowed value, no balance state. Adversarial misuse cannot result in user fund loss. |
| State-space-at-risk | 1 | Fixed-size on-chain state by design: `last_anchor_hash` (single 32-byte slot, overwritten on each call), `anchor_count` (counter), and `owner` (single 32-byte slot). No map, no append-only structure, no growable storage — state-space is constant regardless of call rate. In addition, owner-gated writes mean only the platform operator key can mutate state at all. |

Detailed justification in §4. Evidence and verification in §5.

---

## 2. Contract at a glance

The contract source is `anchor.compact` (40 lines, `pragma language_version >= 0.20`, single file with no imports beyond `CompactStandardLibrary`).

**Ledger fields**:
- `last_anchor_hash: Bytes<32>` — single-slot storage for the most recent commitment hash
- `anchor_count: Counter` — total number of anchor calls processed
- `owner: Bytes<32>` — public key of the sole authorised writer (the platform operator)

**Witness**:
- `localSecretKey(): Bytes<32>` — supplies the operator secret key off-chain (file/env loader; the secret never reaches the chain)

**Pure circuit**:
- `publicKey(sk: Bytes<32>): Bytes<32>` — derives the 32-byte owner public key from a secret key via `persistentHash<Vector<2, Bytes<32>>>([pad(32, "midnightzk-anchor:pk:"), sk])`. The same circuit is used at deploy time (to compute `initialOwner`) and in-circuit (to authenticate), so the two derivations are byte-identical by construction.

**Circuits**:
- `constructor(initialOwner: Bytes<32>)` — sets `owner` once at deploy
- `anchor(fact_hash: Bytes<32>)` — owner-gated; asserts `publicKey(localSecretKey()) == owner` before overwriting `last_anchor_hash` and incrementing the counter
- `rotateOwner(newOwner: Bytes<32>)` — owner-gated; replaces `owner` atomically without redeploy

The contract design is intentionally minimal and bounded-by-design. On-chain state is a fixed-size auxiliary record; the substantive cryptographic proof is the on-chain transaction itself (immutable, indexable, queryable by `txId`). Owner-gating adds an in-circuit authorization layer so that only the platform operator key can mutate state.

---

## 3. Threat model

The adversary we consider is any party able to submit a transaction to Midnight (i.e., holding NIGHT/DUST sufficient to pay fees). The assets at risk are:

1. **Ledger state-space**: unbounded growth of contract storage.
2. **Customer proof integrity**: tenants' ability to prove they anchored a specific hash at a specific block.

For **ledger state-space**: the contract has no growable storage. Every call writes to the same single 32-byte slot, a single counter, and a single owner slot — all fixed-size. Adversarial calls cannot inflate state-space. Moreover, with owner-gated writes, a non-owner `anchor` or `rotateOwner` call is rejected in-circuit before any state change (see §4), so an adversary cannot even overwrite our slot — at most they burn their own fees on a transaction that fails the owner assertion.

For **customer proof integrity**: the substantive proof is the on-chain transaction record, not the mutable contract state. Each `anchor` call produces an immutable transaction containing the customer's hash, indexable by `txId` and verifiable by any third party against the indexer. Owner-gating prevents an adversary from overwriting `last_anchor_hash` at all; and even absent that gate, past transactions could not be erased — tenants' proofs persist in chain history regardless.

The platform's Gateway layer adds operational enforcement off-chain (see §6): per-tenant scoped API keys (`api_keys.scopes`), rate limiting (`CustomRateLimit` middleware), and `app:<tenantId>` namespacing. During the Federation period, only the DApp developer holds the API key for the authorized mainnet RPC endpoint; only the platform's operator key signs on-chain transactions.

**Critical clarification for the Federation period**: tenants do not deploy contracts. Tenants do not hold the operator secret key. Tenants do not interact directly with the chain. All on-chain writes originate from a single operator key — structurally identical to the single-deployer model approved in `deployments/gyotak-catch.md` (PR #96) and `deployments/nightforce-directory.md` (PR #79).

---

## 4. Score-1 justification

**Privacy-at-risk = 1**

The contract accepts only 32-byte hashes as input (`fact_hash: Bytes<32>`). Raw tenant data — payloads, PII, attachments, business records — never enters the contract or the chain. The platform's M2M Gateway hashes data tenant-side before submission; the chain sees only the hash.

**Value-at-risk = 1**

The contract holds no tokens, no escrowed value, no balance state, no claim on any on-chain or off-chain asset. There is no economic mechanism by which adversarial misuse can result in fund loss for any party.

**State-space-at-risk = 1**

The contract storage is fixed-size by design:

1. `last_anchor_hash: Bytes<32>` — a single 32-byte slot, overwritten on every call. Storage footprint does not grow with call count.
2. `anchor_count: Counter` — a single counter, incremented per call. Counter occupies a fixed-size slot.
3. `owner: Bytes<32>` — a single 32-byte slot, written once at construction and replaced (not appended) by `rotateOwner`.

There is no map, no vector, no append-only structure, no growable container anywhere in the contract. The total on-chain storage attributable to this contract is bounded by a small constant regardless of call rate or adversarial behavior.

**Authentication (owner-gated writes).** Both state-mutating circuits — `anchor` and `rotateOwner` — execute the same gate before any state change: `assert(disclose(publicKey(localSecretKey()) == owner), "unauthorized")`. No callable mutation exists without this check. This is the same `localSecretKey` witness + `persistentHash`-derived public-key pattern used in `gyotak-catch.md` (PR #96) and `nightforce-directory.md` (PR #79); it does **not** use `ownPublicKey()`.

Our contract combines both defenses. Unlike those contracts — whose state is a growable map keyed by user identity — our state is bounded-by-design (fixed single-slot fields, no growable container). Both properties hold simultaneously: writes are gated to a single rotatable operator key, and even unrestricted writes could not inflate the ledger.

This revision adopts the owner-gated pattern suggested in review, layered on top of the original bounded-by-design state model.

---

## 5. Evidence & verification

The V2 Score-1 rating rests on the bounded-by-design state model (§4), which is provable by inspection of the contract source and the compiled ledger layout in `managed/compiler/contract-info.json` (three fixed single-slot/counter fields, no growable container). It does not depend on chain-traffic volume.

**Deployment status.** V2 has not been deployed to any network. Mainnet will be its first and only deployment — there is no prior V2 contract, no migration, and no orphaned deployment. A predecessor V1 contract (no in-circuit authentication) was operated on Preprod from 2026-03-28 at `90d132a94a678f188a7371f626aa637224eaee8bee28ca66174ff1cfc21bee30` as the platform's pre-review baseline; it is retained as historical context only and is **not** offered as evidence for the V2 owner gate.

### 5.1 Owner-gate verification

The owner gate is verifiable without a chain deployment, by two independent means:

1. **Source inspection.** Both `anchor` and `rotateOwner` begin with `assert(disclose(publicKey(localSecretKey()) == owner), "unauthorized")`, and no state write precedes the assertion. The published source (`anchor.compact`) and compiled circuits (`managed/`) are sufficient to confirm this.
2. **Local SDK simulation (reproducible).** Because the contract source and compiled artefacts are public, a reviewer can reproduce the negative test locally: a call made under a non-owner secret key will fail the owner assertion during the wallet SDK's local circuit simulation — before any `/prove` request is issued and before any chain state is touched — the same simulation-layer rejection documented for the identical witness/`persistentHash` pattern in `gyotak-catch.md` (F-6 / F-7). The assertion would also fail at proof generation and at chain settlement, giving three independent enforcement layers.

We do not present V2 chain transactions because V2 has not been deployed, and the bounded-by-design Score-1 basis does not require chain evidence. The owner gate is substantiated by source inspection and locally-reproducible simulation as above.

### 5.2 Source & toolchain status

Unlike the V1 submission, the **Compact source has changed** for V2 (added: `owner` ledger field, `localSecretKey` witness, `publicKey` pure circuit, owner-gate assertions, `rotateOwner` circuit). The artefacts published under `managed/` are the V2 compile produced by `compactc 0.31.0` / language `0.23.0` / runtime `0.16.0`, and are the exact version intended for the (first) Mainnet deployment. The build is reproducible — see the SHA-256 fingerprints in §7.

---

## 6. Operational posture

**Operator secret key custody.** The operator secret key is supplied to the contract exclusively through the `localSecretKey` witness — a restricted-permission file/environment loader outside the platform's source repository that rejects world/group-readable files and malformed (non-64-hex) content, with no default-key fallback (a missing key throws explicitly). The secret never reaches the chain; only the derived `owner` public key (`publicKey(sk)`) is stored on-chain. Used key material is zeroed in memory after use (best-effort).

**Mainnet / Preprod isolation.** The Mainnet operator key, when provisioned, will be a fresh artefact independent of any Preprod or development key.

**Multi-tenant authentication boundary.** The operator key is the platform's identity; tenants are never granted operator credentials. Per-tenant scopes are enforced at the Gateway via `api_keys.scopes` (e.g., `app:tw-cbam`, `anchor`), with rate limiting via `CustomRateLimit` middleware. Only the DApp developer (the platform operator) holds and uses the API key issued by Midnight; tenants interact through the platform's M2M API and never directly with the chain.

**Recovery model.** The operator wallet seed is the single source of truth. All wallet sub-keys (shielded Zswap, unshielded NIGHT-external, DUST) are derived deterministically from the seed via HD derivation.

**Rotation.** The V2 contract includes a `rotateOwner(newOwner)` circuit, owner-gated identically to `anchor`. Operator key rotation therefore requires no redeploy and preserves the contract address — and its audit history — across the contract's lifetime. Each rotation is a distinct, owner-authorised on-chain transaction; the resulting `owner` is independently verifiable via a read-only indexer query of the on-chain `ledger.owner`.

---

## 7. Source & build reproducibility

- **Public Compact source**: https://github.com/midnightzk/anchor (Apache License 2.0)
- **Contract**: `anchor.compact` (40 lines, `pragma language_version >= 0.20`, forward-compatible)
- **Compactc**: 0.31.0
- **Compact language version**: 0.23.0
- **Compact runtime version**: 0.16.0
- **Circuits**: `publicKey` (pure, no proof key), `anchor` (proof), `rotateOwner` (proof); witness `localSecretKey`
- **Build**: `compact compile anchor.compact managed`; produces `managed/{contract,keys,zkir}/`

SDK / runtime semvers (production, from `origin/main` `package.json`):

| Package | Version |
|---|---|
| `@midnight-ntwrk/midnight-js-*` (contracts, types, utils, network-id, providers) | 4.0.4 |
| `@midnight-ntwrk/compact-runtime` | 0.15.0 |
| `@midnight-ntwrk/compact-js` | 2.5.0 |
| `@midnight-ntwrk/ledger-v8` | 8.0.3 |
| `@midnight-ntwrk/wallet` | 5.0.0 |
| `@midnight-ntwrk/wallet-sdk-dust-wallet` | 3.0.0 |
| `@midnight-ntwrk/wallet-sdk-facade` | 3.0.0 |
| `@midnight-ntwrk/wallet-sdk-address-format` | 3.1.1 |
| `@midnight-ntwrk/wallet-sdk-abstractions` | 2.0.0 |
| `@midnight-ntwrk/zswap` | 4.0.0 |

SHA-256 fingerprints of the compiled artefacts (`managed/`):

| File | SHA-256 |
|---|---|
| `keys/anchor.prover` | `84bd4cb4bc5038ec6b762eb5a172e7572a6454411924af960d2c976a0512f03b` |
| `keys/anchor.verifier` | `506f22d6313dacaf3f56570fd3b1f3a51e931dc983b4cbd1b4179237ced3163a` |
| `keys/rotateOwner.prover` | `d280d8511396e11adaced0ae101a03939055404cc75ed1261fcd2226c5a2e690` |
| `keys/rotateOwner.verifier` | `5735ad3427d8ed3493de6d82cf60886d98941f1c0e66e64f9dec80cb19678a23` |
| `zkir/anchor.zkir` | `14df58615b0cb089fd410a2a0052903cab39ee386e546222057164dbab8fa636` |
| `zkir/anchor.bzkir` | `b78405ba7cfb6bb99990fbbeaced3e63d68400661586f4a1374f6a283b4fcdf2` |
| `zkir/rotateOwner.zkir` | `d31caa016eabf42258ae500d680cb0400db9fd5f0379ac8012eca9b6b7e5ebb5` |
| `zkir/rotateOwner.bzkir` | `a9aca3a1356eb0d810430ff7505ef06a59aadc8292a4d199d0730a73b83c426b` |

**Reproducibility note**: artefacts above are the output of `compactc 0.31.0` / language `0.23.0`. Source `pragma >= 0.20` is forward-compatible with the current toolchain. Reviewers can reproduce the artefacts by running `compact compile anchor.compact managed` and comparing SHA-256 fingerprints. (`publicKey` is a pure circuit and therefore has no prover/verifier key.)

**Toolchain disclosure**: SDK semvers listed are from production (`origin/main`). The `managed/` artefacts are the V2 compile (`compactc 0.31.0`) and are the version intended for the first Mainnet deployment; V2 has not been deployed to any network (see §5).

---

## 8. Companion documents

- Compact source and managed artefacts: https://github.com/midnightzk/anchor (Apache License 2.0; published)
- Owner-gate verification is reproducible locally from the public source and compiled artefacts (see §5.1); no chain evidence is required for the bounded-by-design Score-1 basis.

---

## Notes

- Implementation deployment infrastructure (Gateway, Worker, Docker stack) is operated by the platform and is not part of this submission. The PR contains only the deployment request document and the on-chain Compact contract source.
- License: both this PR's file and the implementation repository are Apache License 2.0.
