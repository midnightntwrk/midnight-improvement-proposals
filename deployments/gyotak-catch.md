# gyotak-catch — Mainnet Deployment Authorization Application

## 1. Summary

ECOSUS CO., LTD. is a Pranburi-based Thai company operating GYOTAK, a sashimi-grade flash-frozen fish brand serving both B2B and B2C channels. Our product differentiator is end-to-end traceability: every batch is published with provable origin metadata so customers and auditors can independently verify a shipment. The applicant is Takuya Ogura, Chairman, ECOSUS CO., LTD. The gyotak-catch contract publishes catch records — region, date, species, photo hash, GPS hash — onto Midnight as cryptographic commitments. We request Mainnet deployment authorization for the Score-1 design described below.

## 1.1 Risk rubric

| Category | Score | Summary |
| --- | --- | --- |
| Privacy-at-risk | 1 | Catch records' raw witness data (GPS coordinates, photo bytes) never leaves the operator's device; only their hashes reach the chain (see § 2 and § 6 for the witness model). The contract stores no user PII. |
| Value-at-risk | 1 | The contract holds no on-chain assets — neither tokens, NFTs, nor escrowed value. Adversarial misuse cannot result in user fund loss. The single state field at risk is the `batches: Map<Bytes<32>, CatchRecord>` map (see § 3 for full threat model). |
| State-space-at-risk | 1 | Owner-gated writes prevent unbounded ledger growth: an adversary's `recordCatch` attempt fails before any chain state change. Negative tests F-6 and F-7 demonstrate three independent enforcement layers (SDK simulation / proof generation / chain settlement). Detailed justification in § 4, with empirical evidence in § 5. |

## 2. Contract at a glance

The contract source is `contracts/gyotak-catch.compact` (71 lines, `pragma language_version >= 0.22`, single file with no imports beyond `CompactStandardLibrary`). It exposes two ledger fields and four circuits.

Ledger fields (lines 14–15):
- `batches: Map<Bytes<32>, CatchRecord>` — batchId-keyed catch records.
- `owner: Bytes<32>` — public key of the sole authorised writer.

Circuits:
- `constructor(initialOwner: Bytes<32>)` — sets `owner` once at deploy (lines 24–26).
- `recordCatch(...)` — owner-gated, ±600 s block-time bounded (lines 28–60).
- `rotateOwner(newOwner: Bytes<32>)` — owner-gated; replaces `owner` atomically (lines 62–66).
- `verifyCatch(batchId)` — read-only public lookup (lines 68–71).

Two supporting components round out the design. The pure helper `publicKey(sk)` (line 20) derives a 32-byte public key from a secret key via `persistentHash<Vector<2, Bytes<32>>>([pad(32, "gyotak:pk:"), sk])`. The witness `localSecretKey()` (line 18) is implemented in `src/witnesses.ts` as a file/env loader with no default-key fallback; a missing or malformed key throws explicitly rather than silently authenticating with a stand-in.

## 3. Threat model

The adversary we consider is any party holding NIGHT for fees and able to reach a Midnight proof server. The asset at risk is ledger state space: specifically, unbounded growth of the `batches` map, which every successful insert extends permanently.

The predecessor Score-3 contract (deployed for evaluation on Preprod, see Section 8) gated `recordCatch` only by a duplicate-batchId check. Any caller could insert arbitrary 32-byte keys for the cost of DUST, leaving the ledger open to spam-driven storage inflation.

The Score-1 design closes this gap with a single authentication assertion. Every state-mutating circuit asserts `publicKey(localSecretKey()) == owner` before any `disclose` writes (lines 37 and 64), gating all writes to a single rotatable key.

## 4. State-space-at-risk Score 1 justification

The Score-1 design satisfies the four criteria below.

**(a) Authentication.** Both write circuits — `recordCatch` and `rotateOwner` — execute the same gate before any state change: `assert(disclose(publicKey(localSecretKey()) == owner), "unauthorized")` (lines 37 and 64). No callable mutation exists without this check.

**(b) Bounded growth.** Because writes are gated by ownership, the `batches` map grows at the rate of one legitimate operator's submissions, not at adversarial rate. An adversary's `recordCatch` attempt fails before any chain state change; Section 5 documents two such attempts on Preprod.

