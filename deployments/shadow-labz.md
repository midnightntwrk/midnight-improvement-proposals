# [Deployment Request] Shadow Labz

**dApp name:** Shadow Labz (VoteChain)

**Contract repository:** https://github.com/paranormal39/Votechain (private) —
Compact sources in `contract/` (`shadow_polls.compact`, `shadowpollv1.compact`,
`shadowfeedback.compact`, `shadowdaov1.compact`). _The repo is private, but
**Jay Albert** from the Midnight team already has access. Any other reviewer can
request access and we'll add them immediately._

**Target network:** Mainnet

**Brief description:**
Shadow Labz is a suite of privacy-preserving governance tools built on Midnight.
This request covers three contracts, each deployed once and serving many
instances:

- **Shadow Polls** (`shadow_polls.compact` single-question + `shadowpollv1.compact`
  multi-question) — Sybil-resistant, private polling. Participants prove
  eligibility against a Merkle root and cast a ballot; nullifiers enforce
  one-vote-per-identity. Only aggregate per-option tallies are public — never
  the link between a voter and their choice.
- **Shadow Feedback** (`shadowfeedback.compact`) — modular anonymous feedback.
  Submitters prove eligibility (open or member-scoped) and burn a
  one-per-topic nullifier. Feedback content is stored on-chain only as
  ciphertext end-to-end encrypted to the topic owner.
- **ShadowDAO v1** (`shadowdaov1.compact`) — unified DAO governance: permissioned
  proposal creation and commit–reveal voting (choice hidden until reveal). The
  deployed build is **governance-only and holds no funds** — the treasury
  circuits (`fund_treasury` / `execute_approved_spend`) are excluded from the
  deployment.

All three contracts in this request are **fully non-custodial** — none receive,
hold, or send the native token. The standalone **launchpad escrow contract
(`escrow.compact`) is the only fund-custody contract and is deferred to a
separate follow-up submission** — it is NOT part of this request.

---

## Summary scores

| Contract | Privacy-at-risk | Value-at-risk | State-space-at-risk |
|---|---|---|---|
| Shadow Polls (v0 + v1) | 1 | 1 | 2 |
| Shadow Feedback | 1 | 1 | 2 |
| ShadowDAO v1 (governance-only) | 1 | 1 | 2 |

No category scores 3. Per-contract rationale and mitigations follow.

---

## Shadow Polls (`shadow_polls.compact`, `shadowpollv1.compact`)

Exported circuits: `update_block_height`, `create_poll`, `activate_poll`,
`cancel_poll`, `register_ticket`, `cast_vote_open`, `cast_vote_ticketed`,
`close_poll`, `close_poll_by_time`, `finalize_result` (+ `prove_participation`
in v0 / `submit_feedback` in v1). No fund-custody API is present.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | On-chain data is aggregate-only and non-identity: poll config (metaHash, option/question counts, status, deadline, eligibility mode), opaque secret-derived ticket commitments, opaque vote/participation nullifier sets, and aggregate per-option tallies. The voter→choice link never touches the chain (nullifiers are secret-derived; eligibility discloses only a Merkle root). A ZK fault would at worst expose an aggregate tally or an opaque commitment — no real-world-identity data. | Commitment/nullifier design; participant secret never leaves the client; domain-separated hashes (`sp:*`) prevent cross-protocol linking. **Documented caveat:** tallies update live, so in a very small poll an observer correlating tx timing could infer a choice; `HIDDEN_UNTIL_CLOSE` (commit–reveal) is the recorded follow-up and `resultVisibility` is already stored for forward-compat. |
| Value-at-risk | 1 | No funds are ever locked in the contract — there is no coin/native-token API. An exploit cannot cause direct loss of user principal; risk is limited to gas/liveness. | N/A — non-custodial by design. |
| State-space-at-risk | 2 | State grows with usage (ticket commitment tree, nullifier sets, per-poll config and per-(question,option) tallies) but is bounded per poll and per participant: one vote nullifier per identity per poll caps writes, and eligibility gating (TICKET_TREE mode) bounds who can write. No unbounded free-for-all log. | Sybil nullifiers + per-secret one-vote provide a natural ceiling; ticketed mode further gates writes to organizer-registered participants. Merkle trees are fixed-depth (bounded leaves). |

