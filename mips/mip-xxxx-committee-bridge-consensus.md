---
MIP: "xxxx"
Title: Committee Bridge Consensus Integration
Authors:
  - luminight99
  - MicroProofs
Status: Draft
Category: Core
Created: 2026-07-31
Requires: none
Replaces: none
MPS: MPS-0033
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
chain. A *light client* on Cardano tracks the Midnight BEEFY committee by
induction: each committee vouches for its successor. Committee-signed
Merkle Mountain Range (MMR) roots then let Cardano verify Merkle inclusion
of any Midnight block, with no proof system in the loop.
The design is standard Polkadot BEEFY, under which all bridge data is
consensus-enforced chain content through the block header digest. This MIP
specifies that model plus two Midnight-specific pieces: a deduplicated
committee commitment sized for Cardano transaction limits, and a quorum
counted in stake-weighted seats. It also specifies how signed commitments
reach Cardano. A *data pump* in every consensus node submits them, and a
*funding pool* on Cardano pays the fee under a recorded cap. No node
holds a fee-paying wallet.

This MIP enforces the MMR leaf layout, the committee commitment, the
signed-commitment format and the light-client validity rules. It also
enforces the funding-pool spend rules, the bound on the committee-size
parameter, the `beef` key requirement and the six-hour epoch. It leaves
committee selection to Ariadne (Partner Chains), and BEEFY voting and
justification storage to `sc-consensus-beefy`. Candidate registration
stays with the Partner Chains registration flow, and every reward rule
with the Block Production Rewards MIP.

## Motivation

The Block Production Rewards MIP pays NIGHT on Cardano justified solely by
committee-signed Midnight facts. The attack is a relay operator who forges
a committee fact, and the cost is NIGHT paid on it. That is closed only if
the committee and the data it signs are consensus-validated chain content,
not a side channel from whoever runs a relay. It is closed only if
verification also fits Cardano's execution and transaction-size budgets. Beyond rewards, this bridge is the
trust foundation for any flow that must prove a Midnight fact on Cardano.
The bridge carries only the committee lineage and the signed MMR root. It
ferries no other information: each consumer proves its own fact against that
root, in its own transaction, and the bridge tells the consuming contract
which signatures to trust.

## Specification

### Notation

- `Keccak-256(x)`: the 32-byte Keccak-256 hash of `x`, the pre-FIPS
  Ethereum variant. The Plutus V3 builtin `keccak_256` computes it.
- `a ‖ b`: byte concatenation.
- `uN LE`: an unsigned N-bit integer, little-endian, as SCALE encodes it.
- `SCALE(x)`: the encoding of `x` under Parity's
  [SCALE codec](https://docs.polkadot.com/polkadot-protocol/parachain-basics/data-encoding/).
  Field order is struct-definition order.
- *key*: a 33-byte compressed secp256k1 public key.
- *signature*: 64 bytes, `r ‖ s`, low-S. BEEFY produces 65-byte
  recoverable signatures; the submitter drops the recovery byte.
  `verify_ecdsa_secp256k1_signature(key, hash, sig)` is the Plutus builtin
  over a 32-byte message hash; it rejects a high-S `s`.
- *seat*: one unit of committee membership. A committee is a list of keys
  with repetition; a key's seats are its multiplicity.
- `merkle_root(leaves)`: the binary Merkle root over a list of byte
  strings. Level 0 is `Keccak-256(leaf)` for each leaf, in list order. Each
  next level pairs nodes left to right, `Keccak-256(left ‖ right)`; an
  unpaired last node is promoted unchanged. The root is the single node of
  the top level. This is `binary_merkle_tree::merkle_root` in
  `polkadot-sdk` with the Keccak hasher; that function is normative.
- *multiproof*: a tree, as Plutus `Data`, that proves a set of leaves
  against a `merkle_root`. A node `N` is a list of one of five shapes.
  `[L]` is one leaf. `[L1, L2]` is two leaves. `[H, N]` or `[N, H]` is a
  32-byte sibling hash `H` beside a subtree. `[N1, N2]` is two subtrees.
  A leaf `L` is a 37-byte string and appears bare only in `[L]` and
  `[L1, L2]`; a leaf beside a hash is `[[L], H]` or `[H, [L]]`. The hash
  of `[L]` is `Keccak-256(L)`; the hash of any pair is
  `Keccak-256(hash(left) ‖ hash(right))`. The proof's leaves are read in
  tree order, left to right.
- *MMR proof*: for the leaf at index `i` of an MMR with `n` leaves, the
  `items` of `LeafProof` as `pallet-mmr` builds them. It is verified as
  `mmr-lib` 0.8.2 `MerkleProof::calculate_root` does, with the Keccak-256
  merge `Keccak-256(left ‖ right)`; that function is normative. The
  items are the path siblings, then the bagged right peaks, then the
  left peaks (see Test vectors).
- `required(seat_count, n, d)`: the seat quorum,
  `seat_count − ⌊(seat_count − 1) × (d − n) / d⌋`, integer division
  rounded down. At `n/d = 2/3` this is `seat_count − ⌊(seat_count − 1) / 3⌋`,
  the BEEFY threshold in `sc-consensus-beefy`.

Where the node or the contracts and this MIP disagree, the text below
governs and the code is a bug; the Backwards Compatibility Assessment
records the known differences.

### Consensus enforcement via header digest

Every block, `pallet-mmr` appends a *leaf* to an MMR (Keccak-256) and
deposits the new root as a consensus digest in the block header, so the
root is chain content and commits to every prior block and, through the
leaf, to the committee and its successor. Committee selection is chain
content the same way: the runtime validates Ariadne's result in every
block (Ariadne is the Partner Chains selection algorithm, cited in
References; this MIP does not enforce it).

The committee's ECDSA signatures over the commitment (see Signed
commitments and sessions) are the one part that is not chain content. They
live in off-chain *justifications*, produced once per session on the
mandatory round. Votes are gossiped between nodes; each node stores the
aggregated justification alongside the finalized block in its database and
serves it over RPC. The node's data pump carries justifications to Cardano
(see Data pump).

### MMR leaf

The leaf appended in block `b` is `MmrLeaf` of `pallet-beefy-mmr`, SCALE
encoded, and its hash in the MMR is `Keccak-256(SCALE(leaf))`:

| Field | Bytes | Encoding | Value |
|---|---|---|---|
| `version` | 1 | `major << 5 + minor` | `0x00`, version `(0, 0)` |
| `parent_number` | 4 | `u32 LE` | `b − 1` |
| `parent_hash` | 32 | raw | hash of block `b − 1` |
| `beefy_next_authority_set` | 44 | see Committee commitment | the successor committee |
| `leaf_extra` | 1 | `SCALE(Vec<u8>)`, empty | `0x00` |

The leaf appended in block `b` has index `b − 1`. The MMR whose root is
in block `b`'s digest therefore has `b` leaves, and its last leaf commits
block `b − 1`. `parent_number` and `parent_hash` make every block header,
and anything it commits to, provable by inclusion proof; the consuming
MIP defines the header and extrinsic proofs it needs. `leaf_extra` is
empty; a version bump is the extension point if Cardano ever needs extra
per-block data.

`beefy_next_authority_set` is the successor committee. Every leaf of
session N names committee N+1 (`pallet-beefy-mmr`,
`BeefyNextAuthorities`); this MIP relies on that and does not enforce it.

### Committee commitment

The committee is Ariadne's selection: a key list with repetition, in
which a key's seats are its stake weight (see Rationale and References).
Rather than one Merkle leaf per seat (the upstream default), the
commitment deduplicates keys to fit Cardano transaction limits:

1. Group the committee by BEEFY key and count seats per key.
2. Sort ascending by key bytes. One Merkle leaf per distinct key:
   `key (33 bytes) ‖ seats (u32 LE)`, 37 bytes.
3. `keyset_commitment = merkle_root(leaves)`.
4. `seat_count` (the upstream `len` field) is the **total seat count**,
   with repetition, not the distinct key count. It is the quorum
   denominator. It stays `u32`, the upstream width.

The *commitment* of a committee is
`validator_set_id (u64 LE) ‖ seat_count (u32 LE) ‖ keyset_commitment (32 bytes)`,
44 bytes regardless of committee size. `validator_set_id` is the upstream
`id` field; it increments each session.

### Signed commitments and sessions

A *session* is the term of one committee. Its first block is the block
whose header carries `ConsensusLog::AuthoritiesChange` naming the
committee. `pallet_beefy` issues that change at every Midnight epoch,
key list changed or not, so a session is an epoch and this MIP uses the
two words for the same interval. The BEEFY payload is the MMR root alone
(the upstream `mh` payload). Validators sign
`Keccak-256(SCALE(Commitment))`, where `Commitment` is
`{ payload, block_number, validator_set_id }` and the encoding is 48 bytes:

| Field | Bytes | Value |
|---|---|---|
| payload length | 1 | `0x04` (one entry) |
| payload id | 2 | `mh` |
| root length | 1 | `0x80` (32 bytes) |
| `mmr_root` | 32 | the MMR root in block `block_number`'s digest |
| `block_number` | 4 | `u32 LE` |
| `validator_set_id` | 8 | `u64 LE` |

A *justification* is a commitment with signatures from at least
`required(seat_count, 2, 3)` seats of the committee `validator_set_id` names. A
mandatory signed round on the first block of each session, signed by the
incoming committee, guarantees one justification per session as long as two
thirds of seats vote.

### Cardano-side verification

The light client is a Plutus V3 contract. It holds the commitment of the
current committee and of its successor. A submitter presents an *update*
as the redeemer, one Plutus `Data` constructor with these fields in
order:

| Field | Type | Meaning |
|---|---|---|
| `mmr_root` | 32 bytes | the signed root |
| `block_number` | `Int` | the signed block |
| `validator_set_id` | `Int` | the signing committee |
| `signatures` | list of 64 bytes or empty | one per multiproof leaf, in tree order; empty for a leaf that is not a signer |
| `leaf` | `(version, parent_number, parent_hash, next: commitment, extra)` | the leaf of block `block_number` |
| `mmr_proof` | list of 32 bytes | the MMR proof of `leaf` |
| `multiproof` | tree | the signers' leaves against `S.keyset_commitment` |

Every constructor, in the redeemer and in the datum, has index 0;
integers are `Int` and byte strings are `ByteString`. The contract
re-encodes `leaf` to its 82 SCALE bytes for rule 8.

Validity is by exclusion: the update is accepted if no rule below fails.
The state it reads is the light-client datum (see Light-client state)
and the *threshold* `(numerator, denominator)` from the threshold UTxO.

0. Fail if the transaction does not produce exactly one output at the
   script address carrying the NFT, at least the input ADA, and the datum
   computed below.
1. Fail if `block_number ≤ latest_height`.
2. Fail if `validator_set_id` is neither `current_committee.validator_set_id`
   nor `next_committee.validator_set_id`. Let `S` be the matching
   commitment.
3. Fail if the multiproof's root is not `S.keyset_commitment`.
4. Fail if the multiproof's leaves are not in strictly increasing key
   order.
5. Fail if `signatures` has a different length than the multiproof's
   leaves, or if signature `i` is neither empty nor a valid signature
   under the key of leaf `i` over `Keccak-256(SCALE(Commitment))`.
6. Fail if the sum of the seats of the leaves whose signature is not
   empty is less than `required(S.seat_count, numerator, denominator)`.
7. Fail if the presented leaf's `parent_number ≠ block_number − 1`.
8. Fail if the MMR proof does not prove `Keccak-256(SCALE(leaf))` at
   index `block_number − 1` of an MMR with `block_number` leaves against
   `mmr_root`.
9. Let `L` be the leaf's `beefy_next_authority_set`. Fail if
   `L.validator_set_id` is neither `next_committee.validator_set_id` nor
   `next_committee.validator_set_id + 1`.
10. Fail if `validator_set_id = next_committee.validator_set_id` and
    `L.validator_set_id ≠ next_committee.validator_set_id + 1`.

Otherwise the datum becomes: `latest_mmr_root ← mmr_root`,
`latest_height ← block_number`; and if
`L.validator_set_id = next_committee.validator_set_id + 1`, the update is
a *handover*: `current_committee ← next_committee`,
`next_committee ← L`. All other fields are unchanged.

Rules 2, 9 and 10 are the induction. Committee N+1 can sign only after a
root signed by committee N proved a leaf that names N+1; a session's own
mandatory justification can never bootstrap its own committee. Handovers
are consumed in order.