**(c) Time binding.** `recordCatch` enforces a ±600-second window between the caller-supplied timestamp and the producing block's clock via `blockTimeGte(t - 600)` and `blockTimeLte(t + 600)` (lines 45–46). The window prevents back-dated or future-dated commitments outside a 10-minute tolerance.

**(d) Rotation.** `rotateOwner(newOwner)` allows the admin key to change without redeploying. The contract address is preserved across rotations, which keeps the audit history attached to a single address over the contract's lifetime.

## 5. Empirical evidence (Preprod, 2026-05-08)

The Score-1 design has been deployed and exercised on Preprod. The contract address is `53b8303fc72a83abd3d26e5372102a58cc9be55c42e383695b46a0f2d33e285f`. The predecessor Score-3 contract, retained as the historical baseline, lives at `a3f3a04476914b86eb914a4e3626519b1538395901fd376442a1fa8afffe836f` (deployed 2026-05-07, archived 2026-05-08; documented in `ECOSUS-TR-2026-002.html`).

The rehearsal followed `docs/MAINNET-DEPLOY-CHECKLIST.md` Sections A through F. Each row in the table below is independently verifiable on the Preprod indexer.

| Step | Outcome | TX / Block | What it proves |
|---|---|---|---|
| E-1 | Score-1 contract deployed | TX `0056...62cb` / block 691,960 | Constructor wrote `owner = pk-A` |
| E-3 | Indexer query: `ledger.owner == pk-A` | byte-match | Constructor effect persisted on-chain |
| F-3 | `rotateOwner(pk-A → pk-B)` | TX `0078...43e8` / block 692,122 | Forward rotation succeeds for legitimate owner |
| F-4 | Indexer query: `ledger.owner == pk-B` | byte-match | Rotation effect persisted on-chain |
| F-5 (POSITIVE) | `recordCatch` under sk-B | TX `0010...b1c` / block 692,180 | Rotated owner can call write circuits |
| F-6 (NEGATIVE) | `recordCatch` under sk-X | rejected at SDK simulation, no `/prove` issued | Non-owner write rejected before any chain effect |
| F-7 (NEGATIVE) | `rotateOwner` under sk-X-2 | rejected at SDK simulation, owner unchanged | Non-owner rotation rejected; sentinel target blocked |
| F-8 | `rotateOwner(pk-B → pk-A)` (rollback) | TX `00a9...1ccc0` / block 692,289 | Reverse rotation closes the cycle |

**Note on transaction identifiers.** The TX values shown above are transaction identifiers (the Midnight SDK's `txId` field, which corresponds to the indexer's `transactions(offset: { identifier: ... })` query parameter). The indexer also exposes a separate cryptographic hash field for each transaction; reviewers querying the Preprod indexer should use the `identifier` field with the values above. Querying the `hash` field with the same values will return "invalid transaction hash" because the SDK does not surface the on-chain cryptographic hash directly to clients.

Full hex of the chain-write transactions, for direct indexer verification:

- **E-1 TX (deploy)**: `00563f8f726db76ace444451306e0671d2fb6f97558ec991da0e7a6360b28362cb` — block hash `69c85ee06f164b57c7ec588576ba90f9534430c1b8d2a70eadd4b3263941158e`
- **F-3 TX**: `0078a82724af98fa0fda659fe96b0f94e6c16ee03588a295645212974ed93243e8` — block hash `333d52cc606401f74b50cc5c221fd144688a4fb0b49a91cea1d7c782039f44d4`
- **F-5 TX**: `0010d98f43ddf032172146d69376194296da2ffce7f098d8d6d68e25745ea99b1c` — block hash `0325fea2661edd78cacf70baa9519969ddeca568c6583c53f28005eecffb0ba6`
- **F-8 TX**: `00a9fa5888581bd8f2d7dd492db9442c28cf3c85f0beb0c9fc9766a4c0a301ccc0` — block hash `59b31795022a3c1fd2b88195a4915f6fa1c4f6448e78638575414d0bad613a47`

The two negative tests (F-6 and F-7) deserve emphasis. In both cases, the wallet SDK's local circuit simulation (`midnight-js-contracts/dist/index.mjs`, `scoped` helper) caught the assertion failure and aborted the transaction before any `/prove` request was issued to the proof server, and before any chain state was touched. The same assertion would also fail at proof generation and at chain settlement, giving three independent enforcement layers for any single attempt to bypass the owner gate.

### 5.1 On-chain transaction hashes (verified via Preprod indexer)

