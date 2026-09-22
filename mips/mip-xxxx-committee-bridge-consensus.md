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
chain. A light client on Cardano tracks the Midnight BEEFY committee by
induction: each committee vouches for its successor. Committee-signed
Merkle Mountain Range (MMR) roots then let Cardano verify Merkle inclusion
of any Midnight block, with no proof system in the loop.
The design is standard Polkadot BEEFY, under which all bridge data is
consensus-enforced chain content through the block header digest. This MIP
specifies that model plus two Midnight-specific pieces: a deduplicated
committee commitment sized for Cardano transaction limits, and a quorum
counted in stake-weighted seats. It also specifies how signed commitments
reach Cardano: a data pump in every consensus node submits them, and a
funding pool on Cardano pays the fee under a recorded cap, so no node
holds a fee-paying wallet.

## Motivation

The Block Production Rewards MIP pays NIGHT on Cardano justified solely by
committee-signed Midnight facts. That is sound only if the committee and
the data it signs are consensus-validated chain content rather than a side
channel from whoever runs a relay, and only if verification fits Cardano's
execution and transaction-size budgets. Beyond rewards, this bridge is the
trust foundation for any flow that must prove a Midnight fact on Cardano.
The bridge carries only the committee lineage and the signed MMR root. It
ferries no other information: each consumer proves its own fact against
that root, in its own transaction, and the bridge tells the consuming
contract which signatures to trust.

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
The committee's ECDSA signatures over the commitment (see Signed
commitments and sessions) are the one part that is not chain content. They
live in off-chain justifications, produced once per session on the
mandatory round: votes are gossiped between nodes, and each node stores the
aggregated justification alongside the finalized block in its database and
serves it over RPC. The node's data pump carries justifications to Cardano
(see Data pump).

### MMR leaf

Each leaf (version `(0, 0)`, the upstream `MmrLeafVersion` default)
contains `parent_number_and_hash`, which makes every block header and
anything it commits to provable by inclusion proof;
`beefy_next_authority_set`, the successor committee commitment enabling
handover by induction; and `leaf_extra`, which is empty. A version bump is
the extension point if Cardano ever needs extra per-block data. The Block
Production Rewards MIP uses the first field: its reward digest is a
transaction in a Midnight block, and the leaf of the following block
commits to that block's header by hash, and through the header's
`extrinsics_root` to the transaction.

### Committee commitment (deduplicated, seat-weighted)

The committee is what Ariadne, the Partner Chains selection algorithm,
selects: a member list with a seat count per member, with repetition, so
one member can hold multiple seats, and seats are the stake weight (see
Rationale). Ariadne also derives a slot schedule from that list; the
bridge uses the list and not the schedule, so no separate weighted set is
computed for BEEFY. Membership and seat counts are already public, since
Ariadne selection is deterministic over public Cardano data, and the
commitment reveals nothing new. Rather than one Merkle leaf per seat
(the upstream default), the commitment deduplicates keys to fit Cardano
transaction limits:

- Group the committee by BEEFY key (33-byte compressed secp256k1 ECDSA
  public key) and count seats per key.
- Sort ascending by key bytes, with one Merkle leaf per distinct key:
  `pubkey (33 bytes) ‖ seat_count (u32 little-endian)`, the same width as
  the total below.
- `keyset_commitment` is the Keccak-256 binary Merkle root over the leaves.
- `seat_count` (the upstream `len` field) is the **total seat count**, not
  the distinct key count, and is the quorum denominator. It stays `u32`,
  the upstream width; the runtime bounds committees at
  `MaxAuthorities = 10,000` seats.

The commitment `(id: u64, seat_count: u32, keyset_commitment: 32 bytes)`
is 44 bytes regardless of committee size.

### Signed commitments and sessions

A session is the term of one committee. It begins at the first block of a
Midnight epoch and normally lasts that epoch, so this MIP uses session and
epoch for the same interval. The BEEFY payload is the MMR root alone (the
upstream `mh` payload). Validators sign
`(payload, block_number, validator_set_id)`, and a justification is valid
with signatures from at least `seat_count − (seat_count−1)/3` seats. The
set id increments each session. A mandatory signed round on the
first block of each session, signed by the incoming set, guarantees one
justification per session as long as two-thirds of seats vote.

