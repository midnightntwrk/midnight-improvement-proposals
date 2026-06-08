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

---

# gyotak-catch v2 — Mainnet Deployment Authorization Application (Revision)

## 0. What changed since v1 (PR #96)

The v1 Score-1 contract (`contracts/gyotak-catch.compact`, Mainnet address
`e88a2a00b1d78ba1f2891c7017a4be8df2139da02328078fa4fc248b72fea2f4`) records one
catch per `batchId` with a single `fishSpecies: Bytes<32>` plaintext field. v2
adds a **fixed-size plaintext fish manifest** so that a single catch report
covering multiple species can publish each species *and its weight* on-chain.

The change is **additive and schema-bounded**:

- New ledger field on `CatchRecord`: `fishManifest: Vector<10, Bytes<32>>` — up to
  10 slots, each a UTF-8 `"romaji:weight"` string (e.g. `"Koro-Dai:20.8"`),
  zero-padded to 32 bytes; unused slots are all-zero. (10 slots leaves headroom
  over the observed maximum of 5 species per catch report.)
- `recordCatch` gains one argument (`fishManifest`) and discloses it into the
  record. No new witness, no new authority, no asset handling.
- All v1 invariants are retained verbatim: owner-gated writes, ±600 s block-time
  binding, duplicate-batchId rejection, GPS-as-witness (only `gpsHash` reaches
  the ledger). `fishSpecies` is retained for backward compatibility (existing
  readers, e.g. the invoice/catch-record link, keep working unchanged).

Because Compact has no in-place schema migration, v2 is a **new contract at a new
address**; v1 records remain at the v1 address. The operator publishes the v2
address alongside an address-change announcement (same procedure as any redeploy
documented in v1 §6 "Redeploy strategy").

## 1. Summary

ECOSUS CO., LTD. is a Pranburi-based Thai company operating GYOTAK, a
sashimi-grade flash-frozen fish brand serving both B2B and B2C channels. Our
product differentiator is end-to-end traceability: every batch is published with
provable origin metadata so customers and auditors can independently verify a
shipment. The applicant is Takuya Ogura, Chairman, ECOSUS CO., LTD.

v2 extends the published metadata from a single species label to a per-catch
manifest of up to ten `species:weight` entries, in cleartext, alongside the
unchanged GPS-hash / photo-hash commitments. We request Mainnet deployment
authorization for the v2 design described below.

### 1.1 Risk rubric (re-scored for v2)

| Category | v1 | v2 | Summary |
| --- | --- | --- | --- |
| Privacy-at-risk | 1 | **1** | The added data (species name + weight) is already public: it is printed on customer invoices, package labels, and the public catch blog. It is Tier-1 "data intended / already public" with no real-world harm and no PII. The only witness-held datum remains GPS coordinates, of which only `gpsHash` is disclosed — identical to v1. |
| Value-at-risk | 1 | **1** | The contract holds no on-chain assets — no tokens, NFTs, or escrowed value. The new field stores bytes, not value. Owner-gated writes are unchanged; an exploit cannot cause user fund loss. |
| State-space-at-risk | 1 | **1** | The manifest is a **compile-time fixed-size array** (`Vector<10, Bytes<32>>` = a hard 320-byte ceiling per record). Writes remain owner-gated and one-record-per-`batchId`, so growth tracks one legitimate operator's catch reports, not adversarial input. The per-record size increases by ≤320 bytes with a hard ceiling and no per-user unboundedness — squarely Tier-1 "bounded and static state, contract-enforced maximum". |

No category scores 3; the Launch Phased Deployment Override is not triggered.

## 2. Contract at a glance (v2)

Source: `contracts/gyotak-catch.compact` (v2), `pragma language_version >= 0.22`,
single file, no imports beyond `CompactStandardLibrary`.

Ledger record (the single added line is marked `// NEW`):

```compact
struct CatchRecord {
  gpsHash:      Bytes<32>,
  photoHash:    Bytes<32>,
  region:       Bytes<32>,
  catchDate:    Bytes<32>,
  fishSpecies:  Bytes<32>,             // retained: headline / primary species
  fishManifest: Vector<10, Bytes<32>>, // NEW: up to 10 "romaji:weight" plaintext slots
  committedAt:  Uint<64>,
}

export ledger batches: Map<Bytes<32>, CatchRecord>;
export ledger owner:   Bytes<32>;
```