For reviewers preferring `hash`-based lookup, the on-chain cryptographic hashes for the chain-write transactions, retrieved by querying the Preprod indexer with the `identifier` field above:

| Step | Identifier (SDK txId) | On-chain hash |
|---|---|---|
| E-1 | `0056...62cb` | `fa29addcb33007f95fbf364be6c7e7c59860a8a9f17a7e293dd9b4775dbaedf7` |
| F-3 | `0078...43e8` | `83beeb8c61133418991727b6e4d43f56cc8b35ead11f385f220524d6346de2ce` |
| F-5 | `0010...b1c`  | `3058d63e9565303f22a64ba3b4511e70dfaab4e67db38f0f5f541d8ca8390a41` |
| F-8 | `00a9...1ccc0` | `b60f9eba1f7a46d52392f0693582030906331d425bd251a6c3401f41affdf139` |

Verified via indexer query at `https://indexer.preprod.midnight.network` on 2026-05-09, during application preparation.

Each transaction's `identifiers` array on the indexer contains two equivalent lookup handles; either element resolves to the same on-chain transaction. The Midnight SDK's `txId` field corresponds to the second array element, a convention verified across F-3, F-5, and F-8 (where the SDK explicitly logged the txId).

For E-1, the deploy CLI did not surface the SDK's txId in its logs, so the value listed above (`0056...62cb`) is the second `identifiers` element retrieved from the indexer at block 691,960. Reviewers can confirm this resolves to the Score-1 deploy transaction by querying either `identifiers` element with the indexer's `transactions(offset: { identifier: ... })` field.

## 6. Operational posture

**Admin secret key custody.** The canonical sk lives at `~/midnight/.gyotak-secrets/admin-sk.txt`, mode 600, outside the repository. The loader (`loadAdminSecretKey` in `src/witnesses.ts`) refuses world- or group-readable files and rejects malformed (non-64-hex) content; a missing key throws explicitly rather than silently authenticating with a stand-in.

- A physical paper backup is retained at a separate site; `/mnt/c` and cloud-sync paths are explicitly forbidden.
- The Mainnet sk will be a fresh artefact independent of the Preprod sk; backups for the two are physically separated.

**Rotation procedure.** Owner key rotation runs via `npm run preprod -- rotate-owner --new-owner <hex>` (or the analogous `:mainnet` script after Mainnet authorization). The same code path was exercised in F-3 (forward) and F-8 (rollback) on Preprod, and both transactions are independently verifiable on the indexer.

- Each rotation produces a unique chain transaction, eliminating replay risk.
- Post-rotation, the new owner is independently verifiable via `scripts/verify-owner.ts`, a read-only indexer query that returns the on-chain `ledger.owner` byte-for-byte.

**Redeploy strategy.** Key events do not require redeploy — `rotateOwner` covers them, and the contract address remains stable across the contract's operational lifetime. Redeploy is reserved for Compact source changes (logic upgrades, additional circuits); each redeploy generates a new contract address that the operator publishes alongside any address-change announcement.

**Recovery model.** `WALLET_SEED` is the single source of truth. All wallet sub-keys (shielded, unshielded, dust) are derived deterministically from it via HD derivation, and the contract's private state is `salt: 32 zero bytes` with no real entropy, so loss of the LevelDB private state provider is fully recoverable from the seed alone. To eliminate single-point-of-failure on the seed, the operator commits to dual-channel backups (paper + hardware) before any Mainnet deployment.

**Deployment ritual.** `docs/MAINNET-DEPLOY-CHECKLIST.md` Sections A through G cover key generation, derivation determinism, pre-deploy validation, deploy with on-chain owner verification, the rotation rehearsal (positive plus two negative tests), and the Mainnet pre-deployment gate. The Phase 2b execution described in Section 5 above followed the checklist verbatim and produced `preprod-evidence-2026-05-08/` as a complete log archive.

- An appendix specifies airgap-machine key generation for the Mainnet sk. Preprod was deliberately run on the regular WSL2 host because its purpose was evidence collection, not production custody.

## 7. Source & build reproducibility

The contract source and toolchain are pinned to the versions below. Reviewers can reproduce the compiled artefacts deterministically by running `npm run build:contract` and comparing the SHA-256 fingerprints listed at the end of this section.