**Bootstrap.** The first committee is the base case of the induction and is
assumed honest. Governance deploys the light client once, its NFT minted
by a one-shot policy, with an initial datum as follows.
`beefy_activation_block` is the block at which BEEFY voting starts.
`current_committee` is the commitment of the committee active at that
block, under its live `validator_set_id`, call it `c`. `next_committee`
is committee `c + 1`, queued at that block.
`latest_height = beefy_activation_block − 1`, and `latest_mmr_root` is the
MMR root in the digest of block `beefy_activation_block − 1`. `max_fee`
completes the datum. The first funded update is the mandatory
justification of session `c + 1`, which hands over to `c + 2`; the
activation block's own justification advances only the root and is
unfunded. Every field but `max_fee` is a function of public Midnight
state, so anyone can recompute the datum and compare it with the
deployed one before relying on the light client. Every later committee
reaches Cardano by handover alone.

### Committee size

Ariadne's committee-size parameter is the *D-parameter*
`(num_permissioned_candidates, num_registered_candidates)`, held in
`pallet_system_parameters` on Midnight and changed by the governance
extrinsic `update_d_parameter`. The seat total of a session is the sum of
the two. `signer_cap` is a runtime constant: the largest committee of
distinct single-seat members whose quorum update fits one Cardano
transaction, signer entries and multiproof included, in size and in
execution budget. It follows from the update layout above and is fixed by
measurement. At 100 distinct keys and a 67-key quorum the submission is
about 10 KB (estimate, 2026-09). The limit it fits under is `maxTxSize`,
the Cardano protocol parameter: 16,384 bytes on mainnet as of 2026-09,
and not a rule of this MIP.

11. `update_d_parameter` fails if
    `num_permissioned_candidates + num_registered_candidates > signer_cap`.

Every seat can go to a different pool, so this is the one bound under
which no session can produce a handover that a single transaction cannot
carry.

### Light-client state

The light client is a single UTxO, identified by an NFT, whose datum is
the entire bridge state:

| Field | Type | Meaning |
|---|---|---|
| `latest_mmr_root` | 32 bytes | the most recent quorum-verified MMR root, exactly what the committee signed |
| `latest_height` | `Int` (a `u32`) | the Midnight block number the root was signed at |
| `beefy_activation_block` | `Int` (a `u32`) | the first Midnight block at which BEEFY voting is active; carried unchanged, no rule reads it |
| `current_committee` | `(validator_set_id: Int, seat_count: Int, keyset_commitment: 32 bytes)` | the committee authorized to sign now |
| `next_committee` | same | its successor, taken from a proven leaf |
| `max_fee` | `(base: Int, per_signer: Int)` lovelace | the fee cap for an update that the funding pool pays for |

Fields are in this order in one Plutus `Data` constructor. Seat counts
need no state of their own: `seat_count` inside each commitment is the
quorum denominator. An update carries `max_fee` forward unchanged, so no
submitter can raise it.

A second UTxO, the *threshold UTxO*, holds `(numerator, denominator)` and
is read as a reference input; its initial value is `(2, 3)`. It is
identified by a second NFT of the same one-shot policy, a parameter of
the light client; its spending rules are those of
`validators/thresholds.ak` in `midnight-reserve-contracts`. Governance
sets `max_fee`, the threshold, and the bootstrap datum, and re-registers
the committee commitments if the induction ever breaks (see Security
Considerations). Governance is the Council and Technical Authority
multisig of `midnight-reserve-contracts` (`gov_auth`), acting through its
two-stage upgrade, which moves the state to a new script; the light
client itself has the one redeemer above. That path is not enforced by
this MIP.

Consumers never spend the light client. A downstream contract, such as
the rewards contract, reads the datum as a reference input and verifies
a Keccak-256 MMR inclusion proof against `latest_mmr_root` itself. Since
every root commits to all prior blocks, the latest root proves any past
Midnight block. Consumers do not contend
with each other. They contend with updates and with nothing else. An
update spends the UTxO, so a consumer transaction built against the datum
it replaced fails Cardano's first validation phase. It costs nothing and
is rebuilt against the new datum.

### Funding pool

Each flow that the data pump serves has one *funding pool* of its own. The
bridge's pool pays the ADA fee of a light-client update, so the fee does
not come from the party that submits the update. The pool is one UTxO of
ADA at a script address parameterized by the light-client NFT policy id.
Anyone may add to it, and block producers are the expected funders (see
Why does no rule force voting?). Let `debit = pool_in − pool_out`, let `s` be the
number of leaves in the update's multiproof, and let
`cap = max_fee.base + max_fee.per_signer × s`. A transaction that spends
the pool is valid if no rule fails:

12. Fail if the transaction does not produce exactly one output at the
    pool address.
13. If `debit ≤ 0` the transaction is a top-up; no further rule applies.
14. Fail if the transaction does not spend the light-client UTxO with a
    valid update.
15. Fail if that update is not a handover: the light-client input datum
    and output datum have the same `next_committee.validator_set_id`.
16. Fail if `debit > fee`, the transaction's declared fee.
17. Fail if `debit > cap`.

A fee above the cap is allowed; the submitter's own inputs cover the
difference. Both amounts of `max_fee` are measured in advance (see
Rationale). An empty pool stalls updates and loses nothing (see Security
Considerations). A later flow reuses this construction with a pool of
its own; the Block Production Rewards MIP has one.

### Data pump

Every consensus node runs a *data pump*: one component of the Midnight
node that carries periodic work to Cardano automatically, with no
fee-paying wallet and no operator action. The component is modular. Each
periodic job is a *module*. Every module's transaction is paid from the
funding pool of its flow. The contract a module serves records the
`max_fee` for its job, in its state or in a parameter UTxO it reads. This
MIP defines the component and its first module, the light-client update.
The Block Production Rewards MIP adds the reserve release and epoch load
modules, and any later job is one more module.

All modules share four parts:

- **Configuration.** The operator configures a Cardano submit endpoint.
  There is no fee-paying key to configure.
- **Cardano state.** The pump reads the UTxOs a transaction spends, and
  the `max_fee` in force, from the Cardano data source the node already
  uses. These reads are not consensus inputs, so they MAY follow Cardano's
  tip rather than the stability window. A read that a rollback overtakes
  yields a failed transaction and a later attempt, nothing worse.
