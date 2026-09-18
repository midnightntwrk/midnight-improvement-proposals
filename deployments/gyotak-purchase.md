# Deployment Request: gyotak-purchase v3

## 1. Summary

`gyotak-purchase` records purchase commitments for flash-frozen seafood, and is the
third contract in the GYOTAK traceability system. `gyotak-catch` (PR #96) and
`gyotak-temp-log` (PR #224) cover catch provenance and cold-chain evidence; this one
closes the chain at the buyer's end.

This request covers **v3**, a new contract address. v2 was authorized through PR #316
and deployed on 2026-09-18; v3 changes one thing, and the reason is worth stating
plainly because it is the substance of this request.

A buyer who wants to speak publicly about a purchase attaches a record to the
`bindings` map. In v2 that record named the account they would post from — a handle
they told us, which we never verified. Anyone can claim a handle that is not theirs,
and the operator cannot detect it; v2's own request document named this as the sharpest
limit of the design.

v3 adds a second field to that record: the buyer's referral id. That value is different
in kind. The operator issued it to that customer when their payment was confirmed, and
it reaches them only inside their personal connection URL. So the two together do what
neither does alone. The referral id evidences that the claimant is the buyer; the handle
says where they will post; and a verifier checks both against the same binding. A
referral id appears in the post itself, so anyone reading a genuine post can copy it into
a fabricated one — the handle recorded alongside it is what makes that fail.

Ledger state:

- **`purchases`** — one record per purchase: `purchaseCommitment`
  (`persistentCommit<Bytes<32>>(buyerId32, nonce)`), `lotId`, `committedAt` and `schema`.
  The buyer identifier and the opening nonce enter as witnesses and never reach the chain.
  Unchanged from v2.
- **`bindings`** — the subset of purchases a buyer has claimed, holding `handle`,
  **`referrerId`**, `boundAt` and `schema`. Both identifiers are stored in plaintext.
- **`owner`** — the single authorised writer's public key.

The commitment is deliberately the plain `SHA-256(nonce ‖ buyerId32)`, which we verified
empirically against an externally computed digest (§ 5.1); a buyer can open their own
record with one hash, with no Midnight tooling and no cooperation from us.

A full technical report is published as a defensive publication: TR-2026-015
(https://gyotak-tr.pages.dev/tr-2026-015/, CC BY 4.0, archived at
https://perma.cc/ZRR5-7PYW). It describes v2; an update covering v3 follows.

### 1.1 Risk rubric

| Category | Score | Summary |
| --- | --- | --- |
| Privacy-at-risk | 1 | The buyer's identifier never reaches the chain; only `persistentCommit(buyerId32, nonce)` does, and the nonce is held off-chain in a table separate from the commitments. No personal data, no order contents, no prices and no transaction amounts are written to the ledger. The two identifying values that can appear — a public account handle and a referral id — are written only at the buyer's explicit request, for the purchase they choose, and both are values the buyer publishes themselves when they post. A purchase with no binding names no one. § 3.1 covers what the referral id makes enumerable. |
| Value-at-risk | 1 | The contract holds no on-chain assets — no tokens, NFTs, or escrowed value — and no payment passes through it. Adversarial misuse cannot result in user fund loss. The state at risk is two bounded maps of records (§ 3). |
| State-space-at-risk | 1 | All three state-mutating circuits are owner-gated: an unauthorized `recordPurchase`, `recordXBinding` or `rotateOwner` attempt fails before any chain state change. Both maps are insert-only and bounded by physical commerce — one purchase record per (paid order, resolved lot) pair, and at most one binding per purchase. An adversary cannot force writes. |

## 2. Contract at a glance

- Contract name: `gyotak-purchase` (v3)
- Mainnet contract address: `d11d52bd5875ecc2e89e97149e0237db20a91c989f30268950e892655a2a2a57`
- Preprod contract address: `e413ff91958079d1ee2e4c792fe19a9c14a5d4ed7eb7966d3526308a3ea2f846`
- Owner PK: `20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be`
  (same key as v1 and v2; the domain tag is per-contract, see § 7)
- Source: `contracts/gyotak-purchase.compact`, 151 lines, `pragma language_version >= 0.22`,
  single file with no imports beyond `CompactStandardLibrary`

Circuits:

- `publicKey(sk)` — pure helper, generates no proof.
- `constructor(initialOwner)` — sets `owner` once at deploy.
- `recordPurchase(purchaseId, lotId, schema, timestamp)` — owner-gated, ±600 s
  block-time bounded, insert-only.
- `recordXBinding(purchaseId, handle, referrerId, schema, timestamp)` — owner-gated,
  ±600 s bounded, insert-only, and additionally requires that the purchase already
  exists on this ledger.
- `rotateOwner(newOwner)` — owner-gated; replaces `owner` atomically.
- `verifyPurchase(purchaseId)` / `verifyXBinding(purchaseId)` — read-only membership
  checks, returning `[]`.

Witnesses: `getBuyerId(purchaseId)`, `getPurchaseNonce(purchaseId)`, `localSecretKey()`.

The contract is written to by the DApp operator (GYOTAK / ECOSUS CO., LTD.), not by end
users.

### 2.1 Differences from v2

Three changes, listed with their effect on the compiled artefacts.

| Change | Effect |
| --- | --- |
| `XBinding` gains `referrerId: Bytes<32>`, and `recordXBinding` takes it as an argument | `recordXBinding` prover and zkir change |
| `PurchaseRecord` and `XBinding` drop their `purchaseId` field, which duplicated the map key | `recordPurchase` prover and zkir change |
| `verifyPurchase` / `verifyXBinding` return `[]` instead of the record — membership is asserted, and the record itself is read from the indexer | both verify provers shrink (147,103 → 146,751 bytes; zkir 3,683 → 1,593) |

`rotateOwner` is byte-identical to v2 (§ 5.4). Everything in `purchases` is unchanged in
meaning: the same commitment primitive, the same `lotId`, the same time binding.

### 2.2 v2 is frozen

v2 remains deployed at `b6f0b4d275cdc96547042f0c38226aaa95beba95e325a831e1722f1313b15179`
with one purchase and one binding recorded, both from the end-to-end rehearsal described
in PR #316. Nothing further will be written to it. The mirror process that submits
records now targets v3 only, and we made the switch at a moment when no record was
pending, so no purchase is stranded on v2 without its binding.

This follows what `gyotak-catch` did across its own three generations: the operator
points the submitting process at the current contract and leaves the older ones in place,
rather than migrating records or writing to two addresses at once.

## 3. State and threat model

Ledger state:

- `purchases: Map<Bytes<32>, PurchaseRecord>` — one record per purchase, keyed by
  `purchaseId`.
- `bindings: Map<Bytes<32>, XBinding>` — keyed by the same `purchaseId`, holding the
  plaintext `handle` and `referrerId`, plus `boundAt` and `schema`.
- `owner: Bytes<32>` — the single authorised writer's public key.

Private witness data (never reaches chain):

- `buyerId32` — the buyer's identifier, derived off-chain as
  `SHA-256("gyotak:buyer:" ‖ account identifier)`.
- `nonce` — a 32-byte random opening value, issued per (order, lot) pair and held in a
  table separate from the commitments.
- `localSecretKey()` — the operator's admin key, loaded from a mode-600 file outside the
  repository.

Threat model. The adversary we consider is any party holding NIGHT for fees and able to
reach a Midnight proof server. The asset at risk is ledger state space: unbounded growth
of either map.

- **Unauthorized writes.** All three state-mutating circuits execute
  `assert(disclose(publicKey(localSecretKey()) == owner), "unauthorized")` before any state
  change. An adversary lacking the admin key cannot mutate ledger state.
- **Witness leakage.** `buyerId32` and `nonce` are held only in the operator's database and
  loaded into memory by the mirror process for the duration of one submission. They are
  never passed on a command line, never written to logs, and never serialised to chain.
- **Orphan bindings.** `recordXBinding` asserts `purchases.member(purchaseId)`. A binding
  cannot exist without its purchase.
- **Record tampering.** Both maps are insert-only: a second write to the same key is
  rejected. There is no update path and no delete path in the contract. Neither a handle
  nor a referral id can be reassigned after the fact, which is what makes a binding's
  block time meaningful as evidence that the claim preceded the post it supports.
- **Impersonation.** This is the threat v3 addresses, and it is worth stating how far the
  fix reaches. A handle alone can be claimed by anyone. A referral id alone can be copied
  out of a genuine post. Recording both means a fabricated post must match a binding on
  *both* fields, and the only party able to produce that pair is someone holding the
  buyer's own connection URL. It is not a proof of identity — a connection URL can be
  shared or leaked — but it moves the claim from "asserted" to "evidenced". § 6 states
  the remaining gap.
- **Back-dating.** Both writing circuits bind the caller-supplied timestamp to the
  producing block's clock within ±600 seconds, the same window used by `gyotak-temp-log`
  and `gyotak-catch` v3.
- **Value extraction.** Not applicable — the contract custodies no assets.

### 3.1 On storing both identifiers in plaintext

`handle` and `referrerId` are the only fields in either map that identify a person, and
both are stored as plaintext rather than hashed. This is deliberate, and it is a trade
against privacy.

A hashed value would still be verifiable — a reader who sees the post has both values in
front of them and could compute the digest — but it would require every verifier to run a
hash, which for a reader scrolling a social feed is the difference between checking and
not checking. The values are not secret: the buyer publishes both when they post.

The cost is enumerability, and it is larger for the referral id than for the handle. A
referral id is stable across a buyer's purchases, so anyone who reads it from one post can
find every binding that buyer has made. The set of purchases a buyer has *claimed* becomes
publicly linkable. We consider this a property rather than a defect — a reader can
distinguish someone who bought once and keeps promoting from someone who buys repeatedly —
but it is also a consequence a buyer may not anticipate from publishing a single post. The
commitment still protects a buyer who stays silent; a buyer who claims is choosing to be
linkable across their claims, permanently, and we say so in the technical report.

## 4. State-space justification

State growth is bounded by physical commerce, not by adversarial capacity:

- `recordPurchase` is invoked at most once per (paid order, resolved catch lot) pair.
  Commitment issuance is gated on the order being paid and on the picked lot resolving to a
  single catch record; an unpaid or ambiguous order produces no commitment at all.
- `recordXBinding` is invoked at most once per purchase, and only when the buyer requests
  it. In the ordinary case it is never invoked.
- `rotateOwner` is invoked only on operator key rotation.
- The duplicate-key assertions prevent a second write to the same key in either map, even
  from the owner.

All mutations are owner-gated, so an adversary cannot force writes. In practice the write
rate is a few records per order, bounded by GYOTAK's shipment volume.

## 5. Evidence

v3 is deployed on both networks:

```
Mainnet  d11d52bd5875ecc2e89e97149e0237db20a91c989f30268950e892655a2a2a57
Preprod  e413ff91958079d1ee2e4c792fe19a9c14a5d4ed7eb7966d3526308a3ea2f846
Owner    20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be
```

The Mainnet ledger is empty at the time of writing; the records below are from the
Preprod rehearsal.

### 5.1 The commitment identity, measured

The first pair of Preprod records was built specifically to test the claim that the
circuit's `persistentCommit` output equals a plain external SHA-256. Test values were
generated in memory, the digest was computed locally *before* submission, and the
transaction was then sent. Reading the stored record back from the indexer gave:

```
record  PB-20260918-b7f3c55d
local   SHA-256(nonce ‖ buyerId32) = bdeb7b88c067d51437ccf20222b9032faff29920dfdbff1139be2ec3bbc4a4a3
chain   purchaseCommitment         = bdeb7b88c067d51437ccf20222b9032faff29920dfdbff1139be2ec3bbc4a4a3
→ MATCH
```

A binding was then recorded against the same purchase with handle `gyotak_test` and a test
referral id, and read back matching — including the round trip of the referral id through
its 16-byte encoding (§ 5.3).

### 5.2 A real customer order, end to end

The second pair came from a real customer order and travelled the entire path without
manual intervention at any step a customer would not take themselves:

1. The customer ordered through an AI chat connected to the operator's MCP server, over a
   personal token URL that identifies them to the server.
2. Payment was confirmed. The referral id is issued at this moment, automatically.
3. Staff picked three packs of Kuro-Kanpachi from a lot landed on 2026-09-05; the picked
   lot resolved to exactly one catch record already published on `gyotak-catch`.
4. A purchase commitment was issued.
5. The mirror recorded it: `recordPurchase`.
6. The buyer claimed it through their own authenticated connection. The claim carries no
   customer identifier as an argument — the connection token is the only accepted
   credential — and the operator looks up that customer's referral id server-side.
7. The mirror recorded the link: `recordXBinding`, with handle and referral id.

Steps 1–5 happen for every paid order whose lot resolves cleanly. Steps 6–7 happen only
when the buyer asks.

The mirror re-verifies `SHA-256(nonce ‖ buyerId32)` against the stored commitment before
submitting and submits only on a match; it also refuses to write a binding for a purchase
not confirmed on this contract.

| Record | Identifier (SDK txId) | On-chain hash | Block | Time (UTC) |
|---|---|---|---|---|
| Preprod deploy | `001ff976…11f8d01` | `6ca2cdac…ce900c` | 2,600,073 | 06:31:36 |
| Test purchase | `00e63d61…99862c2` | `917a73d9…b6e576` | 2,600,143 | 06:38:36 |
| Test binding | `00ba7342…b585516` | `908c1748…e7bfdd` | 2,600,157 | 06:40:00 |
| Real purchase | `001fa340…2d32636` | `0399eea2…5de27a` | 2,604,101 | 13:14:24 |
| Real binding | `00a63f64…86893e5` | `79623f6b…317608` | 2,604,115 | 13:15:48 |
| Mainnet deploy | `00d4ff9f…1aa8db9` | `6da22391…4cfcce` | 2,636,444 | 13:22:54 |

All six settled successfully. In both Preprod pairs the binding settled after its purchase
— 84 seconds later in each case — and the contract's own `committedAt` and `boundAt`
preserve the same order. Full-length values are in `preprod-evidence-20260918/README.md`.

**Note on transaction identifiers.** As documented in `gyotak-catch`'s application, the
Midnight SDK returns a 66-character *identifier*, which the indexer accepts as
`transactions(offset: { identifier: ... })`. The 64-character cryptographic *hash* is a
separate field. Querying the hash field with an identifier value returns
`invalid transaction hash`. Reviewers using a block explorer should use the hash column.

### 5.3 The referral id encoding, measured

A referral id is `aff_` followed by 32 hex characters — 36 characters, too long for
`Bytes<32>` as ASCII. The hex decodes to 16 raw bytes, which we store right-padded with
zeros. The `aff_` prefix is a constant, so the original value is recoverable.

Both the test binding and the real binding were read back from the indexer and the referral
id reconstructed. The real binding round-tripped
`aff_f2638c39b45f6518f1806bb16f78db87` exactly.

The operator's tooling rejects anything that is not this shape — a missing prefix, a wrong
length, non-hex characters, or a connection token (`gyk_`) passed by mistake — and the
database carries the same constraint independently, so a malformed value cannot reach the
circuit through either path.

### 5.4 Build reproducibility

The contract source is 6,574 bytes,
`SHA-256 b4dc7879e5e872d9044747184f8c8bf62bb0a74b3e647b656d0810bfc901235e`, compiled with
compactc 0.30.0 (language 0.22.0, runtime 0.15.0). A full rebuild from a clean directory,
including proving-key generation, reproduced all twenty artefacts byte for byte.

| File | SHA-256 | vs v2 |
|---|---|---|
| `keys/recordPurchase.prover` | `915146c04fa61ff10ebbeb79e206225566d5f3f1cc53b027c62da989e66d98f6` | changed |
| `keys/recordPurchase.verifier` | `fe840db76d34b841082314d377d66dc1d6e5ad5b0042f757948c81334357409d` | changed |
| `keys/recordXBinding.prover` | `f0ef6c41ea7271c9e4050939b2c9c7a09d14f7575149cd5ee50785a3020d3ab8` | changed |
| `keys/recordXBinding.verifier` | `0eeb65b66bc0a1e4be495eea91c067e3f13d611a81d36dc8e4ae4869f4dece4b` | changed |
| `keys/rotateOwner.prover` | `c507c942114b41c6a364785cc1f844581db54bb46492a13c8389252bb3c8e096` | identical |
| `keys/rotateOwner.verifier` | `f46bf019cac0fa9e0ded222b89353962a9bec5a3ecc342d9cfdfe528981849ea` | identical |
| `keys/verifyPurchase.prover` | `bcb0d01f375896e5a4d0041b4b1f7711435230a973d90d086de18a7d0ed74303` | changed |
| `keys/verifyPurchase.verifier` | `970f0f52ad5796ed92545c9513d373ed01af4412544195271a72108398236eb9` | changed |
| `keys/verifyXBinding.prover` | `99ed09ee710d5854080d26aa0d1ecaf697eb29a3d381bf3624d52dd37bfd14e1` | changed |
| `keys/verifyXBinding.verifier` | `703e775f49c1223982bd38a5246eb408af7161568e45c60058385f4439ed865d` | changed |
| `zkir/recordPurchase.zkir` | `f74dd21a0766d5152b9d36686ac25967ace0d9e4ea555b9fc709415860c8b150` | changed |
| `zkir/recordPurchase.bzkir` | `cf9a72d712a3a9a127a10774415067ad39c07406cf4181043d8f77d90b65f79d` | changed |
| `zkir/recordXBinding.zkir` | `9004c3310f2e027f0e0d8ec3a478662e773b1127a6baf45475fe71b86ed70d59` | changed |
| `zkir/recordXBinding.bzkir` | `bc4019779cdd1ded963b3a7b7c3275058402108ac5fb6017fe2149d3c0df652d` | changed |
| `zkir/rotateOwner.zkir` | `fc727026946475633f6c753076dd0b82d2675813aa5c3393e07f63fe11079e23` | identical |
| `zkir/rotateOwner.bzkir` | `8b5d142c8abc9df303f5b89d2ce59e7f59d03c405e6c937e7a6074313e92f1a3` | identical |
| `zkir/verifyPurchase.zkir` | `737f616b17ff3f746957e6ae0b422339ac8fc683862f8ff0460d22e165219caf` | changed |
| `zkir/verifyPurchase.bzkir` | `6a10fad48a6a8f1e7a5d2524f446ac9cc71992a50f0f70be88cf9e5ed0e05280` | changed |
| `zkir/verifyXBinding.zkir` | `fed0c19a5d70c49364e1ff20127122b6101e56b854e3a3dae4f1ba96bc774fa2` | changed |
| `zkir/verifyXBinding.bzkir` | `623803f676ab3e5066c5e5ba356027e95faf468fb305042d57a7b02f1626bf3c` | changed |

The verifier keys stored in the deployed contracts' state are byte-for-byte identical to
the `keys/*.verifier` files above.

### 5.5 Cost

Preprod: the deploy cost 0.30 DUST and each record 0.30 DUST, so the two rehearsal pairs
came to 1.50 DUST across five transactions.

The Mainnet deploy cost 50 DUST. That figure is a deliberate fee overhead we set for
deploys only: a deploy that fails and has to be repeated produces a different contract
address, which would invalidate every verification link already published. Recording
transactions on Mainnet use the ordinary overhead. The contract holds no assets, so this
is the entire economic footprint.

## 6. Scope of the guarantee

**What the chain attests to:**

- A commitment with this value existed at the stated block and has not been altered since.
- Its timestamp was not back-dated beyond a ±600-second window around the producing block.
- Only a party holding the opening nonce can demonstrate a link to it.
- Where a binding exists, the operator recorded that handle and that referral id together,
  at a block later than the purchase they refer to, and has not altered them since.

**What it does not attest to:**

- **That the claimant is the buyer.** The referral id narrows this considerably — it is
  issued by the operator and reaches the customer only inside their personal connection
  URL, so producing a matching handle-and-referral-id pair requires that URL. But a
  connection URL can be shared or leaked, and the operator would not detect it. The claim
  is evidenced, not proven.

- **That the referral id is still active.** Referral ids are permanent once issued, but the
  operator can suspend one and issue a replacement. A binding is a historical fact — at this
  block, this purchase was claimed under this referral id — not a statement about where a
  commission is payable today.

- **That a purchase physically occurred.** The chain attests that GYOTAK recorded a
  commitment; it cannot attest to events in the world. An operator determined to publish
  false records would produce chain data indistinguishable from honest records.

- **That whoever opens a commitment is the buyer.** Opening demonstrates knowledge of the
  nonce, nothing more. A nonce can be shared or leaked.

- **Anonymity after a binding.** Once a purchase is bound, that purchase is publicly tied to
  both identifiers, and the set of bound accounts and referral ids is enumerable from
  contract state (§ 3.1). The commitment protects a buyer who stays silent; a buyer who
  claims is choosing to be named and to be linkable across their future claims.

We record these limits here rather than only in the technical report, because the
Privacy-at-risk score in § 1.1 rests on what the construction actually protects.

## 7. Operational posture

**Admin key custody.** The admin secret key lives at
`~/midnight/.gyotak-secrets-purchase/admin-sk.txt`, mode 600, in a mode-700 directory
outside the repository. It is a separate file from the key used by `gyotak-catch` and
`gyotak-temp-log`. The loader refuses world- or group-readable files and rejects malformed
content; a missing key throws explicitly rather than silently authenticating with a
stand-in.

**Per-contract authorization.** The `publicKey` derivation uses the domain tag
`gyotak:purchase:pk:`, distinct from `gyotak:pk:` (`gyotak-catch`) and `gyotak:templog:pk:`
(`gyotak-temp-log`). Authorization does not leak across contracts.

**Nonce handling.** Nonces are generated by the Worker and stored in a table separate from
the commitments themselves. The mirror process loads them into memory, zeroes the buffers
after use, and passes them only as in-process witnesses — never on a command line, where
they would appear in the process table. Log output and error messages are filtered for
64-character hex strings before being written.

**Wallet separation.** Preprod and Mainnet use different wallets. The mirror loads the
Mainnet seed from a separate file and refuses to start if that file is absent, rather than
falling back to the Preprod configuration — a fallback would mirror to Mainnet from the
wrong wallet. Every run prints the SHA-256 prefix of the seed it loaded, so a mismatch is
visible in the first line of the log.

**Claim handling.** A buyer claims a purchase through an MCP tool reachable only over their
personal connection URL. That tool takes no customer-identifier argument: the connection
token is the only accepted credential, and the referral id is looked up server-side from
that token rather than supplied by the caller. A referral token (`aff_`) cannot be used to
claim — those connections are deliberately excluded from identity resolution, so a referrer
cannot claim someone else's purchase.

**Rotation.** `rotateOwner` allows the admin key to change without redeploying, preserving
the contract address and therefore every verification link already published.

**Wallet.** This contract shares the operator's Mainnet wallet with `gyotak-catch` and
`gyotak-temp-log`, since DUST is non-transferable and is generated by the NIGHT that wallet
holds. Submission processes for the three contracts are serialised through a shared lock.

## 8. Implementation repository

- Implementation: https://github.com/ecosus-co/gyotak-purchase (public, Apache 2.0)
- Technical report (defensive publication): TR-2026-015 —
  https://gyotak-tr.pages.dev/tr-2026-015/ (CC BY 4.0), archived at
  https://perma.cc/ZRR5-7PYW
- Preprod evidence: included in the implementation repository under
  `preprod-evidence-20260918/`

### Notes

- Implementation code is in the linked external repository per the MIP submission
  convention; this PR adds only the deployment request document.
- License: both this PR's file and the implementation repository are Apache License 2.0.
- v2 was authorized through PR #316 and its address recorded in PR #317. This request
  covers a new contract address and therefore requires its own authorization, as Circuit
  Hub approval is per-contract. v2 remains deployed and frozen (§ 2.2).
- The v3 Mainnet contract was deployed on 2026-09-18 ahead of this request. We record that
  plainly rather than omit it: the deploy transaction is in § 5.2, the ledger is empty, and
  no record has been written to it. If the Foundation would prefer the address retired and
  redeployed after authorization, we will do so.