Circuits (unchanged except `recordCatch`'s extra disclosed argument):

- `constructor(initialOwner)` — sets `owner` once at deploy.
- `recordCatch(batchId, region, catchDate, fishSpecies, fishManifest, photoHash, timestamp)`
  — owner-gated, ±600 s block-time bounded; discloses `fishManifest` into the record.
- `rotateOwner(newOwner)` — owner-gated; replaces `owner` atomically. **Unchanged**
  (circuit byte-identical to v1, see §7).
- `verifyCatch(batchId)` — read-only public lookup; now returns the manifest too.

The owner gate is byte-for-byte the v1 assertion:
`assert(disclose(publicKey(localSecretKey()) == owner), "unauthorized")`. The
witness model is unchanged: `getGpsCoords(batchId)` supplies GPS off-chain, only
`persistentHash(gps)` is stored; `localSecretKey()` loads the admin sk from a
mode-600 file with no default-key fallback.

## 3. Threat model

The adversary is any party holding NIGHT for fees and able to reach a proof
server. The asset at risk is ledger state space — growth of the `batches` map and
now the per-record manifest bytes.

v2 does not widen the attack surface relative to v1: the single owner-gate
assertion still precedes every `disclose` write, so an adversary cannot insert a
record at all, let alone an oversized one. The manifest is a fixed 10-slot vector,
so even the legitimate owner cannot exceed 320 manifest bytes per record. There
is no caller-controlled length, no loop, and no per-user accumulation.

## 4. State-space-at-risk Score 1 justification (v2)

**(a) Authentication.** Unchanged from v1: both write circuits assert
`publicKey(localSecretKey()) == owner` before any state change. Empirically
confirmed by F-6 below (non-owner write rejected at SDK simulation).

**(b) Bounded growth — now with a hard per-record ceiling.** The manifest is
`Vector<10, Bytes<32>>`, a compile-time fixed-size array. The maximum record size
is therefore a constant (5×`Bytes<32>` + `Uint<64>` + 10×`Bytes<32>`), independent
of input. No circuit path can grow a single record beyond this ceiling.

**(c) Time binding.** Unchanged: `recordCatch` enforces a ±600 s window via
`blockTimeGte(t-600)` / `blockTimeLte(t+600)`.

**(d) Rotation.** Unchanged: `rotateOwner` rotates the admin key without redeploy;
the v2 address is stable across rotations.

**Why plaintext disclosure is intentional and safe (justification requested by
reviewers).** Traceability is the product itself: GYOTAK's value proposition is
that any customer, auditor, or autonomous agent can verify a shipment's origin
*without trusting ECOSUS's own database*. Species and weight are already public —
they appear on the invoice, the package label, and the public catch blog — so
publishing them on-chain leaks nothing new; it merely makes the already-public
claim tamper-evident and independently checkable. Concretely, the GYOTAK Fish
Market MCP agent and end consumers resolve a package's `batchId` to the on-chain
`CatchRecord` and confirm species/weight against the physical label. Commercially
sensitive data (exact fishing coordinates) stays witness-side: only `gpsHash`
reaches the chain, exactly as in v1.

## 5. Empirical evidence (Preprod, 2026-06-04)

The v2 design was deployed and exercised on Preprod. The v2 contract address is
`374df4da76f1fde6b0d73d94d0bba01f90f7194e8b04d57320d2ba6b37cfecd5`; the owner
public key (pk-A) is
`5e8c984df5acc1948c6a9b969e92f6d1c237257a8cc0e4b629335ec6818dcc99`, derived from
the admin secret key via the same `publicKey(sk)` pure circuit as the contract.
Each row below is independently verifiable on the Preprod indexer.

| Step | Outcome | TX / Block | What it proves |
|---|---|---|---|
| E-1 | v2 contract deployed on Preprod | addr `374df4da…` (constructor) | Constructor wrote `owner = pk-A` |
| E-3 | Indexer query: `ledger.owner == pk-A` | byte-match (`verify-owner` → `OK: match`) | Constructor effect persisted on-chain |
| F-5 (POS) | `recordCatch` with a 5-slot manifest + 5 zero-pad | TX `00fbbe6e…122654` / block **1068810** | Multi-species write; all 10 slots round-trip **byte-identical** |
| F-9 (POS) | `recordCatch` with a **full 10-slot** manifest (decimal weights) | TX `00ecd76c…988cd2` / block **1068815** | Full-capacity write; all 10 slots round-trip **byte-identical** |
| F-6 (NEG) | `recordCatch` under non-owner sk-X | rejected, no chain effect | `failed assert: unauthorized` at SDK simulation; **no `/prove`, no submit, no dust consumed** |

**F-5 read-back (input → on-chain, byte-identical):**

```
[0] "Koro-Dai:20.8"     -> "Koro-Dai:20.8"      ✓
[1] "Kuro-Kanpachi:10"  -> "Kuro-Kanpachi:10"   ✓
[2] "Oni-Aji:1.5"       -> "Oni-Aji:1.5"        ✓
[3] "Umadsura-Aji:11"   -> "Umadsura-Aji:11"    ✓
[4] "Maguro:5"          -> "Maguro:5"           ✓
[5..9] <zero>           -> <zero>               ✓   ← subsumes F-10 (zero-padding round-trips losslessly)
```

**F-9 read-back (full 10 slots, decimal weights kept verbatim):**

```
[0] "Sp0:0.5" -> "Sp0:0.5"  ✓     [5] "Sp5:5.5" -> "Sp5:5.5"  ✓
[1] "Sp1:1.5" -> "Sp1:1.5"  ✓     [6] "Sp6:6.5" -> "Sp6:6.5"  ✓
[2] "Sp2:2.5" -> "Sp2:2.5"  ✓     [7] "Sp7:7.5" -> "Sp7:7.5"  ✓
[3] "Sp3:3.5" -> "Sp3:3.5"  ✓     [8] "Sp8:8.5" -> "Sp8:8.5"  ✓
[4] "Sp4:4.5" -> "Sp4:4.5"  ✓     [9] "Sp9:9.5" -> "Sp9:9.5"  ✓   ← subsumes F-11 (weights are not rounded)
```

**F-6 (owner gate).** With the witness `localSecretKey()` overridden to a
throwaway non-owner key `sk-X`, `recordCatch` is rejected by the wallet SDK's
local circuit simulation with `Error: failed assert: unauthorized` — before any
`/prove` request is issued and before any transaction is submitted, so no dust is
spent and `ledger.owner` is untouched. The same assertion would also fail at
proof generation and at chain settlement, giving three independent enforcement
layers (identical to v1 F-6/F-7).

**F-3 / F-8 (owner rotation) — re-run omitted; circuit proven byte-identical.**
The `rotateOwner` circuit is **unchanged** in v2: its compiled prover, verifier,
and ZKIR artefacts are byte-for-byte identical to v1 (see §7 — `rotateOwner.prover`
`ff4dcd6d…`, `rotateOwner.verifier` `7d06222e…`, `rotateOwner.zkir` `112835b8…`,
`rotateOwner.bzkir` `04747151…`, all matching the v1 fingerprints in
`deployments/gyotak-catch.md` §7). Because the rotation logic did not change, the
v1 F-3 (forward) / F-8 (rollback) evidence on Preprod carries over unchanged; a
fresh on-chain re-run would add no information. (A live re-run was additionally
gated by transient Preprod dust exhaustion — see the note below — but byte-identity
is the dispositive argument.)

**Note on `Custom error 170` during the rehearsal.** After F-5 and F-9, a third
sequential `recordCatch` in the same wallet session was rejected at submit with
`1010 Invalid Transaction: Custom error: 170`, and the wallet then showed
`dust.coins = 0`. This is **preprod fee-dust exhaustion** (the test wallet's
single ~5-DUST coin was consumed by the two successful records), **not** a
contract, schema, or SDK-version defect: the heavier 10-slot F-9 transaction
settled successfully on the same contract immediately before. On Mainnet the
deploy wallet holds ≈ 382,307 DUST (verified read-only on the Mainnet indexer),
so fee dust is not a constraint there.

Preprod v2 address: `374df4da76f1fde6b0d73d94d0bba01f90f7194e8b04d57320d2ba6b37cfecd5`.
Mainnet v2 address: `5f87dd3078d5938ae83a54161e549ee9f3e1a6ebd7640a527f0d674fb43741f7` (deployed 2026-06-04, block 1106629, tx `00defd999937a9ef9164b4710a6236f43efef3e4ae3b042e7e91b43c4d9645cd27`, owner == pk-A verified on the Mainnet indexer).

## 6. Operational posture

Unchanged from v1 except the contract schema. Admin sk custody, rotation
procedure, recovery model (seed = single source of truth, HD-derived sub-keys),
and the `docs/MAINNET-DEPLOY-CHECKLIST.md` ritual all carry over verbatim. The v2
Mainnet deploy reuses the v1 Mainnet wallet and the credentialed Foundation RPC
endpoint (`rpc.mainnet.midnight.foundation`) established during the v1 1016
remediation; that credential was re-verified live (read-only) and remains valid.
The v2 contract-address file is kept distinct (`.contract-address.mainnet.v2`) so
the live v1 deployment and its invoice link are untouched during cutover.

## 7. Source & build reproducibility

- **Repository**: https://github.com/ecosus-co/gyotak-catch (v2 tag `v2.0.0-fish-manifest`, commit `9df983f`)
- **Compact source**: `contracts/gyotak-catch.compact` (v2), `language_version >= 0.22`
- **Compactc**: 0.30.0 · **language**: 0.22.0 · **runtime**: 0.15.0
  (recorded in `contracts/managed/compiler/contract-info.json`)
- **Build / reproduce**: `git checkout v2.0.0-fish-manifest && npm run build:contract`
  (= `compact compile contracts/gyotak-catch.compact contracts/managed`), then
  compare the SHA-256 fingerprints below against `contracts/managed/{keys,zkir}/*`.
- SDK / runtime semvers (unchanged from v1): `@midnight-ntwrk/ledger-v8` ^8.0.3,
  `@midnight-ntwrk/compact-runtime` ^0.15.0, `@midnight-ntwrk/midnight-js` ^4.0.4
  (and the provider packages at the same version). The added `fishManifest` field
  **compiles and submits cleanly under this toolchain**: it was recompiled and the
  v2 `recordCatch` settled on Preprod (F-5, F-9) on `ledger-v8 8.0.3`. Mainnet
  acceptance under the current Mainnet runtime (node `0.22.5-31b06338`) will be
  re-confirmed at deploy time.

SHA-256 fingerprints of the compiled v2 artefacts (`contracts/managed/`). The
`recordCatch` and `verifyCatch` circuits changed (the record struct gained a
field); `rotateOwner` is **byte-identical to v1**:

| File | SHA-256 | vs v1 |
|---|---|---|
| `keys/recordCatch.prover`   | `4aff8a57d8cdf2af094df3be4058105d2f8864d0cada71fcedd16af80ae71fd5` | changed |
| `keys/recordCatch.verifier` | `022446af5807440cf2ffe62df54401dc1613e4fccc0cacb6ade67801904f71fb` | changed |
| `keys/rotateOwner.prover`   | `ff4dcd6df8bc420592eb58ff5ae6e4a4a4bfbbb45a387aad1967242be7ef9e0c` | **identical** |
| `keys/rotateOwner.verifier` | `7d06222e211911d3ef2147cc982f13d90bb15981cc16823b6d5d340bb905632f` | **identical** |
| `keys/verifyCatch.prover`   | `7b808997438e4be74a07c016c8205f2873eb867ca84f2b885ceeba31c45c25c2` | changed |
| `keys/verifyCatch.verifier` | `9df90b684d3cf1c63b39233cb99280464b8c75c38bf4da241f309bdf645fb082` | changed |
| `zkir/recordCatch.zkir`     | `12862aee37ff22abea0085e5e563d414f7e1dd9b6c47ce8cc28abc3c2b002bdd` | changed |
| `zkir/recordCatch.bzkir`    | `ec8b1dac746840873d98f2f96896502b186ffe18a1705f1a94defb8ad920d598` | changed |
| `zkir/rotateOwner.zkir`     | `112835b845811f9b87c3970a2f55f7d3c51b77646f463ec725e5ea0d3503d7a0` | **identical** |
| `zkir/rotateOwner.bzkir`    | `04747151c77eb8a9ac3ba925b72701f9c4e87e0b097c15b60f2a9ab46b355b18` | **identical** |
| `zkir/verifyCatch.zkir`     | `7ecf5b9ebd1d6b3f2d9cd5ccfb7de62bb307509bb98829be3553c6759ff116f6` | changed |
| `zkir/verifyCatch.bzkir`    | `89a6d48c98dc47f4f4a96d32df77b7d7b8abc1d381cbee62b0d5224d44155506` | changed |

The `recordCatch` prover grew from 2,826,769 bytes (v1) to 5,203,391 bytes (v2),
reflecting the larger fixed-size manifest circuit. The four byte-identical
`rotateOwner` fingerprints are the basis for omitting a fresh F-3/F-8 re-run (§5).

## 8. Companion documents

- `docs/MAINNET-DEPLOY-CHECKLIST.md` — operational procedure (carries over).
- v1 application: `deployments/gyotak-catch.md` (PR #96) — this revision supersedes
  the contract-design sections while retaining v1 as historical baseline.
- v2 Preprod evidence is independently verifiable on the Preprod indexer via the
  contract address and tx hashes/blocks in §5: deploy + `ledger.owner == pk-A`
  (E-1/E-3), `recordCatch` F-5 (block 1068810) and F-9 (block 1068815) with their
  byte-identical read-backs, and the F-6 owner-gate rejection (no settled tx).
