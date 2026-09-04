---
MIP: xxxx
Title: Committee Bridge Consensus Integration
Authors:
  - Kasey White
  - Jon Rossie
Status: Draft
Category: Core
Created: 2026-07-31
Requires: none
License: Apache-2.0
Replaces: none
---

<!--
 Copyright Midnight Foundation

 Licensed under the Apache License, Version 2.0 (the "License");
 you may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     https://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

The committee bridge lets Cardano contracts verify facts about the Midnight
chain. A light client on Cardano tracks the Midnight BEEFY committee by
induction, where each committee vouches for its successor, and
committee-signed Merkle Mountain Range (MMR) roots let Cardano verify
Merkle inclusion of any Midnight block without a proof system in the loop.
The design is standard Polkadot BEEFY, under which all bridge data is
consensus-enforced chain content through the block header digest. This MIP
specifies that model plus two Midnight-specific pieces: a deduplicated
committee commitment sized for Cardano transaction limits, and quorum
weighting by seat count.

## Motivation

The Block Production Rewards MIP pays NIGHT on Cardano justified solely by
committee-signed Midnight facts. That is sound only if the committee and
the data it signs are consensus-validated chain content rather than a side
channel from whoever runs a relay, and only if verification fits Cardano's
execution and transaction-size budgets. Beyond rewards, this bridge is the
foundation for any Midnight-to-Cardano transfer mechanism: anything that
must prove a Midnight fact on Cardano builds on it.

## Specification

### Consensus enforcement via header digest

Every block, the runtime appends a leaf to an MMR (Keccak-256) and deposits
the new root as a BEEFY consensus digest in the block header. Digests are
consensus-enforced: every importing node re-executes the block and rejects
it if the computed digest does not match the header. The root therefore
commits to every prior block and, through the leaf contents below, to the
committee and its successor. Committee selection itself is validated the
same way: every node recomputes Ariadne from its own Cardano observations
and rejects a block carrying a wrong committee, so a signed commitment
always names a committee every honest node agreed was correctly selected.
The BEEFY signatures live in off-chain justifications: votes are gossiped
between nodes, and each node stores the aggregated justification alongside
the finalized block in its database and serves it over RPC. A relay
carries justifications to Cardano.

### MMR leaf

Each leaf (version 0.1) contains `parent_number_and_hash`, which makes
every block header and anything it commits to provable by inclusion proof;
`beefy_next_authority_set`, the successor committee commitment enabling
handover by induction; and `leaf_extra`, which is empty. A version bump is
the extension point if Cardano ever needs extra per-block data.

### Committee commitment (deduplicated, seat-weighted)

Midnight committees are selected by Ariadne, the Partner Chains selection
algorithm, with repetition: one member can hold multiple seats, and seats
are the stake weight (see Rationale). Membership and seat counts are
already public, since Ariadne selection is deterministic over public
Cardano data; the commitment reveals nothing new, and slot-level
production schedules are not involved. Rather than one Merkle leaf per seat
(the upstream default), the commitment deduplicates keys to fit Cardano
transaction limits:

- Group the committee by BEEFY key (33-byte compressed secp256k1 ECDSA
  public key) and count seats per key.
- Sort ascending by key bytes, with one Merkle leaf per distinct key:
  `pubkey (33 bytes) ‖ seat_count (u64 little-endian)`.
- `keyset_commitment` is the Keccak-256 binary Merkle root over the leaves.
- `len` is the **total seat count**, not the distinct key count, and is
  the quorum denominator.

The commitment `(id: u64, len: u32, keyset_commitment: 32 bytes)` is 44
bytes regardless of committee size.

### Signed commitments and sessions

The BEEFY payload is the MMR root alone (the upstream `mh` payload).
Validators sign `(payload, block_number, validator_set_id)`, and a
justification is valid with signatures from at least `len − (len−1)/3`
seats. The set id increments each session. A mandatory signed round on the
first block of each session, signed by the incoming set, guarantees one
justification per session as long as two-thirds of seats vote.

### Cardano-side verification

The light client contract stores the 44-byte commitment of the current
committee. A relay submits a signed commitment along with, for each
signer, the key, its seat count, a Merkle proof against
`keyset_commitment`, and one ECDSA signature. Signers must appear in
strictly increasing key order, which prevents double-counting, and the
root is accepted when the valid signers' seat counts sum to at least
`len − (len−1)/3`.