- **Transaction builder.** The pump includes a Cardano transaction
  builder. It MUST set the fee the transaction needs, MUST NOT set it
  above the cap, and draws it from the funding pool.
- **Ordered submission, one winner.** A module works through its *items*
  in order and reads from the contract state, its *marker*, which items
  are done. Every node attempts the same item, one submission lands, and
  the rest fail on the spent UTxO. A node that sees the item done MUST
  move to the next one and MUST NOT retry it. A node that finds the pool
  unable to pay SHOULD try again later.

For the light-client module the item `k` is the mandatory justification
of session `k`: the justification of the block whose header carries
`ConsensusLog::AuthoritiesChange` naming set `k`, signed by set `k`. Its
leaf names set `k + 1`, so landing it makes `k + 1` the
`next_committee`. The pump reads the justification from the node's
block-justification store under engine id `BEEF`, and adds the leaf and
MMR proof. The marker is the light-client datum: item `k` is done when
`next_committee.validator_set_id > k`, and the next item is
`k = next_committee.validator_set_id`. No state is added for the pump.
When Cardano is several sessions behind, the module MUST submit the oldest
missing session first, because handovers are consumed in order, and
continue until the light client is current. One funded update per session
sets the freshness of the root. A Midnight block becomes provable on
Cardano when the next session's update lands, at most one session after
the block.

### Keys