### Cardano-side verification

The light client contract stores the 44-byte commitment of the current
committee. A submitter presents a signed commitment along with, for each
signer, the key, its seat count and one ECDSA signature, plus one Merkle
multiproof of all signer leaves against `keyset_commitment`. Signers must
appear in strictly increasing key order, which prevents double-counting,
and the root is accepted when the valid signers' seat counts sum to at
least `seat_count − (seat_count−1)/3`.

**Transaction size.** A signer entry is 101 bytes (33-byte key, 4-byte
seat count, 64-byte signature), and membership is proven with the single
Merkle multiproof over the whole signer set. At 100 distinct keys and a
67-key quorum the submission is roughly 10 KB against Cardano's
16,384-byte limit, so on the order of 100 unique signers fit in one
transaction.

**Committee size.** `signer_cap` is the largest committee of distinct
single-seat members whose quorum update fits one Cardano transaction,
signer entries and multiproof included, in size and in execution budget.
It follows from the certificate layout above and is fixed by measurement;
the 100-key figure is the current estimate. The committee-size parameter,
the seat total that governance sets and Ariadne selects to, is held at or
under `signer_cap`, and the runtime selects at most `signer_cap` seats
whatever the parameter says. Every seat can go to a different pool, so
this is the one setting under which no session can produce a handover
that a single transaction cannot carry. `MaxAuthorities` stays the
runtime's type bound and is not the operative limit.

**Handover.** Every leaf of session N carries the commitment of set N+1.
The contract must extract set N+1 from a leaf proven against a root signed
by set N *before* it accepts any signature from set N+1; a session's own
mandatory justification can never bootstrap its own set. Handovers are
consumed in order.

**Bootstrap.** The first committee is the base case of the induction and
is assumed honest. The light client is deployed once, its NFT minted by a
one-shot policy, with an initial datum that names
`beefy_activation_block` and, as `current_committee`, the commitment of
the validator set that signs at that block. Governance deploys it under
the authority that controls the contract. The commitment is a function of
public Midnight state at the activation block, so anyone can recompute it
and compare it with the deployed datum before relying on the light
client. Every later committee reaches Cardano by handover alone.

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
- `max_fee` is the fee cap for an update that the funding pool pays for,
  as two lovelace amounts, `base` and `per_signer` (see Funding pool).

Seat counts need no state of their own: `seat_count` inside each
commitment is the quorum denominator, so the datum carries no separate
stake or weight totals. Accepting a signed commitment replaces
`latest_mmr_root` and `latest_height` in one step and, when the proven
leaf names a new successor set, advances the committee pair. An update
carries `max_fee` forward unchanged, so no submitter can raise it.
Governance sets it, under the authority that already controls the
contract.

Consumers never spend the light client. A downstream contract, such as
the rewards contract, reads the datum as a reference input and verifies
a Keccak-256 MMR inclusion proof against `latest_mmr_root` itself. Since
every root commits to all prior blocks, the latest root proves any past
Midnight block. Consumers do not contend with each other. They contend
with updates and with nothing else: an update spends the UTxO, so a
consumer transaction built against the datum it replaced fails Cardano's
first validation phase, costs nothing, and is rebuilt against the new
datum.

### Funding pool

Each flow that the data pump serves has one funding pool of its own. The
bridge's pool pays the ADA fee of a light-client update, so the fee does
not come from the party that submits the update. The pool is ADA at a
script address bound to the light client. Anyone may add to it, and block
producers are the expected funders (see Rewards coupling). ADA leaves the
pool only in a transaction that carries a valid light-client update, and
only as that transaction's fee: the pool's ADA falls by no more than the
fee, and the fee is no more than the cap from the light-client datum. A
submitter pays no fee of its own, and the pool can pay for nothing else.
A later flow reuses this construction with a pool of its own; the Block
Production Rewards MIP has one, which pays for its release and its epoch
load.

The cap follows the one input that moves the cost of an update, the
number of signers it presents:

```
cap = max_fee.base + max_fee.per_signer × signers
```

Both amounts are calculated in advance from measured cost (see
Rationale). The cap is a ceiling, not a price: an honest submitter sets
the fee the transaction needs, which sits below the cap. Submission stays
permissionless, and the pool pays for whichever valid update lands.