**Transaction size.** A signer entry is 106 bytes (33-byte key, 8-byte
seat count, 65-byte signature), and membership is proven with a single
Merkle multiproof over the whole signer set. At 100 distinct keys and a
67-key quorum the submission is roughly 10 KB against Cardano's
16,384-byte limit, so on the order of 100 unique signers fit in one
transaction.

**Handover.** Every leaf of session N carries the commitment of set N+1.
The contract must extract set N+1 from a leaf proven against a root signed
by set N *before* it accepts any signature from set N+1; a session's own
mandatory justification can never bootstrap its own set. The genesis
committee is installed at deployment. Handovers are consumed in order.

### Light-client state and consumption

The light client is a single UTxO, identified by an NFT, whose datum is
the entire bridge state:

- `latest_mmr_root` is the most recent quorum-verified MMR root. Because
  the payload is the MMR root itself, this is exactly what the committee
  signed.
- `latest_height` is the Midnight block number the root was signed at.
  Submissions must be strictly newer, which orders updates and rejects
  replays.
- `beefy_activation_block` is the first Midnight block at which BEEFY
  voting is active; proofs cannot reach behind it.
- `current_committee` is the 44-byte commitment of the set authorized to
  sign now.
- `next_committee` is the 44-byte commitment of its successor, taken from
  a proven leaf per the handover rules above.

Seat counts need no state of their own: `len` inside each commitment is
the quorum denominator, so the datum carries no separate stake or weight
totals. Accepting a signed commitment replaces `latest_mmr_root` and
`latest_height` in one step and, when the proven leaf names a new
successor set, advances the committee pair.

Consumers never spend the light client. A downstream contract, such as
the rewards contract, reads the datum as a reference input and verifies
a Keccak-256 MMR inclusion proof against `latest_mmr_root` itself. Since
every root commits to all prior blocks, the latest root proves any past
Midnight block, and root updates never contend with consumption.

### Rewards coupling

Block-production rewards are paid in NIGHT on Cardano and unlock strictly
in session order: session N's reward data becomes claimable only after
session N's commitment is verified. No consensus rule forces voting;
participation is driven by this coupling, since a committee that fails to
produce justifications delays all reward payments, including its own.
Individual non-participation is handled by monitoring and the ban lever
(see Accountability).

### Keys

Each block producer registers a dedicated BEEFY session key (`beef`) in
the existing Cardano-side candidate registration (the candidate-keys
datum). There is no fallback to the candidate's cross-chain ECDSA key:
BEEFY signing requires the key in the node's hot keystore, and the
cross-chain key is the candidate's registration identity, which must not
live there. Registrations without a BEEFY key are excluded from
candidacy once BEEFY voting activates. Permissioned members supply keys
through governance configuration rather than the SPO registration flow.
Misbehavior attribution maps the BEEFY key to the candidate through the
registered candidate keys.

## Rationale: paths considered and rejected

- **Explicit stake weights in the commitment (rejected).** A
  stake-weighted quorum conflicts with the standard Polkadot BEEFY model:
  upstream voting and justification verification count seats, so explicit
  stakes would require forking `sc-consensus-beefy`, and they would
  double-count stake that seat repetition already expresses. It also buys
  nothing, since GRANDPA finality is already a two-thirds count over the
  same seats, so a bridge quorum cannot exceed the chain's own security.
  Seats *are* the stake weight under Ariadne; the bridge simply inherits
  the chain's trust model.
- **Extra payload entries (rejected).** The existing prototype carries
  current and next committee-and-stake entries beside the MMR root.
  Everything is provable from the MMR against the signed root, and the
  extra entries only bloat every vote and justification. The payload is
  the MMR root alone, the vanilla Snowbridge-style light-client model.
- **Handover as a mandatory inherent extrinsic (rejected).** Redundant.
  The header digest already makes the same data consensus-enforced every
  block with zero new validity rules. The inherent would have added a hard
  fork and a novel stall mode, and it still could not carry the
  signatures, which are the only part that cannot be chain content.
- **Succinct (ZK) quorum verification (deferred, not rejected).** A ZK
  proof of the quorum check would attest exactly the signature checks the
  script performs, so it changes nothing the contract trusts, but it adds
  proof-system assumptions (setup ceremony, soundness, circuit fidelity).
  A quorum of roughly 100 distinct signers fits one transaction today, so
  the saving does not yet justify those assumptions. If committee growth
  outruns the transaction budget, succinct verification can replace
  direct checking without changing the trust model.
