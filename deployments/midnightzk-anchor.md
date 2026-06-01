# midnightzk-anchor — Mainnet Deployment Authorization Application

> **DRAFT v1.0 — V1 design (no in-circuit authentication), submitted for review.**
> The on-chain contract relies on bounded-state design (no growable storage) for state-space safety.
> Authentication is enforced at the platform's Gateway layer via per-tenant API keys.

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
| State-space-at-risk | 1 | Fixed-size on-chain state by design: a single 32-byte slot (`last_anchor_hash`, overwritten on each call) plus a counter. No map, no append-only structure, no growable storage. Adversarial calls cannot inflate the ledger. |

Detailed justification in §4. Empirical evidence in §5.

---

## 2. Contract at a glance

The contract source is `anchor.compact` (17 lines, `pragma language_version >= 0.20`, single file with no imports beyond `CompactStandardLibrary`).

**Ledger fields**:
- `last_anchor_hash: Bytes<32>` — single-slot storage for the most recent commitment hash
- `anchor_count: Counter` — total number of anchor calls processed

**Circuits**:
- `constructor()` — empty; no initialization arguments
- `anchor(fact_hash: Bytes<32>)` — overwrites `last_anchor_hash` with the supplied 32-byte commitment and increments the counter

**Witnesses**: none. `Witnesses<PS> = {}` is the empty record.

The contract design is intentionally minimal. On-chain state is a fixed-size auxiliary record; the substantive cryptographic proof is the on-chain transaction itself (immutable, indexable, queryable by `txId`), not the contract state.

---

## 3. Threat model

The adversary we consider is any party able to submit a transaction to Midnight (i.e., holding NIGHT/DUST sufficient to pay fees). The assets at risk are:

1. **Ledger state-space**: unbounded growth of contract storage.
2. **Customer proof integrity**: tenants' ability to prove they anchored a specific hash at a specific block.

For **ledger state-space**: the contract has no growable storage. Every call writes to the same single 32-byte slot and increments a fixed-size counter. Adversarial calls cannot inflate state-space; the worst an adversary can do is overwrite our slot or burn their own fees.

For **customer proof integrity**: the substantive proof is the on-chain transaction record, not the contract state. Each `anchor` call produces an immutable transaction containing the customer's hash, indexable by `txId` and verifiable by any third party against the indexer. An adversary overwriting `last_anchor_hash` cannot erase past transactions; tenants' proofs persist in chain history.

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

There is no map, no vector, no append-only structure, no growable container anywhere in the contract. The total on-chain storage attributable to this contract is bounded by a small constant regardless of call rate or adversarial behavior.