The pool pays only for an update that advances the committee pair. That
happens once per session, so the pool's spend is one capped fee per
session whatever else is submitted. The light client still accepts any
strictly newer quorum-signed root; a party that wants a fresher root
between handovers pays that fee itself. An empty pool stalls updates and
loses nothing (see Security Considerations).

### Data pump

Every consensus node runs a data pump: one component of the Midnight node
that carries periodic work to Cardano automatically, with no fee-paying
wallet and no operator action. The component is modular. Each periodic
job is a module, every module's transaction is paid from the funding pool
of its flow, and the contract a module serves records the `max_fee` for
its job, in its state or in a parameter UTXO it reads. This MIP defines
the component and its first module, the light-client update. The Block
Production Rewards MIP adds the reserve release and epoch load modules,
and any later job is one more module.

All modules share four parts:

- **Configuration.** The operator configures a Cardano submit endpoint.
  There is no fee-paying key to configure.
- **Cardano state.** The pump reads the UTXOs a transaction spends, and the
  `max_fee` in force, from the Cardano data source the node already uses.
  These reads are not consensus inputs, so they follow Cardano's tip
  rather than the stability window. The data source allows it: the
  stability offset is applied by the inherent data providers that call
  it, and its UTXO queries take the bounding block from the caller. A
  read that a rollback overtakes yields a failed transaction and a later
  attempt, nothing worse.
- **Transaction builder.** The pump includes a Cardano transaction
  builder. It sets the fee the transaction needs, never more than
  the cap, and draws it from the funding pool.
- **Ordered submission, one winner.** A module works through its items in
  order and reads from the contract state which items are done. Every
  node attempts the same item, one submission lands, and the rest fail on
  the spent UTXO. A node that sees the item done moves to the next one and
  does not retry. A node that finds the pool unable to pay tries again
  later.

For the light-client module an item is one session's mandatory
justification, with the leaf proof for the handover it enables. The marker
is the light-client datum: `latest_height` and the committee pair show
which sessions have landed. These are fields the light client already
needs, and no state is added for the pump. When Cardano is several
sessions behind, the module submits the oldest missing session first,
because handovers are consumed in order, and continues until the light
client is current. One funded update per session sets the freshness of
the root: a Midnight block becomes provable on Cardano when the next
session's update lands, at most one session after the block.

### Rewards coupling

Block-production rewards are paid in NIGHT on Cardano and unlock strictly
in session order: session N's reward data becomes claimable only after
session N's commitment is verified. No consensus rule forces voting;
participation is driven by this coupling, since a committee that fails to
produce justifications delays all reward payments, including its own.
Individual non-participation is visible to anyone from the
justifications; a response to it is future work (see Security
Considerations).

The same coupling keeps the bridge supplied on Cardano. While the light
client does not advance, no reward digest can be loaded and no block
producer is paid. Block producers therefore have a direct reason to run
the data pump and to keep the funding pool topped up. Nothing requires
that they be the funders; anyone may add ADA.

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

### Epoch length

The Midnight epoch is raised from 30 minutes to six hours: 3,600 slots
at the six-second block interval, with the slot count scaled if the
block interval ever changes so that the epoch stays six hours. One
mandatory BEEFY round and one light-client update per session is the
bridge's steady-state cost, and per-epoch settlement of rewards on
Cardano (digest, release, fold) is practical only at that cadence. The
change is a runtime upgrade with a storage migration of session- and
epoch-keyed state, landed together with BEEFY activation.

## Rationale

Paths considered and rejected, the one deferred, the committee size and
the epoch length, and the reasoning behind how updates are funded and
submitted.

### Explicit stake weights in the commitment (rejected)

A stake-weighted quorum conflicts with the standard Polkadot BEEFY model:
upstream voting and justification verification count seats, so explicit
stakes would require forking `sc-consensus-beefy`, and they would
double-count stake that seat repetition already expresses. It also buys
nothing, since GRANDPA finality is already a two-thirds count over the
same seats, so a bridge quorum cannot exceed the chain's own security.
Seats *are* the stake weight under Ariadne; the bridge simply inherits
the chain's trust model.

### Extra payload entries (rejected)