- **BEEFY vs GRANDPA.** BEEFY exists for exactly this use: secp256k1 ECDSA
  signatures that are cheap to verify on foreign chains, an MMR payload
  built for inclusion proofs, and a committee commitment built for
  light-client handover. GRANDPA justifications (ed25519, shaped as vote
  sets) are strictly more expensive to verify on Cardano with no
  difference in trust model: the residual risk, a dishonest two-thirds
  signing a chain that honest nodes reject, is identical under either.

## Accountability (sketch, non-normative)

There is no slashing, since stake is Cardano delegation. Instead:

- **Midnight ban list.** BEEFY equivocation proofs (`DoubleVotingProof`,
  `ForkVotingProof`) are objective and attributable. A reporting extrinsic
  feeds a ban list consulted in candidate filtering. SPO stake is sticky
  and bound to registered keys, so exclusion is hard to circumvent.
- Permissioned members are governed by direct removal.

## Open items

- **Cardano-side submitter.** Carrying justifications to Cardano is
  permissionless, but the incentive and fee source are unspecified. The
  rewards batcher (see the Block Production Rewards MIP), which already
  needs each session's commitment landed, is the natural candidate.
- **Contract handover encoding.** The ordering above is fixed; concrete
  redeemer and datum encoding belongs to the contract implementation.
- **On-chain misbehavior response.** Whether the light client should
  react to committee misbehavior on Cardano itself, for example a
  redeemer that freezes the contract when shown two conflicting
  quorum-signed commitments for the same block, possibly behind a
  challenge window. Undecided; needs design and cost analysis.

## Path to Active

1. Registration: extend the Cardano-side candidate registration and its
   tooling with the `beef` key, require it in candidate filtering, and
   re-register existing SPOs before BEEFY voting activates.
2. Node: enable BEEFY voting (session keys, candidate keys, storage
   migration), the deduplicated committee commitment, and the
   MMR-root-only payload.
3. Equivocation reporting extrinsic and ban-list filtering.
4. The Cardano light-client contracts: seat-sum quorum, handover state
   machine, Keccak MMR proofs, and the single-UTxO reference-input
   state.
5. Relay rework to submit the justifications.
6. Testnet rotation soak across many sessions, and an audit.

Acceptance: multiple committee rotations, including a membership change,
verified end-to-end on a public testnet by the light client, with
adversarial submissions (insufficient seats, stale set, non-successor set,
replayed signer) rejected on-chain.

## Backwards Compatibility

Midnight-side changes ship as a standard runtime upgrade (a session-key
addition with a storage migration); there is no block-format fork. The
prototype payload and light-client format will be retired and the Cardano
contracts reworked.

## Security Considerations

- **The committee attests to its own chain.** A dishonest two-thirds of
  seats can sign a root for a chain that honest nodes reject; the light
  client verifies signatures, not chain validity. This is inherent to any
  committee-signature bridge, but corrupting two-thirds of seats also
  breaks GRANDPA itself, so the bridge adds no trust beyond the chain.
  The consequence scales with the value the bridge gates: anything that
  releases tokens against bridge-verified facts must cap its exposure, as
  the Block Production Rewards MIP does with its flow-limited reserve
  release. Any future transfer mechanism needs an equivalent cap.
- **Equivocation**, signing a divergent root while following the canonical
  chain, is provable and attributable (see Accountability). A
  self-consistent full fork is chain takeover and out of scope here.
- **Liveness.** More than a third of seats withholding votes freezes the
  bridge and rewards until participation resumes. A stall, never a theft,
  and it self-heals: BEEFY never skips a mandatory round, so the round
  stays open and late votes under the same `validator_set_id` complete it
  whenever enough of that session's set returns. The unrecoverable case
  is a session set that permanently lost more than a third of its seats,
  or an off-protocol authority reset (`note_stalled`). Recovery there is
  a governed re-registration of the committee commitment on Cardano,
  under the Council and Technical Committee's existing contract-update
  authority.

## References

- Block Production Rewards MIP (companion; consumes this bridge)
- Polkadot BEEFY protocol documentation
- Snowbridge (the Polkadot-to-Ethereum bridge using the same BEEFY
  light-client model)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed
under the Apache License, Version 2.0. Submission requires agreement to the
Midnight Foundation Contributor License Agreement, which includes the
assignment of copyright for your contributions to the Foundation.