This justification differs from the "owner-gated writes" pattern in `gyotak-catch.md` (PR #96) and `nightforce-directory.md` (PR #79). Those contracts have growable state (maps keyed by user identity) and use in-circuit ownership checks to bound growth. Our contract takes the opposite approach: rather than gate writes, we eliminate growable state entirely, so even unrestricted writes cannot inflate the ledger.

We welcome the reviewer's feedback on whether the bounded-by-design interpretation is acceptable. If owner-gated writes are preferred as an additional defense layer, we have a Battleship-pattern implementation prepared and can revise on request.

---

## 5. Empirical evidence (Preprod, 2026-03-28 → 2026-05-16)

The V1 design has been deployed and exercised on Preprod since 2026-03-28. The contract address is `90d132a94a678f188a7371f626aa637224eaee8bee28ca66174ff1cfc21bee30`, deployed at 2026-03-28T13:51:38Z.

The empirical dataset comprises **237 successful anchor calls** across multiple tenants:

| Tenant | Count | Notes |
|---|---|---|
| Tenant A | 169 | Sustained production traffic across multiple months |
| Tenant B | 62 | Internal validation / health-check pipeline |
| Tenant C | 6 | Active production tenant |

Sample transactions (independently verifiable on the Preprod indexer by `txId`; block heights were not persisted at submission time — a known internal tech-debt item, tracked separately):

| Tenant | TX hash |
|---|---|
| Tenant A | `00b31de5ed8f354be402ed9f5905ba4b5db840dce43b9959b0be1be132403b515f` |
| Tenant A | `006e0635fa9d61af50a333a32151cda9e09ac702d800d7790c3cec3e26182748f2` |
| Tenant B | `0024590d855a94d5fb29ccde98809e59e5527b7e74e1daf3d0053691e3a47e5820` |
| Tenant B | `00f31a6e212a5b509a1bd7fe252c01d11966a040da54f28479ea66ea45871ac40e` |
| Tenant C | `0058203f0fe732bd8faf2fa8e49c93995824ca946925b3d3e6652633aa97da8861` |
| Tenant C | `008a7d499f4f51258456f94be041aefe6c8fa0d1ec40a9f36b972450dd931276ed` |

**Deploy transaction (E-1)**: `000779c1e63a86cb42ff57cc1ad507bb2145d40fa1a68a1c318e924fd107925a09` (2026-03-28T13:51:38Z).

### 5.1 Absence of certain evidence categories

We disclose what our dataset does **not** include, and why:

- **No chain-layer negative tests (F-6 / F-7 in the `gyotak-catch` template)**: our V1 contract has no in-circuit authentication, so unauthorized callers are not rejected at the chain layer. Authorization is enforced at the platform's Gateway layer via API key validation; rejected requests return HTTP 4xx and never reach the chain. Gateway-layer rejection logs available on request.
- **No key rotation evidence (F-3 / F-8)**: the V1 contract does not include a rotation circuit. The operator key is not rotated as part of contract operation.

### 5.2 Toolchain disclosure (currently deployed binary vs. submitted artefacts)

The currently deployed binary at `90d132a9...` was compiled on 2026-03-28 with an earlier Compact toolchain. In 2026-05 we performed a maintenance recompile (internal reference BLOCKER-D) with the current toolchain (`compactc 0.31.0` / language `0.23.0` / runtime `0.16.0`). **The Compact source code (`anchor.compact`) is unchanged**; only the compile toolchain has advanced.

The artefacts published in this repository (under `managed/`) reflect the current recompile and are the version we intend to deploy to Mainnet upon authorization. The Preprod evidence above demonstrates the runtime behaviour of the source code, which is identical between the two toolchain versions.

---

## 6. Operational posture

**Operator secret key custody.** The operator secret key is stored in a restricted-permission file outside the platform's source repository, loaded at runtime only by the worker process via a dedicated `WalletManager` loader. The loader rejects world/group-readable files and rejects malformed (non-64-hex) content; a missing key throws explicitly. Used key material is zeroed in memory after use (`zeroBuffer` best-effort).

**Mainnet / Preprod isolation.** The Mainnet operator key, when provisioned, will be a fresh artefact independent of any Preprod or development key.

**Multi-tenant authentication boundary.** The operator key is the platform's identity; tenants are never granted operator credentials. Per-tenant scopes are enforced at the Gateway via `api_keys.scopes` (e.g., `app:tw-cbam`, `anchor`), with rate limiting via `CustomRateLimit` middleware. Only the DApp developer (the platform operator) holds and uses the API key issued by Midnight; tenants interact through the platform's M2M API and never directly with the chain.

**Recovery model.** The operator wallet seed is the single source of truth. All wallet sub-keys (shielded Zswap, unshielded NIGHT-external, DUST) are derived deterministically from the seed via HD derivation.

**Rotation.** The V1 contract does not include a rotation circuit; operator key rotation is not currently supported in production. If the reviewer requires rotation capability for Mainnet approval, we will add the rotation circuit and re-submit.

---

## 7. Source & build reproducibility

- **Public Compact source**: https://github.com/midnightzk/anchor (Apache License 2.0)
- **Contract**: `anchor.compact` (17 lines, `pragma language_version >= 0.20`, forward-compatible)
- **Compactc**: 0.31.0
- **Compact language version**: 0.23.0
- **Compact runtime version**: 0.16.0
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
| `keys/anchor.prover` | `da4b5cf9fcf881f57794a9cae613f6be6738c9d3f123a6658e7083256a5d7cdc` |
| `keys/anchor.verifier` | `c40367848e711b610cce9d159b0fb43c327f0ada1e3079c269d81de9059fda59` |
| `zkir/anchor.zkir` | `0315c5732c44167d0b97a282b742a870ef8860534e799af36d4b06428a51c9de` |
| `zkir/anchor.bzkir` | `de7e9ef59d5f525b8bb0f8f1f6eb4663ea0aedc09a9b0b3baeeb43247be08dc9` |

**Reproducibility note**: artefacts above are the output of `compactc 0.31.0` / language `0.23.0`. Source `pragma >= 0.20` is forward-compatible with the current toolchain. Reviewers can reproduce the artefacts by running `compact compile anchor.compact managed` and comparing SHA-256 fingerprints.

**Toolchain disclosure**: SDK semvers listed are from production (`origin/main`). The `managed/` artefacts are from the BLOCKER-D toolchain refresh (2026-05) intended for Mainnet. The currently deployed Preprod binary was compiled with the earlier toolchain — see §5.2 for the source/binary correspondence.

---

## 8. Companion documents

- Compact source and managed artefacts: github.com/midnightzk/anchor (to be published alongside this PR)
- Preprod evidence archive: anchor CSV with 237 TX entries + block heights, to be published in the public repo

---

## Notes

- Implementation deployment infrastructure (Gateway, Worker, Docker stack) is operated by the platform and is not part of this submission. The PR contains only the deployment request document and the on-chain Compact contract source.
- License: both this PR's file and the implementation repository are Apache License 2.0.