The existing prototype carries current and next committee-and-stake
entries beside the MMR root. Everything is provable from the MMR against
the signed root, and the extra entries only bloat every vote and
justification. The payload is the MMR root alone, the vanilla
Snowbridge-style light-client model.

### Handover as a mandatory inherent extrinsic (rejected)

Redundant. The header digest already makes the same data
consensus-enforced every block with zero new validity rules. The inherent
would have added a hard fork and a novel stall mode, and it still could
not carry the signatures, which are the only part that cannot be chain
content.

### Succinct (ZK) quorum verification (next step)

A quorum of roughly 100 distinct signers fits one transaction today, so
direct signature checking ships first. Its price is the cap on committee
size below. A ZK proof that a threshold of the committee signed, verified
inside one transaction, replaces the signature list without changing the
trust model and lifts the cap, which makes it the urgent item after this
MIP.

### Committee size held under the signer cap

A certificate that lists signatures grows with the signers, and a Cardano
transaction does not. Handovers are verified one against the next, so a
single session whose quorum does not fit stops every later handover, and
with it every fact that flows to Cardano, until a larger verifier exists
there. Registration is open to any pool, and in the worst case every seat
goes to a different one, so no setting above `signer_cap` makes that
impossible. The rule is enforced twice: governance holds the parameter
under the cap, and the runtime truncates selection at the cap, so a
parameter set too high cannot produce an unverifiable committee. The cost
is decentralization: a committee under the cap seats at most that many
distinct pools per session, however many have registered. How much
diversity that forgoes depends on the live pool distribution. It is a
property of the certificate form and not of the design, and the succinct
proof removes it.

### Six-hour epoch

Two costs pull against each other. Every session costs one funded
update, so the funding pool drains at no more than `cap × sessions per
day`: 48 caps a day at the current 30-minute epoch, 4 at six hours.
Against that, a shorter epoch buys little for the bridge. Committee
selection reads its Cardano inputs, the seat division, the registrations,
each pool's stake and the epoch nonce, once per Cardano epoch of five
days. Sessions shorter than that reselect from the same candidates at
the same weights with a new random draw, so at 30 minutes 240 handovers
per Cardano epoch carry the same inputs, and at six hours 20 do. The
floor comes from the consumers: a Midnight fact becomes provable on
Cardano at the next session's update, and rewards settle once per epoch,
so a five-day epoch would hold every payout for days. Six hours cuts the
drain by twelve and keeps four settlements a day. A pool of `P` lovelace
lasts at least `P ÷ (4 × cap)` days, which a funder can compute.

### BEEFY vs GRANDPA

BEEFY exists for exactly this use: secp256k1 ECDSA signatures that are
cheap to verify on foreign chains, an MMR payload built for inclusion
proofs, and a committee commitment built for light-client handover.
GRANDPA justifications (ed25519, shaped as vote sets) are strictly more
expensive to verify on Cardano with no difference in trust model: the
residual risk, a dishonest two-thirds signing a chain that honest nodes
reject, is identical under either.

### An ADA cost is never compensated in NIGHT

Work on Cardano costs ADA. To repay that cost in NIGHT would make every
submitter price NIGHT against ADA, and the protocol would carry an implied
exchange rate. The Block Production Rewards MIP applies the same principle
to its batcher: ADA covers the ADA cost, and the NIGHT fee is a separate
reward for effort, unrelated to any ADA spent. A light-client update is
almost entirely ADA cost, so ADA pays for it and no NIGHT payment is
attached. The incentive to keep it running is the rewards coupling.

### A funding pool, not an operator wallet

Block producers could pay the update fee themselves, and their incentive
might be enough. But every consensus node is to push updates
automatically, and with a connected wallet that means software that
spends an operator's ADA unattended on every node. A funding pool gives
the automation without the wallet: the node holds no key that controls
funds, and the ADA set aside for the bridge can be spent on the bridge
alone. Each flow funds itself: a funder's ADA pays for the flow it was
given to, and a later flow cannot spend what was set aside for this one.
The rewards batcher was considered as the submitter, since it needs each
session's commitment landed. The nodes already hold every justification
and have the reason to deliver it, so the submitter is the node.

### A fee cap calculated in advance, per signer