Each block producer registers a dedicated BEEFY session key, key type
`beef`, in the Partner Chains candidate registration. Its `keys` map
carries the key beside the Aura and GRANDPA keys; the format is the
[Partner Chains registration guide](https://github.com/input-output-hk/partner-chains/blob/master/docs/user-guides/registered.md)'s.
Permissioned members supply the same key through the governance-managed
permissioned list. There is no fallback to the candidate's cross-chain
ECDSA key. BEEFY signing requires the key in the node's hot keystore, and
the cross-chain key is the candidate's registration identity, which must
not live there.

18. A candidate, registered or permissioned, without a `beef` key is
    excluded from candidacy from `beefy_activation_block`.

Misbehavior attribution maps the BEEFY key to the candidate through the
registered keys.

### Epoch length

The Midnight epoch is raised from 30 minutes to six hours: 3,600 slots
at the six-second block interval. If the block interval ever changes,
the slot count is scaled so that the epoch stays six hours (see
Appendix A). One mandatory BEEFY round and one light-client update per
session is the bridge's steady-state cost. Per-epoch settlement of
rewards on Cardano (digest, release, fold) is practical only at that
cadence. The change is a runtime upgrade with a storage migration of
session- and epoch-keyed state, landed together with BEEFY activation.

## Rationale

### Why not explicit stake weights in the commitment?

A stake-weighted quorum conflicts with the standard Polkadot BEEFY model:
upstream voting and justification verification count seats, so explicit
stakes would require forking `sc-consensus-beefy`. They would also
double-count stake that seat repetition already expresses. It also buys
nothing, since GRANDPA finality is already a two-thirds count over the
same seats, so a bridge quorum cannot exceed the chain's own security.
Seats *are* the stake weight under Ariadne; the bridge simply inherits
the chain's trust model.

### Why may a leaf carry no signature?

So the relay proves any subset of the committee with one tree shape: a
leaf beside a signer can be presented as itself with an empty signature
and counts no seats. The cap counts leaves, so surplus leaves cost the
submitter, and a signers-only proof stays the smallest.

### Why sort the commitment by key bytes?

A sorted tree gives the contract a cheap uniqueness check. Signers must
appear in strictly increasing key order (rule 4), so no key can be
counted twice, and no set of seen keys has to be kept. Any total order
would do; byte order needs no decoding.

### Why one leaf per distinct key?

Upstream `pallet-beefy-mmr` builds the root over one leaf per authority
entry, so a committee of `s` seats costs `s` leaves and `s` signer
entries in a Cardano update. Grouping by key and carrying the seat count
in the leaf keeps the update proportional to distinct signers, which is
what the transaction budget bounds. The pallet has no hook for it; the
grouped root is a Midnight change to the pallet, recorded in Backwards
Compatibility.

### Why does no rule force voting?

The Block Production Rewards MIP pays block producers in NIGHT on Cardano
against facts proven through this light client, session by session: its
reward digest is a transaction in a Midnight block, and the leaf of the
following block commits to that block's header, and through the header's
`extrinsics_root` to the transaction. A committee that fails to produce
justifications delays all reward payments, including its own, and while
the light client does not advance, no block producer is paid. That
coupling drives participation, the data pump and the funding pool
without a consensus rule; nothing requires that block producers be the
funders. Individual non-participation is visible to anyone from the
justifications; a response to it is future work (see Security
Considerations).

### Why Keccak-256?

It is the hasher upstream BEEFY uses for the MMR and the authority tree,
because Ethereum verifies it cheaply. Plutus V3 has a `keccak_256`
builtin, so Cardano verifies it at the same cost. A different hasher
would fork the pallets for no gain.

### Why `seat_count − ⌊(seat_count − 1) / 3⌋`?

It is the BEEFY threshold: the largest number of faulty seats `f` such
that `3f + 1 ≤ seat_count`, subtracted from `seat_count`. The contract
computes it from the threshold UTxO's ratio so that governance can move
the ratio without a contract upgrade. At `(2, 3)` the formula reproduces
the node's own quorum exactly. That includes the case where `seat_count`
is a multiple of three, which a plain ceiling gets one seat low.

### Why is the payload the MMR root alone?

The existing prototype carries current and next committee-and-stake
entries beside the MMR root. Everything is provable from the MMR against
the signed root, and the extra entries only bloat every vote and
justification. The payload is the MMR root alone, the vanilla
Snowbridge-style light-client model.

### Why not a mandatory inherent extrinsic for handover?

Redundant. The header digest already makes the same data
consensus-enforced every block with zero new validity rules. The inherent
would have added a hard fork and a novel stall mode. It still could not
carry the signatures, which are the only part that cannot be chain
content.

### Why not a succinct (ZK) quorum proof now?

A quorum of about 100 distinct signers fits one transaction today, so
direct signature checking ships first. Its price is the cap on committee
size below. A ZK proof that a threshold of the committee signed, verified
inside one transaction, replaces the signature list without changing the
trust model and lifts the cap. That makes it the urgent item after this
MIP.

### Why hold the D-parameter under `signer_cap`?

A certificate that lists signatures grows with the signers, and a Cardano
transaction does not. Handovers are verified one against the next. A
single session whose quorum does not fit stops every later handover, and
with it every fact that flows to Cardano, until a larger verifier exists
there. Registration is open to any pool, and in the worst case every seat
goes to a different one, so no seat total above `signer_cap` makes that
impossible. The bound sits on the one place the seat total is set, the
`update_d_parameter` extrinsic, so a parameter above the cap cannot be
recorded. The cost is decentralization: a committee under the cap seats
at most that many distinct pools per session, however many have
registered. It is a property of the certificate form and not of the
design, and the succinct proof removes it.

### Why six hours?

Two costs pull against each other. Every session costs one funded update,
so the pool drains at no more than `cap × sessions per day`: 48 caps a
day at a 30-minute epoch, 4 at six hours. A shorter epoch buys little.
Committee selection reads its Cardano inputs (seat division,
registrations, stake, epoch nonce) once per five-day Cardano epoch.
Shorter sessions reselect from the same candidates at the same weights
with a new random draw: 240 times per Cardano epoch at 30 minutes, 20 at
six hours. The floor comes from the consumers. A Midnight fact becomes
provable on Cardano at the next session's update, and rewards settle once
per epoch, so a five-day epoch would hold every payout for days. Six
hours cuts the drain by twelve and keeps four settlements a day. A pool
of `P` lovelace lasts at least `P ÷ (4 × cap)` days.

### Why BEEFY and not GRANDPA?

BEEFY exists for exactly this use. It gives secp256k1 ECDSA signatures
that are cheap to verify on foreign chains, an MMR payload built for
inclusion proofs, and a committee commitment built for light-client
handover. GRANDPA justifications (ed25519, shaped as vote sets) are
strictly more expensive to verify on Cardano with no difference in trust
model. The residual risk, a dishonest two-thirds signing a chain that
honest nodes reject, is identical under either.

### Why is an ADA cost never compensated in NIGHT?

Work on Cardano costs ADA. To repay that cost in NIGHT would make every
submitter price NIGHT against ADA, and the protocol would carry an implied
exchange rate. The Block Production Rewards MIP applies the same
principle: ADA covers the ADA cost, and any NIGHT fee is a separate
reward for effort, unrelated to any ADA spent. A light-client update is
almost entirely ADA cost, so ADA pays for it and no NIGHT payment is
attached. The incentive to keep it running is the reward coupling (see
Why does no rule force voting?).

### Why a funding pool and not an operator wallet?

Block producers could pay the update fee themselves. But every consensus
node is to push updates automatically, and with a connected wallet that
means software that spends an operator's ADA unattended on every node. A
funding pool gives the automation without the wallet. The node holds no
key that controls funds, and the ADA set aside for the bridge can be
spent on the bridge alone. Each flow funds itself: a funder's ADA pays for
the flow it was given to, and a later flow cannot spend what was set aside
for this one. The nodes already hold every justification and have the
reason to deliver it, so the submitter is the node.

### Why a fee cap calculated in advance, per signer?

On Cardano the transaction creator sets the fee, and fees go to the reward
pot that SPOs and their delegators share. A submitter that is an SPO gets
part of any fee back, so a pool with no cap could be drained by padded
fees at a profit. A cap can be fixed in advance because script cost is
deterministic: an update runs the same checks over inputs of the same
magnitude each time, so its fee can be measured before deployment. The
one input that moves the cost is the number of signers; each adds a
37-byte leaf, a 64-byte signature and one signature check. A flat amount
would cover the largest quorum and leave a gap under every smaller one.
A base amount plus an amount per presented signer is a slight
overestimate at any committee size. Surplus signers raise the cap and
the real fee together, so they buy nothing.

### Why does the pool pay for one update per session?

The light client accepts any strictly newer quorum-signed root, and BEEFY
finalizes far more often than once per session. A pool that paid for every
valid update could be emptied at no cost to the party that submits them,
with honest fees. The update that advances the committee pair happens once
per session and is the one the bridge cannot live without, so that is the
update the pool pays for. The price is freshness. The rewards digest of
epoch E lands at or after the first block of session E + 1. The root that
proves it is therefore the one from session E + 2. Payouts settle one epoch
later than a continuously updated root would allow. A delayed payout loses
nothing.

### Why one pump in the node and not a service per job?

Work that block producers should do frequently and as a matter of course
belongs in one expandable component of the node. Every module has the same
structure. It has a funding pool for its flow, a `max_fee` recorded in the
same kind of place, and a marker that shows which items are done. It handles
an empty pool, or an item that another node already delivered, the same way.
A new job reuses the configuration, the transaction builder and the
submission rule, and adds only what is specific to its contract.

### Open questions

- **Collateral without a wallet.** Cardano requires every transaction
  that runs a Plutus script to name collateral held at a key address and
  to carry that key's signature. Collateral return does not relax this.
  The ledger takes collateral only when a script fails, which a node that
  validated its own transaction does not incur, but the funding pool
  cannot supply it. How the data pump provides collateral while holding
  no fee-paying wallet is undecided.
- **Where `max_fee` lives.** In the light-client datum as written, in a
  parameter UTxO beside the threshold, or in the funding pool itself.
  Decided with the Aiken contract.
- **On-chain misbehavior response.** Whether the light client should
  react to committee misbehavior on Cardano itself. One form is a
  redeemer that freezes the contract when shown two conflicting
  quorum-signed commitments for the same block, possibly behind a
  challenge window. Undecided; needs design and cost analysis.

## Path to Active

### Acceptance Criteria

- Multiple committee rotations, including a membership change, verified
  end to end on a public testnet by the light client.
- The contract tests of the Testing section passing against the
  deployed contracts on that testnet.
- The light client advanced across those rotations by the data pumps of
  several competing nodes, with every fee paid from the funding pool and
  no fee-paying wallet.
- Every registered candidate carries a `beef` key, and the fallback to
  the cross-chain key is removed before BEEFY voting activates on a
  network whose light client carries value.
- `signer_cap` measured on a committee of distinct single-seat members,
  and `update_d_parameter` shown rejected at `signer_cap + 1` seats.
- The leaf index of block `b` confirmed as `b − 1` against
  `mmr_generateProof` on the target network.
- The deployed light-client datum checked against the datum recomputed
  from Midnight state at `beefy_activation_block`.
- The epoch length change deployed with its storage migration.
- Audit of the light-client contracts complete.

### Implementation Plan

See Implementation.

## Backwards Compatibility Assessment

No re-genesis and no Cardano hard fork; the Midnight side is a runtime
upgrade with a storage migration, and the Cardano side is a new
light-client deployment. Where the node and the contracts are today, read
from `midnight-node` `main` and `lglo/beefy-on-main` and from
`midnight-reserve-contracts` on 2026-09-22, and what changes:

- The `Beefy`, `Mmr` and `BeefyMmrLeaf` pallets are in the runtime on
  `main`, on `polkadot-sdk` `polkadot-stable2606`. The MMR hashes with
  Keccak-256, the leaf version is `(0, 0)`, `leaf_extra` is an empty
  `Vec<u8>`, and the BEEFY key type is `beef` with ECDSA keys. This
  matches the specification.
- The BEEFY payload carries five entries: the MMR root plus current and
  next committee stakes and authority sets. The specification reduces the
  payload to the MMR root alone. The extra entries and the runtime API
  that serves them are to be removed.
- The committee commitment is built with the leaf layout in this MIP
  (`key ‖ seats`). But the count is a little-endian `u64`, every
  validator's seat count is hard-coded to 1, and keys are not
  deduplicated. The specification uses a `u32` count per distinct key.
  The contract's `Leaf` slices a `u64` at the same offset and changes
  with it.
- The `beef` session key is being added on `lglo/beefy-on-main`. That
  branch falls back to the candidate's cross-chain ECDSA key when no
  `beef` key is registered, and its storage migration seeds each
  validator's BEEFY key from the cross-chain key. This MIP requires a
  dedicated `beef` key with no fallback. The fallback is acceptable as a
  migration convenience on test networks and must be removed before BEEFY
  voting activates on a network whose light client carries value.
- The D-parameter is read from `pallet_system_parameters`, not from a
  Cardano UTxO. Rule 11 is a new check in `update_d_parameter`. Every
  shipped network today has zero registered seats.
- Equivocation reporting is a no-op in the runtime's `BeefyApi`, and there
  is no ban list.
- `midnight-reserve-contracts` has the light client
  (`validators/committee_bridge.ak`). It has the datum of this MIP less
  `max_fee`, the threshold UTxO (`BeefyThreshold`), the multiproof shape
  of the Notation, and the handover rule keyed on the leaf. Its quorum is
  `⌈seat_count × numerator / denominator⌉`, one seat below rule 6 when
  `seat_count` is a multiple of three; it changes to `required`. It
  accepts an empty signature for a non-signer leaf, as rule 5 does. The
  funding pool and `max_fee` do not exist yet.
- A relay (`midnight-beefy-relay`) subscribes to justifications, builds
  the signer proofs with Keccak-256, and encodes them as Plutus data for
  the current light client. It is a standalone binary outside the node,
  and it logs the encoded proof without submitting it. This MIP moves
  submission into the node's data pump, paid from the funding pool. The
  relay's proof building and encoding carry over to the pump's
  light-client module, reworked for the MMR-root-only payload.
- The running node has no path that submits to Cardano. Transaction
  building and Ogmios submission exist only in the Partner Chains
  command-line subcommands the node binary carries. The data pump adds
  the submit path to the node service, and gives the pump a handle on the
  db-sync data source bounded at the tip; its UTxO queries take the
  bounding block from the caller
  ([`db_model.rs`](https://github.com/midnightntwrk/midnight-node/blob/d17a54f8df04195e5dce041f79546a9267daa6b9/partner-chains/toolkit/data-sources/db-sync/src/db_model.rs#L406-L469)).

**Session keys and registration.** Candidates without a registered `beef`
key are excluded from candidacy from `beefy_activation_block`, so every
SPO re-registers before activation. Permissioned members supply keys
through governance configuration. The test-network fallback to the
cross-chain key is a migration convenience only.

**Epoch length.** Raised to six hours in the same runtime upgrade,
together with the storage migration that requires. Anything keyed to
the 30-minute epoch, including the rewards settlement cadence, follows.

**Node operators.** The data pump ships in the node. An operator
configures a Cardano submit endpoint, and no fee-paying wallet is
attached. Collateral is the open point (see Open questions).

**Downstream consumers.** Contracts that read the light client as a
reference input see a single-UTxO datum with the fields in Light-client
state. The prototype's datum gains `max_fee` and is otherwise carried
forward.

## Security Considerations

- **The committee attests to its own chain.** A dishonest two-thirds of
  seats can sign a root for a chain that honest nodes reject; the light
  client verifies signatures, not chain validity. This is inherent to any
  committee-signature bridge, but corrupting two-thirds of seats also
  breaks GRANDPA itself, so the bridge adds no trust beyond the chain.
  The consequence scales with the value the bridge gates. Anything that
  releases tokens against bridge-verified facts must cap its exposure, as
  the Block Production Rewards MIP does with its flow-limited reserve
  release. Any future transfer mechanism needs an equivalent cap.
- **Equivocation**, signing a divergent root while following the canonical
  chain, is provable and attributable through BEEFY's `DoubleVotingProof`
  and `ForkVotingProof`. There is no slashing, since stake is Cardano
  delegation. Permissioned members are removed by governance. A
  reporting extrinsic and a ban list consulted in candidate filtering
  are future work for a separate MIP; this MIP makes equivocation
  provable and attributable, not penalized. A self-consistent full fork
  is chain takeover and out of scope here.
- **Liveness.** More than a third of seats withholding votes freezes the
  bridge and rewards until participation resumes. A stall, never a theft,
  and it self-heals. BEEFY never skips a mandatory round, so the round
  stays open, and late votes under the same `validator_set_id` complete
  it whenever enough of that session's committee returns; later sessions
  queue behind the oldest open round. The unrecoverable case is a
  committee that permanently lost more than a third of its seats, or a
  BEEFY restart by `set_new_genesis`, which starts voting again from a
  new genesis block. Recovery there is a governed re-registration of the
  committee commitments on Cardano, under the authority named in
  Light-client state.
- **Funding pool exhaustion.** An empty pool stalls the light client, and
  rewards with it, until someone adds ADA. No information is lost. Nodes
  keep each session's justification beside the finalized block, the
  mandatory round guarantees one per session, and the latest MMR root
  proves every earlier block. When the pool is refilled, the data pumps
  resume from the oldest missing session, in order.
- **Fee extraction from the funding pool.** A submitter that is an SPO
  gets part of any Cardano fee back through Cardano's own rewards, so a
  padded fee paid by the pool is profit, not griefing. Rule 17 bounds
  the pool's debit by a cap that tracks the real cost per presented
  signer. The most a padded update takes is the gap between the cap and
  the real cost; presenting surplus signers raises both together.
  Rule 15 funds one update per session, so extra submissions cannot
  drain it.
- **An unverifiable handover.** A session whose quorum does not fit one
  transaction would stop the induction for good. Rule 11 holds the seat
  total at or under `signer_cap`, so no such session can occur.
- **The base case.** Everything the light client accepts rests on the
  datum it was deployed with. Every field but `max_fee` is recomputable
  from public Midnight state at `beefy_activation_block`, so a wrong
  deployment is detectable before any value rests on it.
- **Updates that disturb readers.** Each update invalidates the consumer
  transactions in flight against the datum it replaced. They fail at the
  first validation phase at no cost and are rebuilt. A party that submits
  extra updates to disturb readers pays a full fee for each, since the
  pool funds one per session. It can submit no faster than BEEFY
  produces justifications.

## Implementation

1. Registration: extend the Partner Chains candidate registration and its
   tooling with the `beef` key, require it in candidate filtering, and
   re-register existing SPOs before BEEFY voting activates.
2. Node: enable BEEFY voting (session keys, candidate keys, storage
   migration), the deduplicated committee commitment, the
   MMR-root-only payload, and rule 11 in `update_d_parameter`.
3. The Cardano light-client contracts: rules 0 to 10 with `required`,
   the datum with `max_fee`, and the funding pool with rules 12 to 17.
4. The data pump in the node: the modular component, its configuration,
   the Cardano transaction builder, and the light-client module, which
   reuses the relay's proof building and encoding.
5. Rollout: each step on a Midnight devnet against a Cardano test
   network, then the public testnet for a soak across many sessions,
   then audit, then mainnet.

Registration lives in the Partner Chains candidate registration and its
tooling. Steps 2 and 4 live in `midnight-node`, with step 4 drawing on
`midnight-beefy-relay`; step 3 in `midnight-reserve-contracts` (the
committee bridge validators). The Block Production Rewards MIP is the
first consumer of the light client and does not change this design; it
adds the pump's next modules.

## Testing

**Runtime unit tests.** The deduplicated commitment: grouping by key,
ascending order, `key ‖ seats` leaves, and total seat count as the
denominator. The same root from a committee with and without repeated
members. The `beef` key requirement in candidate filtering, and the
storage migration on a populated session committee. Rule 11 at `signer_cap`
and `signer_cap + 1`.

**Contract tests.**

- One rejection per rule 0 to 10 and 12 to 17: an output without the
  NFT, a stale height, a foreign committee, a wrong multiproof root,
  signers out of order, a missing or bad signature, one seat short, a
  leaf for the wrong block, a bad MMR proof,
  a non-successor committee, a next-committee signature without
  handover, a pool
  debit above the cap, a pool debit above the fee, a funded update that
  is not a handover.
- An update accepted with every rule at its boundary.
- A committee of `signer_cap` distinct single-seat members verified in
  one transaction within size and execution budget, which is the
  measurement that fixes `signer_cap`.
- `max_fee` unchanged by an update.
- The deployed datum matching the datum recomputed from Midnight state
  at `beefy_activation_block`.
- A consumer transaction built against a replaced datum failing at the
  first validation phase.

**Data pump.**

- Several nodes racing on one session, one submission landing and the
  others not retrying it.
- A light client several sessions behind caught up oldest first.
- An empty pool stalling the module and a top-up resuming it.
- The fee set to the transaction's need, below the cap.

**End to end.** On a Midnight devnet against a Cardano test network:

- Justifications submitted each session by the data pump, and the light
  client advanced through several rotations including a membership
  change and a Cardano-epoch boundary.
- A downstream contract proving a past block by MMR inclusion against
  the latest root.
- The contract rejections above reproduced on chain.
- A liveness stall with a third of seats withheld and recovery when they
  return.

### Test vectors

Values are fixed when the reference code exists; the inputs and the
expected relations are fixed here.

- **Commitment.** Keys `k1 < k2 < k3` with seats `(1, 2, 1)`:
  `seat_count = 4`; leaves `k1 ‖ 01000000`, `k2 ‖ 02000000`,
  `k3 ‖ 01000000`; `keyset_commitment = Keccak-256(Keccak-256(Keccak-256(l1) ‖ Keccak-256(l2)) ‖ Keccak-256(l3))`.
  The same committee given as the list `[k2, k1, k2, k3]` yields the same
  root.
- **Signed bytes.** `mmr_root = 00…00`, `block_number = 1`,
  `validator_set_id = 0`: the 48 bytes
  `04 6d68 80 00…00 01000000 0000000000000000`, and the hash to sign is
  `Keccak-256` of them.
- **Leaf.** Version `(0, 0)`, `parent_number = 600`, a given
  `parent_hash`, the commitment above, empty extra: 82 bytes
  `00 58020000 <parent_hash> <commitment> 00`, hashed with Keccak-256.
- **Non-signer leaves (rules 5 and 6).** All leaves of a 12-key
  committee presented, 9 with signatures and 3 empty: the seat sum is
  the 9 signers' seats. One leaf fewer than signatures: rejected.
- **Quorum (rule 6), ratio `(2, 3)`.** `seat_count = 10`: 7 seats
  accepted, 6 rejected. `seat_count = 6`: 5 accepted, 4 rejected.
  `seat_count = 3`: 3 accepted, 2 rejected. `seat_count = 1`: 1 accepted.
- **Height (rule 1).** `latest_height = 600`: `block_number = 600`
  rejected, `601` accepted.
- **MMR proof, three peaks.** Leaf under the middle peak `P2` of an MMR
  with peaks `P1, P2, P3`: items are the path siblings, then `P3` (as
  `R`), then `P1`; root = `Keccak-256(Keccak-256(P3 ‖ P2) ‖ P1)`.
- **MMR proof edges (rule 8).** `block_number = 1`: one leaf, no
  siblings, no peaks; the root equals the leaf hash. A leaf that is
  itself a peak (`block_number` a power of two plus one): no siblings,
  peaks only.
- **Bootstrap.** `beefy_activation_block = 1000`, committee `c` active
  there: `latest_height = 999`, `latest_mmr_root` = the digest root of
  block 999, `current_committee = c`, `next_committee = c + 1`. The
  justification of block 1000 is accepted (rule 1) but not funded (rule
  15); the first funded update is session `c + 1`'s.
- **Handover (rules 9 and 10).** Current `id = 4`, next `id = 5`: a
  leaf naming `id = 5` signed by committee 4 accepted without handover;
  a leaf naming `id = 6` signed by committee 5 accepted with handover; a
  leaf naming `id = 5` signed by committee 5 rejected; a leaf naming
  `id = 7` rejected.

## References

- Block Production Rewards MIP (companion; consumes this bridge):
  [midnight-improvement-proposals PR #321](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/321).
- This MIP's proposal PR:
  [midnight-improvement-proposals PR #262](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/262).
- BEEFY primitives in `polkadot-sdk`:
  [`sp-consensus-beefy`](https://github.com/paritytech/polkadot-sdk/tree/660acefe66599a3e54363797007befcb01bd610b/substrate/primitives/consensus/beefy/src)
  (`commitment.rs`, `payload.rs`, `mmr.rs`),
  [`pallet-beefy-mmr`](https://github.com/paritytech/polkadot-sdk/tree/660acefe66599a3e54363797007befcb01bd610b/substrate/frame/beefy-mmr/src),
  [`pallet-mmr`](https://github.com/paritytech/polkadot-sdk/tree/660acefe66599a3e54363797007befcb01bd610b/substrate/frame/merkle-mountain-range/src),
  [`binary-merkle-tree`](https://github.com/paritytech/polkadot-sdk/blob/660acefe66599a3e54363797007befcb01bd610b/substrate/utils/binary-merkle-tree/src/lib.rs),
  [threshold in `sc-consensus-beefy`](https://github.com/paritytech/polkadot-sdk/blob/660acefe66599a3e54363797007befcb01bd610b/substrate/client/consensus/beefy/src/round.rs#L141-L144).
- MMR proof verification, `MerkleProof::calculate_root` in `mmr-lib`:
  [`paritytech/merkle-mountain-range`](https://github.com/paritytech/merkle-mountain-range/blob/864ec208b126606eb5e0ecb5e7e4c538be086f34/src/mmr.rs).
- Session digest and justification store in `polkadot-sdk`:
  [`ConsensusLog::AuthoritiesChange`](https://github.com/paritytech/polkadot-sdk/blob/660acefe66599a3e54363797007befcb01bd610b/substrate/frame/beefy/src/lib.rs#L616-L620),
  [`BEEFY_ENGINE_ID` justification append](https://github.com/paritytech/polkadot-sdk/blob/660acefe66599a3e54363797007befcb01bd610b/substrate/client/consensus/beefy/src/worker.rs#L699-L742).
- Ariadne selection and the D-parameter in Partner Chains (vendored in
  `midnight-node`):
  [`ariadne_v2.rs`](https://github.com/midnightntwrk/midnight-node/blob/d17a54f8df04195e5dce041f79546a9267daa6b9/partner-chains/toolkit/committee-selection/selection/src/ariadne_v2.rs),
  [`DParameter`](https://github.com/midnightntwrk/midnight-node/blob/d17a54f8df04195e5dce041f79546a9267daa6b9/partner-chains/toolkit/sidechain/domain/src/lib.rs#L1165-L1170),
  [`pallet_system_parameters`](https://github.com/midnightntwrk/midnight-node/blob/d17a54f8df04195e5dce041f79546a9267daa6b9/pallets/system-parameters/src/lib.rs).
- Partner Chains candidate registration (`RegistrationData`, `keys`):
  [`domain/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/d17a54f8df04195e5dce041f79546a9267daa6b9/partner-chains/toolkit/sidechain/domain/src/lib.rs#L1041-L1072).
- Light client and governance contracts:
  [`midnight-reserve-contracts`](https://github.com/midnightntwrk/midnight-reserve-contracts)
  (`validators/committee_bridge.ak`, `lib/bridge/`, `validators/gov_auth.ak`).
- Plutus builtins: [CIP-0049 secp256k1](https://cips.cardano.org/cip/CIP-0049),
  [CIP-0101 Keccak-256](https://cips.cardano.org/cip/CIP-0101).
- Snowbridge BEEFY light client (the same model, on Ethereum):
  [`Snowfork/snowbridge`](https://github.com/Snowfork/snowbridge/tree/main/contracts/src).
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

## Appendix A: The block rate, assumed fixed

Midnight produces a block every six seconds, and the six-hour epoch is
3,600 of them. The tokenomics whitepaper anticipates the block interval
changing as the protocol evolves. Epoch duration needs adjusting in that
case, because it follows from the block rate. An epoch is a count of
slots, while the rewards contracts of the Block Production Rewards MIP
release per wall-clock interval. One release funds one epoch only
because 3,600 six-second slots and a six-hour interval are the same
length. Three-second blocks would halve the epoch to three hours and
leave two digests to be folded against each release. Twelve-second
blocks would stretch it to twelve hours and leave releases arriving with
no digest to fold. The remedy is arithmetic: scale the epoch's slot count
by the same factor as the block rate, to 7,200 slots at three seconds or
1,800 at twelve. An epoch then stays six hours. What else a block-rate
change touches on the rewards side is recorded in that MIP. Nothing in
this appendix is normative.

## Acknowledgements

The reviewers of the successive drafts, and the node engineers whose
BEEFY work on `midnight-node` this MIP builds on.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed
under the Apache License, Version 2.0. Submission requires agreement to the
Midnight Foundation Contributor License Agreement, which includes the
assignment of copyright for your contributions to the Foundation.
