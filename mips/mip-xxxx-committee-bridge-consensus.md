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
Replaces: none
License: Apache-2.0
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
committee and its successor. The BEEFY signatures live in off-chain
justifications: votes are gossiped between nodes, and each node stores the
aggregated justification alongside the finalized block in its database and
serves it over RPC. A relay carries justifications to Cardano.

### MMR leaf

Each leaf (version 0.1) contains `parent_number_and_hash`, which makes
every block header and anything it commits to provable by inclusion proof;
`beefy_next_authority_set`, the successor committee commitment enabling
handover by induction; and `leaf_extra`, which is empty. A version bump is
the extension point if Cardano ever needs extra per-block data.

### Committee commitment (deduplicated, seat-weighted)

Midnight committees are selected by Ariadne, the Partner Chains selection
algorithm, with repetition: one member can hold multiple seats, and seats
are the stake weight (see Rationale). Rather than one Merkle leaf per seat
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

**Handover.** Every leaf of session N carries the commitment of set N+1.
The contract must extract set N+1 from a leaf proven against a root signed
by set N *before* it accepts any signature from set N+1; a session's own
mandatory justification can never bootstrap its own set. The genesis
committee is installed at deployment. Handovers are consumed in order.

### Rewards coupling

Block-production rewards are paid in NIGHT on Cardano and unlock strictly
in session order: session N's reward data becomes claimable only after
session N's commitment is verified. No consensus rule forces voting;
participation is driven by this coupling, since a committee that fails to
produce justifications delays all reward payments, including its own.
Individual non-participation is handled by monitoring and the ban lever
(see Accountability).

### Keys

Each block producer registers a BEEFY session key: either a dedicated key
(`beef`) or, by default, the candidate's registered cross-chain ECDSA key.
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
  rewards batcher, which already needs each session's commitment landed,
  is the natural candidate.
- **Contract handover encoding.** The ordering above is fixed; concrete
  redeemer and datum encoding belongs to the contract implementation.
- **On-chain misbehavior response.** Whether the light client should
  react to committee misbehavior on Cardano itself, for example a
  redeemer that freezes the contract when shown two conflicting
  quorum-signed commitments for the same block, possibly behind a
  challenge window. Undecided; needs design and cost analysis.

## Path to Active

### Acceptance Criteria

Multiple committee rotations, including a membership change, verified
end-to-end on a public testnet by the light client, with adversarial
submissions (insufficient seats, stale set, non-successor set, replayed
signer) rejected on-chain.

### Implementation Plan

1. Node: enable BEEFY voting (session keys, candidate keys, storage
   migration), the deduplicated committee commitment, and the
   MMR-root-only payload.
2. Equivocation reporting extrinsic and ban-list filtering.
3. The Cardano light-client contracts: seat-sum quorum, handover state
   machine, Keccak MMR proofs.
4. Relay rework to submit the justifications.
5. Testnet rotation soak across many sessions, and an audit.

## Backwards Compatibility Assessment

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
- **Equivocation**, signing a divergent root while following the canonical
  chain, is provable and attributable (see Accountability). A
  self-consistent full fork is chain takeover and out of scope here.
- **Liveness.** More than a third of seats withholding votes freezes the
  bridge and rewards until participation resumes. A stall, never a theft.

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