---

## Shadow Feedback (`shadowfeedback.compact`)

Exported circuits: `create_topic`, `register_member`, `close_topic`,
`submit_feedback`. No fund-custody API is present.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | Submitter identity and feedback content are private: content is stored on-chain only as opaque ciphertext (fixed 2048-byte slots) end-to-end encrypted to the topic owner's off-chain key. Public data is topic existence, owner pubkey, eligibility mode, open/closed flag, a hash of off-chain metadata, opaque member commitments, an opaque feedback-nullifier set, and a per-topic submission count. A ZK fault would expose a ciphertext blob or opaque commitment — not plaintext or identity. | Commitment/nullifier design; member secret never leaves the client; topic-scoped commitments prevent cross-topic linking; domain-separated hashes (`fb:*`). Content confidentiality relies on off-chain E2E encryption to the owner key, independent of the ZK layer. |
| Value-at-risk | 1 | No funds are ever locked in the contract; there is no coin/native-token API. | N/A — non-custodial by design. |
| State-space-at-risk | 2 | State grows with usage (member commitment tree, feedback-nullifier set, per-topic count, and per-(topic,index) ciphertext slots) but is bounded per submitter: one submission per (topic, submitter) via nullifier caps writes, and MEMBER-mode topics gate writes to owner-registered members. Content slots are fixed-size (2048 bytes). | One-per-(topic,submitter) nullifier + MEMBER-mode eligibility provide the natural ceiling; fixed-depth Merkle tree; fixed-size content slots keep per-write cost predictable. |

---

## ShadowDAO v1 (`shadowdaov1.compact`) — governance-only build

Exported circuits in the deployed build: `manage_voter`, `update_block_height`,
`create_proposal`, `vote_commit`, `vote_reveal`, `advance_proposal_by_time`,
`check_proposal_result`. **The treasury circuits (`fund_treasury` /
`execute_approved_spend`) are excluded from this deployment**, so the deployed
contract has no coin/native-token API and never receives, holds, or sends funds.

| Category | Self-assessed score (1–3) | Rationale | Mitigations (if applicable) |
|---|---|---|---|
| Privacy-at-risk | 1 | Vote choice is a private witness and hidden until reveal (true commit–reveal); admin/member secret keys never leave the client. On-chain data is aggregate/opaque only: admin pubkeys, an opaque eligible-voter Merkle root, per-proposal metadata/state/deadlines/quorum, aggregate tallies, and opaque commit/reveal nullifier sets. With the treasury excluded, no financial amounts are disclosed. A ZK fault would at worst expose an aggregate tally or an opaque commitment — no real-world-identity data. | Commit–reveal hides the ballot at commit time; range-checked reveal; domain-separated hashes (`dao:*`); eligible-voter set stored as an opaque Merkle root. |
| Value-at-risk | 1 | The deployed governance-only build holds no funds — the treasury circuits are removed, so there is no coin/native-token API. An exploit cannot cause direct loss of user principal; risk is limited to gas/liveness. | N/A — non-custodial by design. |
| State-space-at-risk | 2 | State grows with usage (eligible-voter Merkle tree, per-proposal metadata/state/deadlines/quorum/tallies, commit/reveal nullifier sets, vote-commitment tree) but is bounded per proposal and per voter: one commit + one reveal nullifier per voter per proposal caps write frequency, and proposal creation is permissioned. No cheap unbounded public log. | Permissioned `create_proposal` (admin or eligible non-revoked member) + proposalId uniqueness prevent spam/overwrite; per-proposal commit/reveal nullifiers cap writes per voter; fixed-depth Merkle trees; `manage_voter` revocation bounds the eligible set. |

---

## Deferred: Launchpad Escrow (`escrow.compact`)

Not part of this request. The launchpad escrow is a dedicated per-project
custodian of the native token (open → deposit → release/refund). Because funds
accumulate per project until the goal is met or refunded, it warrants its own
value-at-risk review and TVL-cap discussion, and will be submitted separately
once those mitigations are finalized.

---

_If any category is judged a 3 during review, we will discuss architecture in
[Discord #dev-chat](https://discord.gg/midnightnetwork) / the
[developer forum](https://forum.midnight.network/) before going live._