On Cardano the transaction creator sets the fee, and fees go to the reward
pot that stake pool operators and their delegators share. A submitter
that is an SPO gets part of any fee back, so a pool with no cap could be
drained by padded fees at a profit. A cap fixed in advance bounds what
each transaction can take, and it can be fixed in advance because script
cost is deterministic: an update runs the same checks over inputs of the
same magnitude each time, so its fee can be measured before deployment.
The one input that moves the cost is the number of signers, each of which
adds 101 bytes and one signature check. A single flat amount would have
to cover the largest quorum and would leave a gap under every smaller
one, so the cap is a base amount plus an amount per signer, and it stays
tight at any committee size. The values are left to measurement.

### The pool pays for one update per session

The light client accepts any strictly newer quorum-signed root, and BEEFY
finalizes far more often than once per session. A pool that paid for
every valid update could be emptied at no cost to the party that submits
them, with honest fees. The update that advances the committee pair
happens once per session and is the one the bridge cannot live without,
so that is the update the pool pays for. The price is freshness: the
rewards digest of epoch E lands at or after the first block of session
E + 1, so the root that proves it is the one from session E + 2, and
payouts settle one epoch later than a continuously updated root would
allow. A delayed payout loses nothing.

### One pump in the node, not a service per job

Work that block producers should do frequently and as a matter of course
belongs in one expandable component of the node. Every module has the
same structure: a funding pool for its flow, a `max_fee` recorded in the same
kind of place, a marker that shows which items are done, and the same
handling of an empty pool or an item that another node already delivered.
A new job reuses the configuration, the transaction builder and the
submission rule, and adds only what is specific to its contract.

## Path to Active

### Acceptance Criteria

- Multiple committee rotations, including a membership change, verified
  end to end on a public testnet by the light client.
- Adversarial submissions rejected on chain: insufficient seats, a stale
  set, a non-successor set, a replayed signer, a fee above the cap
  charged to the funding pool, a funded update that does not advance the
  committee pair.
- The light client advanced across those rotations by the data pumps of
  several competing nodes, with every fee paid from the funding pool and
  no fee-paying wallet.
- Every registered candidate carries a `beef` key, and the fallback to
  the cross-chain key is removed before BEEFY voting activates on a
  network whose light client carries value.
- `signer_cap` measured on a committee of distinct single-seat members,
  and selection shown truncated at it when the committee-size parameter
  is set above it.
- The deployed light-client datum checked against the commitment
  recomputed from Midnight state at `beefy_activation_block`.
- The epoch length change deployed with its storage migration.
- Audit of the light-client contracts complete.

### Implementation Plan

The steps in the Implementation section land in order, registration
first, then node, then contracts and data pump, each on a Midnight devnet
against a Cardano test network, then on the public testnet for a soak
across many sessions, then audit, then mainnet.

## Backwards Compatibility Assessment

No re-genesis and no Cardano hard fork; the Midnight side is a runtime
upgrade with a storage migration, and the Cardano side is a new
light-client deployment. Where the node is today, read from `main` and
the `lglo/beefy-on-main` branch on 2026-09-04, and what changes:

- The `Beefy`, `Mmr` and `BeefyMmrLeaf` pallets are in the runtime on
  `main`. The MMR hashes with Keccak-256, the leaf version is `(0, 0)`,
  and the BEEFY key type is `beef` with ECDSA keys. This matches the
  specification.
- The BEEFY payload carries five entries: the MMR root plus current and
  next committee stakes and authority sets. The specification reduces the
  payload to the MMR root alone. The extra entries and the runtime API
  that serves them are to be removed.
- The committee commitment is built with the leaf layout in this MIP
  (`pubkey ‖ seat_count`), but the count is a little-endian `u64`, every
  validator's seat count is hard-coded to 1, and keys are not
  deduplicated. The specification uses a `u32` count per distinct key.
- The `beefy` session key is being added on `lglo/beefy-on-main`. That
  branch falls back to the candidate's cross-chain ECDSA key when no
  `beef` key is registered, and its storage migration seeds each
  validator's BEEFY key from the cross-chain key. This MIP requires a
  dedicated `beef` key with no fallback. The fallback is acceptable as a
  migration convenience on test networks and must be removed before BEEFY
  voting activates on a network whose light client carries value.
- Equivocation reporting is a no-op in the runtime's `BeefyApi`, and there
  is no ban list.
