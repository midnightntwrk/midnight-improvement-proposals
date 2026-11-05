---
MIP: "0010"
Title: Consensus Migration from AURA to BABE
Authors:
  - Dominik Zajkowski (dzajkowski)
Status: Proposed
Category: Core
Created: 2026-06-08
Requires: none
Replaces: none
MPS: none
License: Apache-2.0
---

<!--
 Copyright 2026 Midnight Foundation

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

Midnight's current AURA consensus uses a deterministic, rotating leader schedule across a small, permissioned committee. The next block producer is publicly predictable, so an attacker can direct a surgical DDoS at a single known node and stall the chain for that slot - at constant, low cost across consecutive slots.

This MIP migrates block production from AURA to **BABE**. In BABE each slot has a deterministic secondary author (AURA-like) and may also have a VRF-elected **primary** author whose identity is hidden until the block is published. The active slot coefficient *c* sets the fraction of VRF-hidden slots: higher *c* means more hidden slots but more slot-ownership collisions (short-lived forks). Suppressing a VRF-assigned slot requires DDoSing the **entire** committee, so attack cost scales with committee size rather than staying constant. BABE is production-grade and runs Polkadot's relay chain. Finality remains GRANDPA, unchanged.

The migration is performed **in place on a running chain** - no re-genesis - with AURA and BABE coexisting during a phased switch. This MIP does **not** address block-propagation limits, on-chain proof size, or throughput; those are scoped to the performance MPS (MPS PR #82).

This MIP is one step in a larger roadmap toward decentralization. It is necessary but not sufficient for full permissionless operation: BABE's hidden-leader property is a prerequisite for opening block production to unknown validators, but does not on its own establish that doing so is safe. The migration to BABE is performed and proven entirely under federated operation, ahead of any permissionless opening.

## Motivation

**Targeted DDoS vulnerability (MPS PR #83).** AURA's leader schedule is deterministic and public. With some effort, an observer can discover the physical location of the next block producer and direct a surgical DDoS at that single node, stalling the chain for that slot. The cost of sustaining this across consecutive slots is constant - the schedule is known in advance.

**Small committee vulnerability (MPS PR #83).** Even with leader anonymity, full committee membership is public. If the committee is small, a coordinated DDoS against all members is economically viable. Under AURA, committee size is irrelevant to *targeted* attacks - an attacker only needs to hit one known node. BABE makes committee size matter: suppressing a VRF-assigned slot requires attacking the whole committee.

## Specification

BABE becomes the block-production engine; GRANDPA finality and the Ariadne committee-selection algorithm are unchanged - the committee pallet only gains a record of the committee queued in the session machinery (see *Authority Management*). The migration is staged on a running chain (see *Migration Architecture* below). This section first specifies BABE's steady-state behavior after the switch, then the migration mechanism.

Identifiers throughout this document (pallet, call, and storage names) follow the node v3 implementation.

### BABE Configuration

- **Slot duration:** 6,000 ms (unchanged from AURA).
- **Epoch length:** bound to the sidechain epoch length. `pallet_babe::EpochDuration` reads the sidechain pallet's `slots_per_epoch`, so BABE epochs and sidechain epochs coincide by construction. It sets how often VRF randomness refreshes and authority sets rotate.
- **Active slot coefficient *c* = 1/4**, applied from the **first** BABE epoch (a single jump; there is no intermediate secondary-only / `c = 0` epoch). `c` is held in `pallet_babe`'s `EpochConfig`, written at the flip from the runtime constant `BABE_GENESIS_EPOCH_CONFIG` when absent (an upgraded chain never ran BABE genesis), and adjustable thereafter via governance (`NextEpochConfig`) without a code change. `allowed_slots = PrimaryAndSecondaryVRFSlots`: after the flip, secondary blocks carry a VRF output as well. The `SecondaryPlain` variant appears only in the pre-flip dual-digest window (see *Migration Architecture*).

### Slot Assignment

Each slot has two potential authors:

1. **Primary (VRF):** each validator evaluates a VRF against the epoch randomness and its secret key; if the output is below a threshold derived from *c*, it is the primary author for that slot. The winner is unknown to others until the block is published with its VRF proof. Multiple validators may win one slot (a collision), producing short-lived forks resolved by fork choice.
2. **Secondary (deterministic):** a fallback author assigned round-robin - mechanically identical to AURA. A secondary block loses fork-choice priority to any primary block in the same slot.

### Fork Choice

BABE uses longest-chain with primary-block priority: between equal-length chains, the one with more primary blocks wins. GRANDPA finalizes independently - once finalized, a block cannot be reverted.

### Collision Costs

With a per-validator primary threshold `p = 1 − (1−c)^(1/n)` (set so `P(slot has ≥1 primary) = c`), the collision rate `P(≥2 primary winners) = 1 − (1−c) + (1−c)·ln(1−c)` depends on `c` alone, not committee size `n`. At *c* = 1/4 (0.25) that is ~3.4%, producing ~34 collisions per 1,000 blocks. Each collision wastes the losing block's bandwidth and the importing nodes' verification work. At 2 MB blocks of shielded transactions (~136 transactions, ~544 proofs per block):

| | Per collision | Per 1,000 blocks (34 collisions) |
|---|---|---|
| Bandwidth | 2 MB | 68 MB |
| Verification (sequential) | 1.7 s | 57.8 s |
| Verification (8 cores, parallel) | 211 ms | 7.2 s |

Sequential verification makes each collision expensive (1.7 s of CPU discarded); parallel verification makes it manageable. The proof count and verification times here come from the throughput experiments in MPS PR #82 (to become its own MIP); the millisecond figures are indicative - a token value measured on the specific machine the tests ran on, not an absolute. Mitigation of verification cost is scoped to that work.

### Migration Architecture

The switch is performed in place, without re-genesis. AURA and BABE are both compiled into the runtime and node; blocks are routed by the engine ID in their pre-runtime digest. A new pallet, **`pallet-consensus-engine`**, holds the engine state (`EngineState`: `Aura`, `ScheduledFlip`, `Babe`) and drives the transition.

**Coexistence.** `pallet_aura` is **not removed**; it keeps authoring until the flip. `pallet_babe` is added alongside it and takes part in session rotation like any other consensus pallet (see *Authority Management*). `pallet-consensus-engine` routes timestamp and slot bookkeeping (`OnTimestampSet`, `current_slot()`) and block-author lookup (`FindAuthor`) to whichever engine is active, so the sidechain epoch calculation, committee rotation and authorship attribution stay correct before and after the switch.

**Rollout.** The migration is one binary rollout, one runtime release and one governance call. No flip slot is recorded in advance:

1. **Binary rollout.** The migration-aware node (v3) is deployed to every node before any runtime change. It runs both engines' import pipelines and attaches a BABE `SecondaryPlain` pre-runtime digest to every AURA block it authors, carrying the AURA slot and author index, whether or not the runtime knows about it yet. A runtime without the pallet ignores the extra digest.
2. **Runtime upgrade (activation).** A single runtime upgrade adds `pallet_babe`, `pallet-consensus-engine` and the `babe` session key, and runs one migration that translates the stored committees and session keys to the new shape (see *Key Model*), seeds `QueuedCommittee`, and activates the engine pallet by pre-seeding `pallet_babe::GenesisSlot` with a sentinel (`u64::MAX`) so that `pallet_babe` does not self-initialize its genesis epoch from the first BABE digest it sees. From the first block executed by the new runtime (the **activation block**), every block MUST carry the AURA pre-digest followed by a matching BABE `SecondaryPlain` digest; a block without it is rejected, so an author still on a pre-migration binary is excluded from that point. `EngineState` is `Aura`.
3. **Schedule (governance).** The governance origin dispatches `consensusEngine.scheduleFlip()`, which moves `EngineState` from `Aura` to `ScheduledFlip`; from any other state the call fails with `InvalidEngineState`. Nothing else is recorded. Two preconditions are operational and are not checked by the runtime: the activation block MUST be finalized by GRANDPA, so that the upgrade is irreversible before the point of no return, and `pallet_babe`'s authority set MUST have been populated by at least one committee rotation since the upgrade.
4. **Flip (automatic).** While in `ScheduledFlip`, the pallet flips in the first produced block whose slot is the last slot of an epoch. If no block lands in that slot, the flip fires at a later epoch's last slot. The flip is postponed, epoch by epoch, while `pallet_babe`'s authority set is empty. At the flip the pallet sets `GenesisSlot` to the first slot of the next epoch, writes `EpochConfig` (`c = 1/4`) if absent, and sets `EngineState = Babe`. The flip block is the last AURA block; its child is the first BABE block. From that first BABE epoch `c = 1/4` (VRF live).

**Digest guards.** The engine pallet checks every block's pre-runtime digests against the engine state. In `Aura` and `ScheduledFlip`, the AURA pre-digest MUST be followed by a BABE `SecondaryPlain` digest with the same slot and author index; `Primary` and `SecondaryVRF` variants are rejected there, since their VRF material is never client-verified while blocks import through the AURA pipeline. In `Babe`, any AURA pre-digest is rejected, so a stray one cannot hijack slot or author extraction. The pallet is ordered after `pallet_babe` and before `pallet_session` and `pallet_scheduler`, so the guards evaluate the same parent state the author and the client-side verifier used.

### Authority Management (committee staging on `pallet_session`)

BABE must announce the **next** epoch's authority set *at* the epoch boundary (via the `NextEpochData` digest) so clients can validate the next epoch in advance. Mid-epoch, the Ariadne inherent selects the committee for the next unset sidechain epoch and stores it as `NextCommittee`. The committee pallet (`pallet-session-validator-management`, storage version 2) tracks three sets: `NextCommittee` (selected by the inherent), `QueuedCommittee` (the set handed to `pallet_session` as its queued validators at the last rotation) and `CurrentCommittee` (the active set). At each rotation they shift (`Next → Queued → Current`). Because the inherent selects one epoch ahead and the rotation at the start of epoch E queues the committee for E+1, `pallet_session` always holds the next set at the boundary and `pallet_babe` announces it there.

This is always computable: a committee's inputs come from Cardano data offset by 2 mainchain epochs (finalized), and a sidechain epoch (minutes to hours) is far shorter than a Cardano epoch (5 days), so a bounded look-ahead sits well within the stable margin, including the Cardano-epoch straddle, where the next committee is computed from the newer, already-final snapshot.

**Session integration.** The chain uses stock `pallet_session`. The committee pallet implements `SessionManager` and `ShouldEndSession` and registers committee keys itself at genesis and on each rotation (the `set_keys` extrinsic is disabled). `pallet_babe` is therefore a regular session handler: it receives the queued validators' BABE keys at each rotation, which keeps `Authorities` and `NextAuthorities` in step with the committee, and it runs with `ExternalTrigger`, so its epoch changes are driven by session rotation rather than by its own slot counter.

**Exactly one rotation per BABE epoch.** The BABE client accepts exactly one epoch-change announcement per epoch, in that epoch's first block. `pallet_babe` announces one on every session rotation, so the chain must rotate exactly once per BABE epoch. `ShouldEndSession` rotates at most once per sidechain epoch: if sidechain epochs were skipped (no blocks produced), it catches up in a single rotation, stamping the queued committee with the current epoch so the inherent selects for the following epoch and no further rotation is due until then. With BABE's epoch length bound to the sidechain epoch length, this gives exactly one rotation, and one announcement, per BABE epoch.

### Key Model

**The requirement.** At the flip, every producing-committee member holds a registered, usable BABE key, and BABE's authorities are the BABE keys of the committee that `pallet_session` installs at each rotation (see *Authority Management*). **Cardano is the single source of truth for keys**; the chain holds no competing registration.

**Carrier.** The BABE key is a plain field of `SessionKeys` (`aura`, `grandpa`, `babe`, `beefy`) on stock `pallet_session`. The runtime extracts it from a candidate's registered keys.

**Key rollout.** The BABE key is a real key that operators generate and register, and its uptake gates the switch. It SHOULD be distinct from the AURA key (see *Key reuse* below):

1. **Binary reports readiness.** The node binary, rolled out before the runtime upgrade, loads a BABE key from the keystore and reports whether that key is registered on Cardano for a permissioned candidate (metric `midnight_babe_key_registered`).
2. **Operators register BABE keys.** For federated nodes the BABE key is added to the permissioned-candidates list via a governance action on Cardano; governance's due diligence is what assures the registered key is correct, so no separate proof-of-possession is needed on that path. The operator loads the key into their node's keystore for VRF signing, and the chain observes only the registered public key, so if an operator never loads the matching private key, the design can do no more than let them fail to author.
3. **A BABE key is required for selection.** From the activation runtime onward, session-key extraction requires a BABE key: a candidate whose registration carries no BABE key is filtered out of committee selection at the first rotation after the upgrade. The runtime does not derive a BABE key from the AURA key for registrations.
4. **Scheduling is gated on adoption.** The gate is on-chain: after at least one committee rotation since the upgrade, `pallet_babe::Authorities` MUST equal the committee's registered BABE keys; copied AURA keys or a missing member mean the committee did not rotate onto the registered keys, and governance does not schedule. The `midnight_babe_key_registered` metric is advisory: it reads the plain keystore, so it reports 0 for a validator that authors through the keystore fallback, including one that registered its AURA key as its BABE key. The runtime enforces only the weaker condition that the authority set is non-empty: a scheduled flip is postponed while it is empty.

**One-shot key translation at the upgrade.** The activation migration translates `CurrentCommittee`, `NextCommittee` and `pallet_session` keys to the new `SessionKeys` shape and seeds `QueuedCommittee` from `CurrentCommittee`. For each stored member the BABE key is **copied from the AURA key**, so the committees already in storage stay valid until the first rotation installs the registered keys. This copy is the only runtime-side fallback; it applies once and is superseded at the first rotation. Because of it, a registered BABE key equal to another candidate's AURA key would collide, so registrations MUST be checked for such collisions before the upgrade.

**Transitional keystore fallback (node side).** The node wraps its keystore so that a BABE request for a public key with no usable BABE key falls back to the AURA key with the same public key. The fallback is keyed by the *requested public key*, not by whether any BABE key exists in the store, so a present-but-mismatched BABE key does not suppress it. It only helps a validator whose on-chain BABE authority key equals its AURA public key, which is the state of the copied keys between the upgrade and the first rotation; the node warns when it is in use. It is a safety net for the migration window. It also serves, indefinitely, any validator that registered its AURA key as its BABE key (see *Key reuse*).

**Key reuse.** Reusing a validator's own AURA key as its BABE key is permitted (recorded decision, midnight-node PR #2296): it passes selection, the keystore fallback signs for it, and it stays valid after the first rotation. It forgoes the hygiene benefit of distinct keys, so operators SHOULD register a fresh key. Exit criterion for the fallback: the keystore wrapper can be removed once no committee candidate's registered BABE key equals its AURA key; until then it is part of the steady state, not only of the migration window.

### Node Service

- **Block import and verification:** a single import queue dispatches each block to the AURA or BABE pipeline by the engine that authored it, read from the block header. BABE pre-digests are trusted only when the block's `pallet-version` digest reports a runtime at or above the activation spec version (`3_000_000`); older blocks are treated as plain AURA blocks whatever their header carries, so pre-upgrade history, including warp and gap sync, imports as before. The BABE epoch tree is seeded on the verify path at the flip block.
- **Authoring:** one authoring supervisor runs AURA until it observes the flip on chain, then hands over to BABE; the hand-over is one-way, and a node restarted after the flip starts directly in BABE. The node starts on runtimes without `BabeApi`, so binaries can be rolled out before the runtime upgrade.
- **Pre-digest injection:** the AURA proposer attaches a BABE `SecondaryPlain` pre-runtime digest (same slot and author index) to every AURA block, unconditionally, from binary rollout until the flip.
- **Keystore and readiness:** each operator loads their **distinct BABE key** into the keystore; the migration keystore wrapper (see *Key Model*) covers the window before registered keys are installed. The `midnight_babe_key_registered` metric reports whether the loaded BABE key is registered on Cardano.

## Rationale

### Why BABE over Sassafras

Sassafras (ring VRF, single-leader-per-slot, RFC-0026, peer-reviewed at ACNS 2025) would eliminate collisions entirely, but has **no production-ready client engine** - the pallet sits in polkadot-sdk behind an experimental flag, stripped from stable releases, deployed nowhere. Safrole, its simplified variant inside JAM, runs only on JAM testnets and is not yet on mainnet. BABE is the only production-ready AURA alternative in Substrate and runs Polkadot's relay chain. Migrating to BABE does not preclude a later move to Sassafras/Safrole.

### Why not a DAG-based protocol

MPS PR #82 identifies Narwhal/Bullshark as a path to decoupling dissemination from ordering, which could address throughput limits BABE does not. But DAG protocols require fundamental execution-model changes, have no Substrate integration, and introduce new finality semantics. BABE is the pragmatic short-term fix for the DDoS vulnerability, within the existing framework, without blocking future DAG work.

### Why not re-genesis

Re-genesis (relaunch from a fresh genesis on BABE) would make the switch trivially safe but discards the continuity this MIP exists to preserve - history, live state, users, contracts, the bridge. It is also unclear what re-genesis even means for a zk-contract chain (carrying shielded state and proof-relevant data across a genesis boundary), so it is left to future work.

### Single jump to c = 1/4

We switch straight to VRF (`c = 1/4`) at the flip, rather than AURA → BABE-secondary-only (`c = 0`) → VRF. Seeding the first BABE epoch already at `c = 1/4` means there is no mid-life coefficient change to announce, so the client epoch tree needs no deferral machinery and the migration has one fewer moving part. A staged `c` ramp remains possible later via governance.

### Committee staging on `pallet_session`

BABE commits each epoch's authorities one epoch ahead (it announces the next set at the boundary). AURA needs no look-ahead, so a committee selected mid-epoch could be applied directly at the boundary; BABE needs a stage from which the client can learn the next set in advance. Stock `pallet_session` provides it: its one-session queue holds the committee selected mid-epoch until the next boundary, so the next set is present when BABE announces it. The Ariadne inherent and its selection semantics are untouched.

### Keys: distinct BABE keys, adoption-gated rollout

BABE could have silently reused the AURA key (zero operator action), but that gives no readiness signal and reuses one sr25519 key across AURA sealing and BABE VRF, which is poor hygiene. Instead, operators register BABE keys, distinct by default, the node reports whether its key is registered, and governance schedules the flip only once the committee can actually run BABE. Reuse of a validator's own AURA key is tolerated rather than enforced against, since the registration is governance-vetted and the keystore fallback covers it; the cost is that the fallback has no fixed removal date. Requiring a BABE key at session-key extraction makes registration a hard condition for selection rather than a soft signal; the one-shot copy at the upgrade and the node keystore fallback exist only so that the committee already in storage keeps producing until the first rotation installs the registered keys.

### Tuning c

`c` is a direct tradeoff between DDoS resilience and fork rate. At `c = 1/4`, ~25% of slots are VRF-hidden and the collision rate is ~3.4%. Lowering `c` reduces forks but also the fraction of attack-resistant slots. `c` is governance-adjustable; the initial value should be informed by testnet fork-cost measurements under realistic conditions.

## Path to Active

### Acceptance Criteria

- An in-place AURA → BABE switch (no re-genesis) demonstrated on a testnet with the current validator set.
- The rollout executed end-to-end: vanilla AURA → migration-aware binary (dual digests) → BABE keys registered and observed → single runtime upgrade (activation) → `scheduleFlip` → automatic flip at an epoch boundary.
- BABE-key **adoption is observable**: after a rotation `babe.authorities()` equals the committee's registered BABE keys, and the readiness metric reports per-validator key state. A scheduled flip whose epoch's last slot produced no block is shown to fire at the next epoch's last slot.
- No regression in block-production rate or GRANDPA finality across the transition; partner-chains inherent integration (mc-hash, Ariadne committee selection, bridge) intact before, during, and after.
- Committee rotation continues correctly under BABE - the next epoch's authorities are announced at each boundary (from the session queue), including across a Cardano-epoch boundary.
- Fork rate and collision cost at `c = 1/4` measured and consistent with the model.

### Implementation Plan

**Build.** The runtime, node, and partner-chains changes specified in this MIP are implemented by the engineering team and ship in midnight-node v3.

**Deploy to reach Active.** Reaching Active means performing the in-place migration defined in *Migration Architecture* (binary rollout → BABE keys registered and observed → runtime upgrade → `scheduleFlip` → automatic flip) on each network in turn: **devnet → testnet → mainnet**. Each environment must satisfy the Acceptance Criteria before promotion, and scheduling on each is gated on BABE-key adoption. The first end-to-end validation is a ≥4-node testnet of the current validators (partner-chains data mocked where the testbed requires it).

## Backwards Compatibility Assessment

This is a consensus-mechanism change, but it is performed **in place on the running chain** - no re-genesis and no chain split. AURA and BABE coexist during the transition (blocks routed by pre-digest), giving the network a window to migrate before the switch.

**Node software.** All nodes must run the migration-aware binary **before the runtime upgrade**, not merely before the flip. From the activation block the runtime rejects blocks from authors that do not emit the BABE pre-digest, and a full node on the old binary cannot call the v3 block-verification API and stalls at the activation block. Light clients that verify blocks must likewise upgrade; afterward, un-upgraded nodes cannot validate BABE blocks.

**Session keys.** `SessionKeys` gains a `babe` field, changing its on-chain encoding; the activation migration translates existing committee and `pallet_session` storage to the new layout, copying each stored member's AURA key into the BABE slot. Operators MUST register BABE keys on Cardano (distinct from their AURA keys unless they deliberately reuse them, see *Key Model*) and load them into their nodes **before the upgrade**: a candidate without a registered BABE key is dropped from committee selection at the first rotation after the upgrade. Registration happens on Cardano ahead of time and the chain observes it, so there is no flag-day re-registration on the chain itself; the on-chain check that `babe.authorities()` equals the registered keys after a rotation is the readiness gate, with the registered-key metric as an advisory per-validator signal.

**Unchanged.** GRANDPA finality and the Ariadne committee-selection algorithm are unaffected, and the inherent-data integration (mc-hash, committee selection, bridge) is preserved. The committee pallet gains a record of the queued committee (`QueuedCommittee`) and moves onto stock `pallet_session` (storage version 2); its `set_keys` extrinsic is disabled, since the pallet registers committee keys itself.

## Security Considerations

**Secondary slots remain targetable.** At any `c`, most slots are secondary (deterministic) and predictable like AURA. BABE improves the situation but does not eliminate it - a sustained attacker can still target secondary producers; the VRF-hidden primary slots are what keep the chain progressing under attack.

**Fork depth under collisions.** At `c = 1/4`, collisions are rare (~3.4%) and forks resolve within a slot. Higher `c` raises collision probability and could deepen forks across consecutive collided slots. GRANDPA finality bounds fork depth - a finalized block cannot be reverted.

**VRF randomness.** Epoch randomness derives from the previous epoch's VRF outputs. A small, colluding committee could bias it by selectively withholding blocks - a known BABE property shared with Polkadot, mitigated by committee size.

**Key management.** Compromise of a validator's BABE (VRF) key lets an attacker predict that validator's primary-slot assignments for the epoch, enabling targeted attacks against it. BABE keys must be protected like other session keys.

**Transitional key reuse.** Between the runtime upgrade and the first committee rotation, every stored member's BABE key is a copy of their AURA key, and the node keystore fallback signs BABE requests with the AURA key when no usable BABE key is loaded; one key then serves both AURA sealing and BABE VRF. This is acceptable transitionally but is weaker hygiene than distinct keys; the first rotation installs the registered keys, and the cross-use ends there for every validator that registered a distinct key. A validator that registered its AURA key as its BABE key keeps the cross-use indefinitely (see *Key reuse* in the Key Model). Because a BABE key equal to an AURA key is valid during this window, registrations are checked before the upgrade for a BABE key that collides with another candidate's AURA key.

**No equivocation reporting.** `pallet_babe` is configured with equivocation reporting disabled (`EquivocationReportSystem = ()`), matching the existing GRANDPA and BEEFY configuration, and the node does not generate equivocation proofs. A validator that authors two blocks in one slot is therefore not penalized on chain. Under federated operation the deterrent is removal from the permissioned-candidates list by governance. Whether on-chain accountability is required before block production opens beyond the federation is tracked under *Open Design Questions*.

## Implementation

The migration touches the runtime, the node, and the vendored partner-chains modules. All work is implemented by the engineering team in midnight-node v3.

**Runtime (`midnight-node-runtime`):**
- Add `pallet_babe` to `construct_runtime!` (AURA is retained, not removed), with `EpochDuration` bound to the sidechain epoch length, `ExternalTrigger`, and equivocation reporting disabled; implement `sp_consensus_babe::BabeApi`.
- Add **`pallet-consensus-engine`**: holds `EngineState` (`Aura` / `ScheduledFlip` / `Babe`); exposes `schedule_flip` to the governance origin; commits the flip in the first produced block at the last slot of an epoch (postponed while BABE has no authorities), writing `GenesisSlot` and `EpochConfig`; enforces the pre-digest guards; routes `OnTimestampSet`, `current_slot()` and `FindAuthor` by active engine; exposes `ConsensusEngineApi::active_engine`.
- Add a `babe` field to `SessionKeys`; session-key extraction requires it.
- Stock `pallet_session` (`historical` enabled, `set_keys` disabled); the committee pallet implements `SessionManager` and `ShouldEndSession`; `pallet_authorship` is wired to the engine pallet's `FindAuthor`.
- One activation migration: translate committees and session keys (BABE copied from AURA), seed `QueuedCommittee`, activate the engine pallet; with try-runtime checks.

**Partner-chains committee pallet (`pallet-session-validator-management`):**
- Committee staging `NextCommittee → QueuedCommittee → CurrentCommittee` on stock `pallet_session`, storage version 2; the Ariadne inherent-data provider is unchanged.
- Rotate at most once per sidechain epoch, catching skipped epochs up in a single rotation.

**Node (`midnight-node`):**
- Engine-dispatched verifier and block import (route by the authoring engine; BABE digests trusted only at or above the activation spec version).
- Single authoring supervisor: AURA until the flip, then BABE, one-way.
- BABE `SecondaryPlain` pre-digest attached to every AURA block from binary rollout.
- Migration keystore wrapper (AURA fallback per requested public key) and the `midnight_babe_key_registered` readiness metric.
- Consensus-agnostic partner-chains consensus crates (`PartnerChainsBlockImport`, `PartnerChainsVerifier`, `PartnerChainsProposerFactory`) replacing the forked AURA crates.

**Dependencies:** `pallet-babe`, `sp-consensus-babe`, `sc-consensus-babe`, `sc-consensus-epochs` from polkadot-sdk (pinned `polkadot-stable2609`), used unmodified; partner-chains–specific logic is confined to `pallet-consensus-engine` and the node wrappers so that a fork of the BABE crates is not required. The readiness metric is part of this work; the decision to schedule the flip is a governance action informed by it.

## Testing

Testing spans three layers plus the staged-rollout validation.

**Unit (pallet/runtime).** The consensus-engine state machine: `schedule_flip` succeeds only from `Aura`; the digest guards accept dual-digest blocks in `Aura` and `ScheduledFlip`, reject `Primary` and `SecondaryVRF` variants there, and reject AURA digests in `Babe`; the scheduled flip fires only at the last slot of an epoch and is postponed while the authority set is empty; the flip writes `GenesisSlot` to the next epoch start and `EpochConfig` when absent; epoch 0 is seeded with the committee queued for the epoch it covers. Committee staging: `NextCommittee` matches the computed committee, rotation shifts `Next → Queued → Current`, a skipped sidechain epoch is caught up in a single rotation, and the Cardano-straddle case uses the newer snapshot; its mocked main-chain data MUST be built to span a mainchain epoch boundary, or the case isn't actually exercised. Key plumbing: session-key extraction rejects a candidate without a BABE key and still requires AURA, GRANDPA and BEEFY keys. The activation migration: translates `Current`/`Next` committees and `pallet_session` keys with BABE copied from AURA, seeds `QueuedCommittee`, pre-seeds the `GenesisSlot` sentinel, and passes its try-runtime checks.

**Node.** Engine detection from the header; import dispatch (AURA vs BABE, with pre-activation blocks treated as AURA regardless of their digests); the authoring supervisor hand-over; keystore fallback resolution per requested public key; BABE block verification.

**Integration / e2e (the provable rollout).** On a ≥4-node network of current validators (partner-chains data mocked where needed), exercise the full procedure and assert at each stage:
- *Binary rollout:* AURA undisturbed; every upgraded author's blocks carry the dual pre-digests; the readiness metric is observable.
- *Runtime upgrade:* the activation block runs the migration; committees survive in the new key shape; `babe.genesisSlot()` holds the sentinel; after the next rotation `babe.authorities()` matches the registered keys and `NextEpochData` digests appear at session boundaries; a block from a non-upgraded author is rejected.
- *Schedule + flip:* `scheduleFlip` moves the engine to `ScheduledFlip`; the flip lands on the last slot of an epoch; BABE + VRF live; both primary and secondary blocks produced; next-epoch authorities announced each boundary; GRANDPA keeps finalizing.
- *Regression:* block interval and finality lag unchanged; mc-hash / Ariadne / bridge inherents intact throughout.
- *Gate:* an empty last slot of the epoch while in `ScheduledFlip` defers the flip to the next epoch's last slot. Postponement while `babe.authorities()` is empty is unreachable through governance (scheduling follows a rotation) and is covered by the unit test `flip_postpones_when_babe_authorities_empty`.
- *Cardano-straddle (authoritative):* run long enough on a Cardano-connected testnet (e.g. Preview) to cross real mainchain epoch boundaries, confirming committee staging holds across an actual straddle. The mock above only makes this fast and deterministic in CI; the real run is the authoritative coverage.

**Failure modes to cover** (representative): empty `NextAuthorities` after a session rotation; committee/key encoding mismatch after the migration; an empty last slot of the epoch (flip deferred to a later epoch); a committee order or membership change across the flip epoch (epoch-0 authorities must be the incoming committee, in its order); a validator whose registered BABE key is not loaded in its keystore at the flip; two candidates sharing a session key.

## Open Design Questions

**Driving non-committee verifier upgrades.**
Full nodes and light clients outside the committee cannot validate BABE blocks after the flip unless upgraded, and nothing on-chain gates on their readiness. Short of a well-run upgrade campaign ahead of the flip it is unclear what more can be done - this is an operational problem, not a protocol one.

**Equivocation and accountability.**
Neither AURA before the flip nor BABE after it has on-chain equivocation reporting (see *Security Considerations*). Under federated operation the deterrent is removal from the permissioned-candidates list by governance. Whether on-chain reporting and penalties are a prerequisite for permissionless block production, and for which engines (BABE, GRANDPA, BEEFY), is an open decision owed under the migration's security review.

**Permissionless key registration and observability (downstream).**
Federated nodes register keys via a governance action on Cardano, and the chain already observes them. Permissionless operation will additionally require self-registration with proof-of-possession and on-chain observability of those registrations, with Cardano remaining the source of truth. This is downstream work, addressed when block production opens beyond the federation.

- MPS PR #83: Addressing Censorship Vulnerabilities via Block Producer Anonymity
- MPS PR #82: Addressing Consensus-Related Performance Issues
- midnight-node repository (v3 release): runtime and node support for the AURA to BABE migration
- midnight-node PR #2310: seed BABE epoch 0 with the committee queued for the epoch it covers (requirement 2.7)
- midnight-node #1750: further-ahead Ariadne committee, closed as not needed once the chain moved onto stock `pallet_session`
- paritytech/polkadot-sdk#12754 and midnight-node #1748: the BABE client no longer requires the finalized header to carry a BABE pre-digest, which removed the former bootstrap constraint on in-place migration
- BABE specification - [Web3 Foundation research](https://research.web3.foundation/protocols/block-production/BABE)
- GRANDPA specification - [Web3 Foundation research](https://research.web3.foundation/protocols/finality/grandpa)
- Sassafras RFC-0026 - [Polkadot Fellowship RFCs](https://polkadot-fellows.github.io/RFCs/approved/0026-sassafras-consensus.html)

## Appendix A: Block-Import Validity Requirements by Migration Phase

The BABE import path (`sc-consensus-babe`, pinned `polkadot-stable2609`) enforces several hard invariants on every block it imports; each aborts the import outright. Because the switch is performed in place - on a chain whose history is AURA blocks with no BABE pre-digest - these invariants are not satisfied by construction the way they are on a chain that ran BABE from genesis. This appendix enumerates, per migration phase, the requirements a valid block import must satisfy so the invariants hold throughout the switch. Pre-activation history is unaffected: the node treats any block executed by a runtime older than the activation runtime as a plain AURA block whatever its header carries, so historical sync, including warp sync, imports it as before.

The upstream checks referenced below:

- **(I1)** the imported block's header carries a BABE pre-digest;
- **(I2)** the imported block's **parent** header carries a BABE pre-digest;
- **(I3)** the first block of an epoch carries a `NextEpochData` digest (a first-in-epoch block without one is rejected as a missing epoch change);

The hybrid import and verifier route **by engine-id of the block**, never by the mere presence of a BABE digest.

### Phase 0 - Baseline (migration-aware binary deployed, runtime not yet upgraded)

- **0.1** Every block MUST carry engine-id `aura` and a valid AURA pre-digest and route to the AURA import path; no BABE import checks apply. Blocks authored by upgraded nodes additionally carry an injected BABE `SecondaryPlain` pre-digest, which the pre-upgrade runtime ignores and the import treats as a passenger.
- **0.2** No engine-id `babe` block exists in this phase. The hybrid import MUST apply a deterministic policy (reject) for any engine-id `babe` block, and MUST treat any BABE pre-digest in a block executed by a runtime below the activation spec version as a passenger, so that re-sync of this history is unaffected.

### Phase 1 - Dual-digest window (from the activation block through the flip; `Aura`, then `ScheduledFlip`)

- **1.1** Blocks MUST remain engine-id `aura` and route to the AURA path. The hybrid import/verifier MUST route by engine-id, treating the injected BABE pre-digest as a passenger - a dual-digest AURA block is the normal case in this phase.
- **1.2** Every block from the activation block through the flip MUST carry a valid injected BABE `SecondaryPlain` pre-digest, **with no gaps**; the runtime enforces this by rejecting any block without it. This is the invariant that satisfies **I2** at the flip: the block immediately preceding the flip becomes the first BABE block's parent, so injection must continue on every block up to the flip.
- **1.3** Each injected pre-digest MUST carry the block's actual slot on the shared 6,000 ms clock and the AURA author index, so the strictly-increasing-slot check at the flip holds and the runtime guard accepts the block.
- **1.4** `scheduleFlip` records no target slot; it moves the engine to `ScheduledFlip`, and the flip commits in the first produced block at the last slot of an epoch at which `pallet_babe` has authorities. Until that block `ActiveEngine` remains `aura` and 1.1–1.3 hold.

### Phase 2 - Flip boundary (first BABE block; engine-id `babe`, first-in-epoch for epoch 0)

- **2.1 (I2)** The parent, which is the flip block and the last AURA block, MUST carry a BABE pre-digest (guaranteed by 1.2).
- **2.2 (I1)** The first BABE block MUST carry a valid BABE pre-digest.
- **2.3** `GenesisSlot` is set at the flip to the first slot of the next epoch, which strictly exceeds the flip block's slot (read from the injected pre-digest, per 1.3).
- **2.4 (I3)** The first BABE block MUST carry a `NextEpochData` digest announcing epoch 1's authorities, populated from the committee queued in `pallet_session` at the last rotation; otherwise the import is rejected. The session rotation due in the first BABE block emits this digest. A rotation that lands in the flip block itself is harmless: it deposits a `NextEpochData` digest into an AURA block no BABE client reads, and the first BABE block, in the next epoch, still has its own rotation due.
- **2.5** The BABE import pipeline MUST be installed before any engine-id `babe` block is routed to it; the epoch tree is seeded from the flip block on the verify path.
- **2.6** The flip slot MUST be the last slot of a BABE epoch, so that the first BABE block is first-in-epoch, and the committee for that epoch MUST already be queued, so 2.4's digest is well-defined. The runtime guarantees the former by construction (the flip only fires at the last slot of an epoch) and the latter by postponing the flip while the BABE authority set is empty.
- **2.7** The seeded epoch-0 producing authorities MUST be the committee queued for the epoch that epoch 0 covers, so the set that produces in epoch 0 is the live committee, not the outgoing one. Order matters as much as membership: Ariadne shuffles the committee each epoch, so seeding epoch 0 from the outgoing set would make the authority index in each epoch-0 pre-digest resolve to a different validator than `babe.authorities()` names, and when membership also changes, rotated-out members would keep authoring and new members could not, until epoch 1 realigned the sets. The postponement check in 2.6 therefore waits on the queued set.
- **2.8** Every authority in the seeded epoch-0 set and the announced epoch-1 set MUST resolve to a *usable* BABE key in its validator's keystore. The runtime guarantees presence (a candidate without a registered BABE key is not selected); usability is a node concern: the keystore wrapper falls back to the AURA key per requested public key, and a present-but-malformed BABE key MUST fall through to that fallback rather than leave a member unable to author at the flip.

### Phase 3 - BABE steady state (post-flip)

- **3.1** Every block MUST carry engine-id `babe` and a valid BABE pre-digest and route to the BABE import pipeline with full BABE verification (slot, VRF, seal).
- **3.2 (I2)** The parent of every BABE block carries a BABE pre-digest; the pre-digest chain is unbroken from the last injected AURA block onward.
- **3.3 (I3)** The first block of every epoch carries a `NextEpochData` digest.
- **3.4** AURA authoring MUST be guarded off at the flip; no engine-id `aura` block with a slot after the flip slot may exist (the runtime rejects an AURA pre-digest in `Babe`), and forks straddling the boundary resolve to the BABE chain.

## Acknowledgements

- Tomasz Bartos (@Klapeyron)
- Mike Clay (@m2ux)
- Giles Cope (@gilescope)
- Justin Frevert (@justinfrevert)
- Lech Głowiak (@LGLO)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