- **Repository**: https://github.com/ecosus-co/gyotak-catch
- **Compact source**: `contracts/gyotak-catch.compact`, `pragma language_version >= 0.22`
- **Compactc**: 0.30.0 (recorded in `contracts/managed/compiler/contract-info.json`)
- **Compact language version**: 0.22.0
- **Compact runtime version**: 0.15.0
- **Build**: `npm run build:contract` = `compact compile contracts/gyotak-catch.compact contracts/managed`; produces `contracts/managed/{contract,keys,zkir}/`.

SDK / runtime semvers (from `package.json`):

- `@midnight-ntwrk/compact-runtime` ^0.15.0
- `@midnight-ntwrk/ledger-v8` ^8.0.3
- `@midnight-ntwrk/midnight-js` ^4.0.4 (and `midnight-js-contracts`, `midnight-js-types`, the four provider packages, and `midnight-js-network-id` at the same version)
- `@midnight-ntwrk/wallet-sdk-facade` ^3.0.0 (with `wallet-sdk-shielded`, `wallet-sdk-unshielded-wallet`, and `wallet-sdk-dust-wallet`, all ^3.0.0)
- `@midnight-ntwrk/wallet-sdk-hd` ^3.0.1, `wallet-sdk-capabilities` ^3.2.0
- `@midnight-ntwrk/wallet-sdk-abstractions` ^2.1.0, `wallet-sdk-address-format` ^3.1.0, `wallet-sdk-indexer-client` ^1.2.1

SHA-256 fingerprints of the compiled artefacts (`contracts/managed/`):

| File | SHA-256 |
|---|---|
| `keys/recordCatch.prover` | `473a752c5c139ba4ed3c0271ad908496e974043278b96c2c41b5ee46b90db8b1` |
| `keys/recordCatch.verifier` | `3b929ac15c6d2dfabe584b9ac42e36377045a16672f9ac5a861a5312ebb51ee8` |
| `keys/rotateOwner.prover` | `ff4dcd6df8bc420592eb58ff5ae6e4a4a4bfbbb45a387aad1967242be7ef9e0c` |
| `keys/rotateOwner.verifier` | `7d06222e211911d3ef2147cc982f13d90bb15981cc16823b6d5d340bb905632f` |
| `keys/verifyCatch.prover` | `1f952b514eb3e7494c08db9a8ad8f53d726b08a386b377b89c5bdaf4b9928087` |
| `keys/verifyCatch.verifier` | `dd9f8158e06e1407553e226f60a889e6c23e7ceeacea659ec4f7d5bf53a26959` |
| `zkir/recordCatch.zkir` | `184c62435d56c54bc100079eb48d3eed7422e4db83c696b355fa1f9620916b7c` |
| `zkir/recordCatch.bzkir` | `a1e485854f49e95b1e0dcff9b9c25a6ce0b277c49d282c52b6b19f1e1ff95872` |
| `zkir/rotateOwner.zkir` | `112835b845811f9b87c3970a2f55f7d3c51b77646f463ec725e5ea0d3503d7a0` |
| `zkir/rotateOwner.bzkir` | `04747151c77eb8a9ac3ba925b72701f9c4e87e0b097c15b60f2a9ab46b355b18` |
| `zkir/verifyCatch.zkir` | `569a4250a336bf6f26d46428915654f61375507d8976b66d2cb82f554c752175` |
| `zkir/verifyCatch.bzkir` | `09fda0a4b3ebe56f784f3b840bb159dc347d28e2026e9e8ea2969a98531ae248` |

## 8. Companion documents
- `docs/MAINNET-DEPLOY-CHECKLIST.md` — full operational procedure (English).
- `docs/PREPROD_MIGRATION_DAY1_2026-05-05.md` — Day 1 of the Preprod migration (Score-3 era infrastructure).
- `docs/PREPROD_MIGRATION_DAY2_2026-05-07.md` — Day 2 of the Preprod migration (LevelDB persistence + Score-3 deploy).
- [ECOSUS-TR-2026-002.html](https://perma.cc/X36J-KVGM) — predecessor technical report (Score-3 design under Preprod, retained as the historical baseline). The perma.cc archive is captured 2026-05-08; live source: https://pub-fbde8afd2bac4908adc357514f2a2271.r2.dev/ECOSUS-TR-2026-002.html
- `preprod-evidence-2026-05-08/` — raw log archive (A1.log through F10.log, plus gitleaks scan and 64-hex audit), the source material behind every claim in this application.