- A relay (`midnight-beefy-relay`) subscribes to justifications, builds
  the signer proofs with Keccak-256, and encodes them as Plutus data for
  the current prototype light client. It is a standalone binary outside
  the node, and it logs the encoded proof without submitting it. This MIP
  moves submission into the node's data pump,
  paid from the funding pool. The relay's proof building and encoding
  carry over to the pump's light-client module, reworked for the
  MMR-root-only payload and the light-client state in this MIP.
- The running node has no path that submits to Cardano. Transaction
  building and Ogmios submission exist only in the command-line tooling
  the node binary carries from Partner Chains. The data pump adds the
  submit path to the node service, and gives the pump a handle on the
  db-sync data source bounded at the tip.

**Session keys and registration.** Candidates without a registered `beef`
key are excluded from candidacy once BEEFY voting activates, so every
SPO re-registers before activation. Permissioned members supply keys
through governance configuration. The test-network fallback to the
cross-chain key is a migration convenience only.

**Epoch length.** Raised to six hours in the same runtime upgrade,
together with the storage migration that requires. Anything keyed to
the 30-minute epoch, including the rewards settlement cadence, follows.

**Node operators.** The data pump ships in the node. An operator
configures a Cardano submit endpoint, and no fee-paying wallet is
attached. Collateral is the open point (see Open Design Questions).

**Downstream consumers.** Contracts that read the light client as a
reference input see a single-UTxO datum with the fields in Light-client
state and consumption; the prototype light client's state layout is not
carried forward.

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
  chain, is provable and attributable through BEEFY's `DoubleVotingProof`
  and `ForkVotingProof`. There is no slashing, since stake is Cardano
  delegation. Permissioned members are removed by governance. A
  reporting extrinsic and a ban list consulted in candidate filtering
  are future work for a separate MIP; this MIP makes equivocation
  provable and attributable, not penalized. A self-consistent full fork is chain takeover and out of
  scope here.
- **Liveness.** More than a third of seats withholding votes freezes the
  bridge and rewards until participation resumes. A stall, never a theft,
  and it self-heals: BEEFY never skips a mandatory round, so the round
  stays open and late votes under the same `validator_set_id` complete it
  whenever enough of that session's set returns. The unrecoverable case
  is a session set that permanently lost more than a third of its seats,
  or an off-protocol authority reset (`note_stalled`). Recovery there is
  a governed re-registration of the committee commitment on Cardano,
  under the Council and Technical Authority's existing contract-update
  authority.
- **Funding pool exhaustion.** An empty pool stalls the light client, and
  rewards with it, until someone adds ADA. No information is lost. Nodes
  keep each session's justification beside the finalized block, the
  mandatory round guarantees one per session, and the latest MMR root
  proves every earlier block. When the pool is refilled, the data pumps
  resume from the oldest missing session, in order.
- **Fee extraction from the funding pool.** A submitter that is an SPO
  gets part of any Cardano fee back through Cardano's own rewards, so a
  padded fee paid by the pool is profit, not griefing. The per-signer
  cap bounds the fee of every update the pool pays for, so the most a
  padded update takes is the gap between the cap and the real cost. The
  pool pays only for the update that advances the committee pair, so the
  number of funded updates is one per session and extra submissions
  cannot drain it.
- **An unverifiable handover.** A session whose quorum does not fit one
  transaction would stop the induction for good. The committee size is
  held at or under `signer_cap` by governance and by the runtime's
  selection, so no such session can occur (see Committee size).
- **The base case.** Everything the light client accepts rests on the
  first committee and on the datum it was deployed with. That commitment
  is recomputable from public Midnight state at
  `beefy_activation_block`, so a wrong deployment is detectable before
  any value rests on it.
- **Updates that disturb readers.** Each update invalidates the consumer
  transactions in flight against the datum it replaced. They fail at the
  first validation phase at no cost and are rebuilt. A party that submits
  extra updates to disturb readers pays a full fee for each, since the
  pool funds one per session, and can submit no faster than BEEFY
  produces justifications.

## Implementation

1. Registration: extend the Cardano-side candidate registration and its
   tooling with the `beef` key, require it in candidate filtering, and
   re-register existing SPOs before BEEFY voting activates.
