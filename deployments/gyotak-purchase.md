# Deployment Request: gyotak-purchase

## 1. Summary

`gyotak-purchase` is a Compact smart contract that records purchase commitments for
flash-frozen seafood, deployed as the third contract in the GYOTAK traceability system.
The first two — `gyotak-catch` (PR #96) and `gyotak-temp-log` (PR #224) — are
Mainnet-approved and cover catch provenance and cold-chain evidence respectively. This
contract closes the chain at the buyer's end.

The problem is specific. When a customer publishes an account of a meal on a social
network the seller does not control, a reader has no way to distinguish it from a
fabricated one, and platform-side "verified purchase" badges do not travel outside the
platform. `gyotak-purchase` publishes two kinds of record. Every purchase becomes a
commitment that fixes the record in time while keeping the buyer off-chain. Separately,
a buyer who chooses to speak publicly can attach their account handle to one of their
purchases; that attachment is a second record, written later, and never written for a
purchase that has none.

Ledger state:

- **`purchases`** — one record per purchase, holding `purchaseCommitment`
  (`persistentCommit<Bytes<32>>(buyerId32, nonce)`), `lotId`, `committedAt` and `schema`.
  The buyer identifier and the opening nonce enter as witnesses and never reach the chain.
- **`bindings`** — the subset of purchases a buyer has attached a public handle to. The
  handle is stored in plaintext; `boundAt` records when the claim was made.
- **`owner`** — the single authorised writer's public key.

Two design points are worth stating up front because the rest of the document rests on
them. First, the commitment is deliberately the plain `SHA-256(nonce ‖ buyerId32)`, which
we verified empirically against an externally computed digest (§ 5.1); a buyer can
therefore open their own record with one hash, with no Midnight tooling and no cooperation
from us. Second, the handle in a binding is self-reported: the operator authenticates the
buyer but does not verify that the account is theirs. § 6 states the boundary in full.

A full technical report is published as a defensive publication: TR-2026-015
(https://gyotak-tr.pages.dev/tr-2026-015/, CC BY 4.0, permanently archived at
https://perma.cc/5TXA-G7BP).

### 1.1 Risk rubric

| Category | Score | Summary |
| --- | --- | --- |
| Privacy-at-risk | 1 | The buyer's identifier never reaches the chain; only `persistentCommit(buyerId32, nonce)` does, and the nonce is held off-chain in a table separate from the commitments. No personal data, no order contents, no prices and no transaction amounts are written to the ledger. The one piece of identifying information that can appear — a public account handle — is written only at the buyer's explicit request, for the specific purchase they choose, and is a handle they have already published themselves. A purchase with no binding names no one (see § 3). |
| Value-at-risk | 1 | The contract holds no on-chain assets — no tokens, NFTs, or escrowed value — and no payment passes through it. Adversarial misuse cannot result in user fund loss. The state at risk is two bounded maps of records (§ 3). |
| State-space-at-risk | 1 | All three state-mutating circuits are owner-gated: an unauthorized `recordPurchase`, `recordXBinding` or `rotateOwner` attempt fails before any chain state change. Both maps are insert-only and bounded by physical commerce — one purchase record per (paid order, resolved lot) pair, and at most one binding per purchase. An adversary cannot force writes. |

Detailed justification with cross-references follows in §§ 3–5.

## 2. Contract at a glance

- Contract name: `gyotak-purchase`
- Preprod contract address: `fe71367b28596c91a490e7e900e3d521d0597ec5456020b3e8e19cd5e76d7043`
- Initial owner PK: `20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be`
- Source: `contracts/gyotak-purchase.compact`, 180 lines, `pragma language_version >= 0.22`,
  single file with no imports beyond `CompactStandardLibrary`

Circuits:

- `publicKey(sk)` — pure helper, generates no proof. Derives a 32-byte public key via
  `persistentHash<Vector<2, Bytes<32>>>([pad(32, "gyotak:purchase:pk:"), sk])`.
- `constructor(initialOwner)` — sets `owner` once at deploy.
- `recordPurchase(purchaseId, lotId, schema, timestamp)` — owner-gated, ±600 s
  block-time bounded, insert-only.
- `recordXBinding(purchaseId, handle, schema, timestamp)` — owner-gated, ±600 s bounded,
  insert-only, and additionally requires that the purchase already exists on this ledger.
- `rotateOwner(newOwner)` — owner-gated; replaces `owner` atomically.
- `verifyPurchase(purchaseId)` / `verifyXBinding(purchaseId)` — read-only public lookups.

Witnesses: `getBuyerId(purchaseId)`, `getPurchaseNonce(purchaseId)`, `localSecretKey()`.

The contract is written to by the DApp operator (GYOTAK / ECOSUS CO., LTD.), not by end
users. Purchase commitments originate from the operator's ordering system → Cloudflare
Worker → D1 → a mirror process that submits to this contract.

### 2.1 Relationship to `gyotak-catch` v3

The structure follows `gyotak-catch` v3's `recordCatch` (documented in TR-2026-012) with
two differences: there is no GPS range proof, because a purchase has no coordinates, and
the committed value is the buyer's identifier rather than a location. The commitment
primitive is identical.

## 3. State and threat model

Ledger state:

- `purchases: Map<Bytes<32>, PurchaseRecord>` — one record per purchase, keyed by
  `purchaseId`.
- `bindings: Map<Bytes<32>, XBinding>` — keyed by the same `purchaseId`, holding the
  plaintext `handle`, `boundAt` and `schema`.
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
  change. An adversary lacking the admin key cannot mutate ledger state; the attempt fails
  at SDK simulation, again at proof generation, and again at settlement.
- **Witness leakage.** `buyerId32` and `nonce` are held only in the operator's database and
  loaded into memory by the mirror process for the duration of one submission. They are
  never passed on a command line, never written to logs, and never serialised to chain.
  Only the commitment and the ZK proof reach the ledger.
- **Orphan bindings.** `recordXBinding` asserts `purchases.member(purchaseId)`. A binding
  cannot exist without its purchase, so there is no way to publish a claim about a purchase
  that was never recorded.
- **Record tampering.** Both maps are insert-only: a second write to the same key is
  rejected in either map. There is no update path and no delete path in the contract. A
  handle in particular cannot be reassigned after the fact, which is what makes a binding's
  block time meaningful as evidence that the claim preceded the post it supports.
- **Back-dating.** Both writing circuits bind the caller-supplied timestamp to the producing
  block's clock within ±600 seconds via `blockTimeGte(t - 600)` and `blockTimeLte(t + 600)`,
  the same window used by `gyotak-temp-log` and `gyotak-catch` v3.
- **Value extraction.** Not applicable — the contract custodies no assets.

### 3.1 On the choice to store handles in plaintext

`handle` is the only field in either map that identifies a person, and it is stored as
plaintext ASCII rather than hashed. This is deliberate, and it is a trade against privacy.

A hashed handle would still be verifiable — a reader who sees the post knows the handle and
could compute the digest — but it would require every verifier to run a hash, which for a
reader scrolling a social feed is the difference between checking and not checking. The
cost of plaintext is that the set of accounts that have claimed a GYOTAK purchase is
publicly enumerable from contract state. We accept that cost because these are accounts
that chose to speak: a binding exists only where a buyer asked for one, naming an account
they have already made public themselves.

Buyers who do not ask are unaffected. Their purchases appear in `purchases` as commitments
and nowhere else, which is the ordinary case and the reason the two maps are separate.

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

## 5. Preprod evidence (2026-09-17)

The contract is deployed on Preprod at
`fe71367b28596c91a490e7e900e3d521d0597ec5456020b3e8e19cd5e76d7043` with
`owner = 20fc1d0d…2af2e1be`. At the time of writing `purchases.size` and `bindings.size`
are both 2. Every transaction below is independently queryable on the Preprod indexer.

### 5.1 The commitment identity, measured

The first pair of records was built specifically to test the claim that the circuit's
`persistentCommit` output equals a plain external SHA-256. Test values were generated in
memory, the digest was computed locally *before* submission, and the transaction was then
sent. Reading the stored record back from the indexer gave:

```
record  PB-20260917-fa72e7fd
local   SHA-256(nonce ‖ buyerId32) = b4677b65f1219aa532c9efee4af7283c2a84565e5495433ca022e748b4fe602a
chain   purchaseCommitment         = b4677b65f1219aa532c9efee4af7283c2a84565e5495433ca022e748b4fe602a
→ MATCH
```

The `lotId` matched the independently computed `SHA-256("gyotak:lot:" ‖ catch_report_id)`
as well. A binding was then recorded against the same purchase with handle `gyotak_test`
and read back matching. This is the empirical basis for the claim that a buyer needs no
Midnight tooling to open their own commitment — one hash, computed anywhere, in any
language.

### 5.2 A real customer order, end to end

The second pair came from a real customer order and travelled the entire path without
manual intervention at any step a customer would not take themselves:

1. The customer ordered through an AI chat connected to the operator's MCP server, over a
   personal token URL that identifies them to the server.
2. Payment was confirmed.
3. Staff picked three packs of Kuro-Kanpachi from a lot landed on 2026-09-06; the picked
   lot resolved to exactly one catch record already published on `gyotak-catch`.
4. A purchase commitment was issued.
5. A mirror process recorded it: `recordPurchase`.
6. The buyer claimed it through their own authenticated connection and supplied the handle
   `gyotaku_proto`.
7. The mirror recorded the link: `recordXBinding`.

Steps 1–5 happen for every paid order whose lot resolves cleanly. Steps 6–7 happen only
when the buyer asks. The mirror re-verifies `SHA-256(nonce ‖ buyerId32)` against the stored
commitment before submitting and submits only on a match; it also refuses to write a
binding for a purchase not confirmed on this contract.

| Record | Identifier (SDK txId) | On-chain hash | Block | Time (UTC) |
|---|---|---|---|---|
| Deploy | `004008a1…9ef75a` | `46dc75f2…93274` | 2,588,244 | 10:48:42 |
| Test purchase | `00eed986…a05792` | `a5a901e6…5d290` | 2,588,347 | 10:59:00 |
| Test binding | `00c63466…5ed16a` | `7f6e2f1c…f18e1` | 2,588,389 | 11:03:12 |
| Real purchase | `00055790…f166eac` | `79b36e0c…9286c9c` | 2,589,935 | 13:37:48 |
| Real binding | `00270451…e0a9b5` | `30e4147b…37f66a` | 2,589,950 | 13:39:18 |

In both pairs the binding settled after its purchase — 4 min 12 s later for the test pair,
90 s for the real one — and the contract's own `committedAt` and `boundAt` preserve the
same order. Full-length values for all five transactions are in
`preprod-evidence-20260917/README.md`.

**Note on transaction identifiers.** As documented in `gyotak-catch`'s application, the
Midnight SDK returns a 66-character *identifier*, which the indexer accepts as
`transactions(offset: { identifier: ... })`. The 64-character cryptographic *hash* is a
separate field. Querying the hash field with an identifier value returns
`invalid transaction hash`; we verified this explicitly. Reviewers using a block explorer
should use the hash column; reviewers querying the indexer may use either field with the
correct name.

### 5.3 Independent verification path

The Preprod indexer exposes contract state over GraphQL. Querying `contractAction(address)`
for the contract address above and reading the `state` field returns both maps without any
GYOTAK service in the path. To check a post, a verifier takes the purchase id from the post,
reads both records, and compares the on-chain handle to the account that published it.
Where a buyer has published their nonce, the commitment opens locally with one SHA-256.

One operational caveat we consider worth stating, because we observed it in practice: the
indexer accepts `POST` only and returns HTTP 405 to `GET`. An AI agent that can fetch URLs
but cannot issue a POST receives no data, and may incorrectly report that no record exists.
TR-2026-015 § 6 addresses this directly, instructing such agents to report "could not
verify" rather than "no such record."

### 5.4 Build reproducibility

The contract source is 7,123 bytes,
`SHA-256 0740779ca34a8eb5b75c7fee9d2b02c25881e392c2b2d803888a0f65422ad9c8`, compiled with
compactc 0.30.0 (language 0.22.0, runtime 0.15.0). A full rebuild from a clean directory,
including proving-key generation, reproduced all twenty artefacts under
`contracts/managed/keys` and `contracts/managed/zkir` byte for byte.

| File | SHA-256 |
|---|---|
| `keys/recordPurchase.prover` | `6022a4241fc876fd9c2f3de0fd6c75e745b84639d88c216c12eab0106b490df2` |
| `keys/recordPurchase.verifier` | `445c2f551e1d2538034311f35b6e457a889bb7bbab84e004b1e239f3e8963d53` |
| `keys/recordXBinding.prover` | `c627e68483266e947e5b6f8ccc50d9c150b3a81637db16f0efdcebaa81f71f36` |
| `keys/recordXBinding.verifier` | `673c6d9959685b9b0be6cc75d8b9941b9ead468c68edf977d545d056464a557c` |
| `keys/rotateOwner.prover` | `c507c942114b41c6a364785cc1f844581db54bb46492a13c8389252bb3c8e096` |
| `keys/rotateOwner.verifier` | `f46bf019cac0fa9e0ded222b89353962a9bec5a3ecc342d9cfdfe528981849ea` |
| `keys/verifyPurchase.prover` | `af7d5dafb2793f4b63e69c52987a5453492b820e8081740e527b61529261f9e2` |
| `keys/verifyPurchase.verifier` | `fcfda347ac3d18dd697fa8cf74ea2f31687a274686ea60e23bb0135fce7da795` |
| `keys/verifyXBinding.prover` | `8288475d946d53f37241e5cfa482c485235eb45cf036e6f2d98bd16034980a80` |
| `keys/verifyXBinding.verifier` | `dff7f33d1e23379e7ff3c085ee50787cb7d696a30ac80d5bdcb2a00042074e68` |
| `zkir/recordPurchase.zkir` | `5c0b067681ccc896fdd3d5022ff54475dc5763eaa6580cfd57f20b76027a3f84` |
| `zkir/recordPurchase.bzkir` | `d00323133c4038311c3467c43b287101370903a572e4b36c15768473f10c89ab` |
| `zkir/recordXBinding.zkir` | `a1ccd0185f67d9af2f3e53f25469ce55038a6f36894b1a3f314c87dce0ccbc36` |
| `zkir/recordXBinding.bzkir` | `a08381976f80db5e121cc7eba1d6e442fa59b387be7c2e4383a22a32538c8c84` |
| `zkir/rotateOwner.zkir` | `fc727026946475633f6c753076dd0b82d2675813aa5c3393e07f63fe11079e23` |
| `zkir/rotateOwner.bzkir` | `8b5d142c8abc9df303f5b89d2ce59e7f59d03c405e6c937e7a6074313e92f1a3` |
| `zkir/verifyPurchase.zkir` | `5dd7bbb032b37b5ad3af5c16b82c826a78c08e1c38b9fb093026dc8c61638e71` |
| `zkir/verifyPurchase.bzkir` | `6918b9694b327036f98d857f66e0f63d9fd3cf91d9cbf7500ade36bbec278e78` |
| `zkir/verifyXBinding.zkir` | `a16787f26c257ce06cfcbaac21378e5a0a8d0a2b7fb68d6dc402a26a4737b007` |
| `zkir/verifyXBinding.bzkir` | `640656a3b675cb2692b2fe126771482b17772bb74b6777cce1cb699a408fbb5b` |

The verifier keys stored in the deployed Preprod contract's state are byte-for-byte
identical to the `keys/*.verifier` files above, which confirms that the published source is
the one deployed.

### 5.5 Cost

Each transaction settled for approximately 0.30 DUST, read from the `DustSpendProcessed`
events on chain rather than from the indexer's fee field, which reports 1 speck and does not
reflect what was actually spent. A purchase and its binding therefore cost about 0.60 DUST
in total. The contract holds no assets, so this is the entire economic footprint.

## 6. Scope of the guarantee

We state the boundary explicitly, because the word "proof" invites a stronger reading than
this construction supports.

**What the chain attests to:**

- A commitment with this value existed at the stated block and has not been altered since.
- Its timestamp was not back-dated beyond a ±600-second window around the producing block.
- Only a party holding the opening nonce can demonstrate a link to it.
- Where a binding exists, the operator recorded that claim at a block later than the
  purchase it refers to, and has not altered it since.

**What it does not attest to:**

- **That the handle belongs to the buyer.** This is the sharpest limit. The claim arrives
  over the buyer's own authenticated connection, so the operator knows *which customer* is
  speaking — but the handle itself is self-reported and never checked. A buyer could
  register an account that is not theirs, and the operator would not detect it. The
  mitigation available today is social rather than cryptographic: an account that publishes
  its own purchase id, in its own profile or a pinned post, is doing something the operator
  cannot do on its behalf, and a reader can check that directly. The contract does not
  require it.

  One structural pressure is worth naming. We intend to tie referral rewards to the bound
  handle, in which case registering someone else's handle sends the reward to that person,
  so a buyer has no self-interested reason to misreport. That does nothing against a buyer
  whose aim is to impersonate rather than to profit.

- **That a purchase physically occurred.** The chain attests that GYOTAK recorded a
  commitment; it cannot attest to events in the world. An operator determined to publish
  false records would produce chain data indistinguishable from honest records.

- **That whoever opens a commitment is the buyer.** Opening demonstrates knowledge of the
  nonce, nothing more. A nonce can be shared or leaked. The binding addresses a different
  question — which account claims the purchase — and does not close this one.

- **Anonymity after a binding.** Once a purchase is bound, that purchase is publicly tied to
  the handle, and the set of bound accounts is enumerable from contract state (§ 3.1). The
  commitment protects a buyer who stays silent; a buyer who claims is choosing to be named.

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
(`gyotak-temp-log`). The three contracts' on-chain owner keys are correspondingly distinct;
authorization does not leak across contracts.

**Nonce handling.** Nonces are generated by the Worker and stored in
`purchase_commitment_secrets`, a table separate from the commitments themselves. The mirror
process loads them into memory, zeroes the buffers after use, and passes them only as
in-process witnesses — never on a command line, where they would appear in the process
table. Log output and error messages are filtered for 64-character hex strings before being
written or stored.

**Claim handling.** A buyer claims a purchase through an MCP tool reachable only over their
personal token URL. That tool takes no customer-identifier argument: the connection token
is the only accepted credential. This is narrower than our other customer-scoped tools,
which accept a customer key as well, and the reason is that a binding cannot be undone once
recorded — we require a credential that appears only in the buyer's own connection URL,
rather than one that is printed in their order confirmation email.

**Rotation.** `rotateOwner` allows the admin key to change without redeploying, preserving
the contract address and therefore every verification link already published.

**Recovery.** `WALLET_SEED` is the single source of truth for wallet sub-keys, derived
deterministically via HD derivation. The admin key is backed up separately from the seed.

**Wallet.** This contract shares the operator's Mainnet wallet with `gyotak-catch` and
`gyotak-temp-log`, since DUST is non-transferable and is generated by the NIGHT that wallet
holds. Submission processes for the three contracts are serialised through a shared lock to
prevent concurrent DUST spends.

## 8. Implementation repository

- Implementation: https://github.com/ecosus-co/gyotak-purchase (public, Apache 2.0)
- Technical report (defensive publication): TR-2026-015 —
  https://gyotak-tr.pages.dev/tr-2026-015/ (CC BY 4.0), archived at
  https://perma.cc/5TXA-G7BP
- Preprod evidence: included in the implementation repository under `preprod-evidence-20260917/`

### Notes

- Implementation code is in the linked external repository per the MIP submission
  convention; this PR adds only the deployment request document.
- License: both this PR's file and the implementation repository are Apache License 2.0.
- `gyotak-catch` (PR #96) and `gyotak-temp-log` (PR #224) are approved and live on Mainnet.
  This request covers a third, separate contract, which requires its own authorization as
  Circuit Hub approval is per-contract.