2. Node: enable BEEFY voting (session keys, candidate keys, storage
   migration), the deduplicated committee commitment, the
   MMR-root-only payload, and selection truncated at `signer_cap`.
3. The Cardano light-client contracts: seat-sum quorum, handover state
   machine, Keccak MMR proofs, the single-UTxO reference-input state
   with `max_fee`, and the funding pool.
4. The data pump in the node: the modular component, its configuration,
   the Cardano transaction builder, and the light-client module, which
   reuses the relay's proof building and encoding.
5. Testnet rotation soak across many sessions, and an audit.

Registration lives in the Cardano-side candidate registration and its
tooling; steps 2 and 4 in `midnight-node`, with step 4 drawing on
`midnight-beefy-relay`; step 3 in `midnight-reserve-contracts` (the
committee bridge validators). The rewards batcher of the Block Production
Rewards MIP is the first consumer of the light client and does not
change this design. That MIP adds the pump's next two modules, the
reserve release and the epoch load.

## Testing

**Runtime unit tests.** The deduplicated commitment: grouping by key,
ascending order, `pubkey ‖ seat_count` leaves, total seat count as the
denominator, and the same root from a committee with and without
repeated members. Quorum arithmetic at the `seat_count − (seat_count−1)/3`
boundary. The `beef` key requirement in candidate filtering, and the
storage migration on a populated session set. Selection truncated at
`signer_cap` when the committee-size parameter is set above it.

**Contract tests.** Signature verification over the commitment bytes;
signers accepted only in strictly increasing key order; the multiproof
against `keyset_commitment`; quorum by seat sum, one seat short rejected;
handover only from a leaf proven against a root signed by the current set,
a session's own justification never bootstrapping its set; `latest_height`
strictly increasing; proofs behind `beefy_activation_block` rejected; a
10 KB submission at 100 distinct keys within the transaction budget; the
funding pool debited by no more than the fee, a fee above the
per-signer cap rejected, the pool unspendable without an update that
advances the committee pair, and `max_fee` unchanged by an update; a
committee of `signer_cap` distinct single-seat members verified in one
transaction within size and execution budget, which is the measurement
that fixes `signer_cap`; the deployed datum matching the commitment
recomputed from Midnight state at `beefy_activation_block`; a consumer
transaction built against a replaced datum failing at the first
validation phase.

**Data pump.** Several nodes racing on one session, one submission landing
and the others not retrying it; a light client several sessions behind
caught up oldest first; an empty pool stalling the module and a top-up
resuming it; the fee set to the transaction's need, below the cap.

**End to end.** On a Midnight devnet against a Cardano test network:
justifications submitted each session by the data pump, and the
light client advanced through several rotations including a membership
change and a Cardano-epoch boundary; a downstream contract proving a
past block by MMR inclusion against the latest root; the adversarial
submissions of the Acceptance Criteria rejected; a liveness stall with a
third of seats withheld and recovery when they return.

## Open Design Questions

- **Collateral without a wallet.** Cardano requires every transaction
  that runs a Plutus script to name collateral held at a key address and
  to carry that key's signature. The ledger takes collateral only when a
  script fails, which a node that validated its own transaction does not
  incur, but the funding pool cannot supply it. How the data pump provides
  collateral while holding no fee-paying wallet is undecided.
- **Contract handover encoding.** The ordering above is fixed; concrete
  redeemer and datum encoding belongs to the contract implementation.
- **On-chain misbehavior response.** Whether the light client should
  react to committee misbehavior on Cardano itself, for example a
  redeemer that freezes the contract when shown two conflicting
  quorum-signed commitments for the same block, possibly behind a
  challenge window. Undecided; needs design and cost analysis.

## References

- Block Production Rewards MIP (companion; consumes this bridge):
  [midnight-improvement-proposals PR #321](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/321).
- BEEFY relay and payload worked example:
  [`midnight-node/relay/README.md`](https://github.com/midnightntwrk/midnight-node/blob/main/relay/README.md)
- Polkadot BEEFY protocol documentation
- Snowbridge (the Polkadot-to-Ethereum bridge using the same BEEFY
  light-client model)

## Appendix A: The block rate, assumed fixed

Midnight produces a block roughly every six seconds, and the six-hour
epoch is 3,600 of them. The tokenomics whitepaper anticipates the block
interval changing as the protocol evolves, and neither the epoch nor the
reward machinery of the Block Production Rewards MIP would survive that
change untouched. This appendix records what breaks and what compensates;
nothing in it is normative.

The two sides of the design count in different units. The pallet awards
per block, at a rate configured per block, while the reserve contract
releases per wall-clock interval, timed against a transaction validity
range. At six seconds those coincide: 3,600 blocks in six hours draw
345,196 NIGHT from a full reserve, and the pool ceiling releases the
same 345,196. Change the block rate and nothing in either component
notices. At three-second blocks the pallet awards about 690,400 NIGHT per
six hours against an unchanged release of 345,196, and at two seconds
about 1,035,600. Rewards outpace releases by the same factor the chain
speeds up, the pool drains to nothing within the first interval or two,
and there is no NIGHT on Cardano to pay the rewards that have been
earned. A slower chain inverts it: twelve-second blocks award about
172,600 per six hours against the same release, so the pool accumulates
an unearned float and emission falls to half the published rate. Nothing
in the system corrects that. The pool running dry is visible to a batcher
as a failed fold, but no component adjusts, and the shortfall persists
until a rate is changed by hand on one chain or the other.

One edit compensates: the per-block rate `R` in the pallet's
configuration, the integer pair `rn/rd`. What has to stay fixed across a
blocktime change is `Ra`, the annual rate, and since `R = Ra ÷ γ`,
doubling the blocks in a year means halving `R`. At 3% the pair goes from
`7/438,000,000` at six seconds to `7/876,000,000` at three and
`7/2,628,000,000` at one. Which chain pays for the fix depends on which of
two intents the change carries: hold annual emission where the published
rate put it, or let emission scale with the block rate and accept a new
annual figure.

| Direction | Intent | Accompanying change |
|---|---|---|
| Faster blocks | Hold annual emission | Halve `R` in the pallet configuration |
| Faster blocks | Let emission scale | Recompute the contract's factor, recompile, upgrade the reserve and pool |
| Slower blocks | Hold annual emission | Double `R` in the pallet, recompute the contract's factor, upgrade the reserve and pool |
| Slower blocks | Let emission scale | None; emission and the published rate diverge |

Speeding up while holding emission is the cheap case: `R` halves as `γ`
doubles, per-interval emission returns to where it was, and the
contract's factor still covers it, since finer slicing compounds a little
more and the draw lands marginally below the configured value. A pallet
configuration change is the whole of it. Every other row reaches the
Cardano contracts: the interval factor is compile-time configuration
rather than a datum field, so covering a larger draw means recomputing
the factor, recompiling, and taking the reserve and pool through a
two-stage governance upgrade to new script addresses, in step with the
runtime change on Midnight. Balances and addresses are unaffected, as are
the batcher and the accounts. Where both sides change, they have to land
in order. Widening before narrowing is the safe sequence: a ceiling that
is too high over-releases into the pool, which the next release absorbs,
while a ceiling that is too low leaves an interval underfunded and its
recipients unpaid.

Epoch duration needs adjusting in every one of those cases, because it
follows from the block rate rather than from the intent. One release funds
one epoch's payouts, and that holds only because 3,600 six-second slots
and a six-hour interval are the same length. An epoch is a count of slots
while the contract's interval is wall-clock milliseconds, so three-second
blocks halve the epoch to three hours and leave two digests to be folded
against each release, while twelve-second blocks stretch it to twelve
hours and leave releases arriving with no digest to fold. The remedy is
arithmetic: scale the epoch's slot count by the same factor as the block
rate, to 7,200 slots at three seconds or 1,800 at twelve, so an epoch
stays six hours and one release still funds one fold.

The rest of the reward machinery is insulated. The reserve's half-life is
`ln 2 ÷ 2.8π` and depends on `Ra` alone, so it survives a blocktime change
untouched. DUST generation and decay are denominated in seconds rather
than blocks, so nothing there moves either.

## Acknowledgements

The reviewers of the successive drafts, and the node engineers whose
BEEFY work on `midnight-node` this MIP builds on.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed
under the Apache License, Version 2.0. Submission requires agreement to the
Midnight Foundation Contributor License Agreement, which includes the
assignment of copyright for your contributions to the Foundation.
