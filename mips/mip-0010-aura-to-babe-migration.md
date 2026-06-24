---
MIP: "0010"
Title: Consensus Migration from Aura to BABE
Authors:
  - Dominik Zajkowski (dzajkowski)
Status: Proposed
Category: Core
Created: 2026-06-08
Requires: none
Replaces: none
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

Midnight's current Aura consensus uses a deterministic, rotating leader schedule across a small, permissioned committee. The next block producer is publicly predictable, so an attacker can direct a surgical DDoS at a single known node and stall the chain for that slot — at constant, low cost across consecutive slots.

This MIP migrates block production from Aura to **BABE**. In BABE each slot has a deterministic secondary author (Aura-like) and may also have a VRF-elected **primary** author whose identity is hidden until the block is published. The active slot coefficient *c* sets the fraction of VRF-hidden slots: higher *c* means more hidden slots but more slot-ownership collisions (short-lived forks). Suppressing a VRF-assigned slot requires DDoSing the **entire** committee, so attack cost scales with committee size rather than staying constant. BABE is production-grade and runs Polkadot's relay chain. Finality remains GRANDPA, unchanged.

The migration is performed **in place on a running chain** — no re-genesis — with Aura and BABE coexisting during a phased switch. This MIP does **not** address block-propagation limits, on-chain proof size, or throughput; those are scoped to the performance MPS (MPS PR #82).

This MIP is one step in a larger roadmap toward decentralization. It is necessary but not sufficient for full permissionless operation: BABE's hidden-leader property is a prerequisite for opening block production to unknown validators, but does not on its own establish that doing so is safe. The migration to BABE is performed and proven entirely under federated operation, ahead of any permissionless opening.

## Motivation

**Targeted DDoS vulnerability (MPS PR #83).** Aura's leader schedule is deterministic and public. With some effort, an observer can discover the physical location of the next block producer and direct a surgical DDoS at that single node, stalling the chain for that slot. The cost of sustaining this across consecutive slots is constant — the schedule is known in advance.

**Small committee vulnerability (MPS PR #83).** Even with leader anonymity, full committee membership is public. If the committee is small, a coordinated DDoS against all members is economically viable. Under Aura, committee size is irrelevant to *targeted* attacks — an attacker only needs to hit one known node. BABE makes committee size matter: suppressing a VRF-assigned slot requires attacking the whole committee.

## Specification

BABE becomes the block-production engine; GRANDPA finality and the Ariadne committee-selection algorithm are unchanged — the migration only deepens the committee pallet's look-ahead staging by one slot (see *Authority Management*). The migration is staged on a running chain (see *Migration Architecture* below). This section first specifies BABE's steady-state behavior after the switch, then the migration mechanism.

Identifiers throughout this document (pallet, trait, method, and storage names) are illustrative; final naming is an engineering decision and is not prescribed by this design.

### BABE Configuration

- **Slot duration:** 6,000 ms (unchanged from Aura).
- **Epoch length:** unchanged from the current environment configuration; it sets how often VRF randomness refreshes and authority sets rotate.
- **Active slot coefficient *c* = 1/4**, applied from the **first** BABE epoch (a single jump — there is no intermediate secondary-only / `c = 0` epoch). `c` is held in `pallet_babe`'s `EpochConfig`, seeded at the switch, and adjustable thereafter via governance (`NextEpochConfig`) without a code change. `allowed_slots = PrimaryAndSecondaryPlainSlots`.

### Slot Assignment

Each slot has two potential authors:

1. **Primary (VRF):** each validator evaluates a VRF against the epoch randomness and its secret key; if the output is below a threshold derived from *c*, it is the primary author for that slot. The winner is unknown to others until the block is published with its VRF proof. Multiple validators may win one slot (a collision), producing short-lived forks resolved by fork choice.
2. **Secondary (deterministic):** a fallback author assigned round-robin — mechanically identical to Aura. A secondary block loses fork-choice priority to any primary block in the same slot.

### Fork Choice

BABE uses longest-chain with primary-block priority: between equal-length chains, the one with more primary blocks wins. GRANDPA finalizes independently — once finalized, a block cannot be reverted.

### Collision Costs

With a per-validator primary threshold `p = 1 − (1−c)^(1/n)` (set so `P(slot has ≥1 primary) = c`), the collision rate `P(≥2 primary winners) = 1 − (1−c) + (1−c)·ln(1−c)` depends on `c` alone, not committee size `n`. At *c* = 1/4 (0.25) that is ~3.4%, producing ~34 collisions per 1,000 blocks. Each collision wastes the losing block's bandwidth and the importing nodes' verification work. At 2 MB blocks of shielded transactions (~136 transactions, ~544 proofs per block):

| | Per collision | Per 1,000 blocks (34 collisions) |
|---|---|---|
| Bandwidth | 2 MB | 68 MB |
| Verification (sequential) | 1.7 s | 57.8 s |
| Verification (8 cores, parallel) | 211 ms | 7.2 s |

Sequential verification makes each collision expensive (1.7 s of CPU discarded); parallel verification makes it manageable. The proof count and verification times here come from the throughput experiments in MPS PR #82 (to become its own MIP); the millisecond figures are indicative — a token value measured on the specific machine the tests ran on, not an absolute. Mitigation of verification cost is scoped to that work.

### Migration Architecture

The switch is performed in place, without re-genesis. Aura and BABE are both compiled into the runtime and node; blocks are routed by the engine ID in their pre-runtime digest. A new pallet, **`pallet-consensus-engine`**, holds an `ActiveEngine` flag (default `aura`) and drives the transition.

**Coexistence.** `pallet_aura` is **not removed** — it keeps authoring until the flip. `pallet_babe` is added but is *not* wired as a session handler (see *Key Model* below). A conditional `OnTimestampSet` adapter and an `active_current_slot()` helper route slot bookkeeping to whichever engine is active, so the sidechain epoch calculation (and committee rotation) stay correct before and after the switch.

**State machine.** `ActiveEngine` moves `aura → BABE` exactly once, at an epoch boundary, driven by two self-guarded, idempotent runtime-upgrade migrations plus an automatic flip:

1. **Upgrade #1 — bootstrap ("arm"):** set `BootstrapArmed`; seed the static BABE config (`EpochConfig` `c = 1/4`, initial randomness); run the `AuthorityKeys` migration (below); arm BABE pre-digest injection. Does **not** seed authorities and does **not** flip.
2. **Upgrade #2 — schedule:** record the next epoch-boundary slot (`FlipAtSlot`). Engine stays Aura until that slot.
3. **Flip (automatic, at the boundary):** seed BABE epoch state — authorities converted from the current committee, `GenesisSlot` aligned to the boundary, `EpochIndex = 0` — and set `ActiveEngine = BABE`. From the first BABE epoch, `c = 1/4` (VRF live).

While not yet BABE, a guard keeps `pallet_babe`'s epoch state inert. After the flip, an `on_finalize` fallback enacts the epoch change if a partner-chains session rotation doesn't land exactly on the BABE epoch boundary, and backfills the next-epoch authorities (see *Authority Management* below).

**Bootstrap constraint.** The node's BABE block import calls `prune_finalized`, which reads the **finalized** header and requires it to carry a BABE pre-digest. So a finalized BABE-pre-digest block must exist *before* the engine flips. During the armed window (between upgrade #1 and the flip), Aura blocks therefore carry an injected BABE `SecondaryPlain` pre-digest; once GRANDPA finalizes one, the node can construct the BABE import. This is why arming must precede scheduling — it gives finality time to bury a BABE-pre-digest block. The full set of block-import invariants this constraint belongs to, and the requirements that satisfy them at each migration phase, are enumerated in *Appendix A*.

### Authority Management (two-deep committee staging)

BABE must announce the **next** epoch's authority set *at* the epoch boundary (via the `NextEpochData` digest) so clients can validate the next epoch in advance. Partner-chains stages exactly **one** committee ahead, and delivers it *mid-epoch* via an inherent — so at a boundary the next committee isn't yet available. To bridge this, the committee pallet stages **two** ahead: `NextCommittee` (current + 1) and `NextNextCommittee` (current + 2). At each rotation they shift (`NextNext → Next → Current`), guaranteeing the next set is populated at the boundary.

This is always computable: a committee's inputs come from Cardano data offset by 2 mainchain epochs (finalized), and a sidechain epoch (minutes–hours) is far shorter than a Cardano epoch (5 days), so a bounded look-ahead sits well within the stable margin — including the Cardano-epoch straddle, where the further committee is computed from the newer, already-final snapshot.

`pallet_babe` is **not** a session handler: partner-chains passes empty `queued_validators` to session handlers, which would zero BABE's next authorities. Instead, `pallet-consensus-engine` derives BABE's current and next authorities directly from committee storage at each boundary, mapping each member's babe key in committee order.

### Key Model

**The requirement.** Mechanism-independent: at the flip, every producing-committee member holds a registered, proven, usable babe key (the aura fallback covering the rest), and babe's authorities are sourced from committee storage (see *Authority Management*), not the session pallet. The rollout, proof-of-possession, fallback, and flip gate below follow from this requirement, not from any particular key carrier. **Cardano is the single source of truth for keys**; the chain holds no competing registration.

How the babe key rides in session keys is an engineering decision left to implementation: a phantom holder over the current `pallet_partner_chains_session` keys, or a plain generic session key if the work that removes the custom session pallet lands first. The choice does not change the requirement above; the generic-session-keys path is the cleaner enabler.

**Adoption-driven key rollout.** The babe key is a real, distinct key that operators opt into, and its uptake gates the switch:

1. **Binary enables babe keys.** The node binary — rolled out *before* any runtime switch — teaches the chain to process registrations that carry a babe key: `CandidateKeys` may include a `BABE` entry, and `SessionKeys::maybe_from` extracts it.
2. **Operators register babe keys.** For federated nodes the babe key is added via a governance action on Cardano; governance's due diligence is what assures the registered key is correct, so no separate proof-of-possession is needed on that path. Self-registration — the permissionless path — has no such vetting, so there the operator MUST prove possession of the babe key at registration (a proof-of-possession over the registered public key); see *Open Design Questions*. Either way the operator loads the key into their node's keystore for VRF signing, and the chain observes only the registered public key — so if an operator never loads the matching private key, the design can do no more than let them fail to author.
3. **Adoption is measurable.** Because babe keys arrive through registrations, the chain reports how many committee candidates carry one — a direct readiness signal. Adoption counts registered babe keys — governance-vetted for federated nodes, proven by proof-of-possession for self-registration; a candidate without a counted key relies on the transitional fallback.
4. **The flip is gated on adoption.** Governance does not schedule the switch (upgrade #2) until a sufficient share of the committee has registered babe keys (optionally, the schedule migration itself refuses below a threshold). The chain only moves to BABE once the committee can actually produce and validate BABE blocks.

**Transitional fallback (safety net).** Until an operator registers a distinct babe key, `maybe_from` derives babe from that member's aura sr25519 key (`find(BABE).or_else(find(AURA))`). This keeps not-yet-migrated candidates valid in selection and ensures the committee never holds a member with no usable babe key. It is a safety net for the transition window, not the end state — adoption is tracked against distinct registered babe keys.

**Encoding migration.** If the carrier adds a field to `SessionKeys` (the phantom-holder path), its on-chain encoding changes; a versioned `AuthorityKeys` migration (run in upgrade #1) translates the stored committees (`Current`/`Next`/`NextNext`) and `pallet_session` keys to the new layout, idempotent via a version marker.

### Node Service

- **Block import:** a hybrid import wraps the existing GRANDPA import and routes by pre-digest — Aura blocks to the Aura/GRANDPA path, BABE blocks to a `BabeBlockImport` that computes the epoch descriptor for network blocks and tracks the epoch tree. The BABE import is installed lazily (see *Authoring & keystore*).
- **Import-queue verifier:** a hybrid verifier routes by pre-digest — Aura blocks get full partner-chains Aura verification; BABE blocks get BABE verification (slot, VRF, and seal checks). Block execution at import additionally runs the runtime's inherent checks.
- **Authoring & keystore:** a lazy BABE task waits for `ActiveEngine == BABE` *and* a finalized BABE-pre-digest block, then builds `BabeBlockImport` and starts BABE authoring; the existing Aura authoring is wrapped in a guard that stops it after the flip. Each operator loads their **distinct babe key** into the keystore (the aura key is aliased under the BABE key-type only as the transitional fallback).

## Rationale

### Why BABE over Sassafras

Sassafras (ring VRF, single-leader-per-slot, RFC-0026, peer-reviewed at ACNS 2025) would eliminate collisions entirely, but has **no production-ready client engine** — the pallet sits in polkadot-sdk behind an experimental flag, stripped from stable releases, deployed nowhere. Safrole, its simplified variant inside JAM, runs only on JAM testnets and is not yet on mainnet. BABE is the only production-ready Aura alternative in Substrate and runs Polkadot's relay chain. Migrating to BABE does not preclude a later move to Sassafras/Safrole.

### Why not a DAG-based protocol

MPS PR #82 identifies Narwhal/Bullshark as a path to decoupling dissemination from ordering, which could address throughput limits BABE does not. But DAG protocols require fundamental execution-model changes, have no Substrate integration, and introduce new finality semantics. BABE is the pragmatic short-term fix for the DDoS vulnerability, within the existing framework, without blocking future DAG work.

### Why not re-genesis

Re-genesis (relaunch from a fresh genesis on BABE) would make the switch trivially safe but discards the continuity this MIP exists to preserve — history, live state, users, contracts, the bridge. It is also unclear what re-genesis even means for a zk-contract chain (carrying shielded state and proof-relevant data across a genesis boundary), so it is left to future work.

### Single jump to c = 1/4

We switch straight to VRF (`c = 1/4`) at the flip, rather than Aura → BABE-secondary-only (`c = 0`) → VRF. Seeding the first BABE epoch already at `c = 1/4` means there is no mid-life coefficient change to announce, so the client epoch tree needs no deferral machinery and the migration has one fewer moving part. A staged `c` ramp remains possible later via governance.

### Two-deep committee staging

BABE commits each epoch's authorities one epoch ahead (it announces the next set at the boundary). Partner-chains — built for Aura, which needs no look-ahead — stages one committee ahead, delivered mid-epoch, so the next set isn't ready at the boundary. Rather than hack the consumer (e.g. re-announcing mid-epoch, which stock BABE clients don't expect), we **align the producer**: stage two committees ahead so the next set is present at the boundary. The underlying Cardano data is offset by two mainchain epochs (≫ the look-ahead), so staging deeper is always computable.

### Keys: phantom holder + adoption-gated rollout

Babe could have silently reused the aura key (zero operator action), but that gives no readiness signal and reuses one sr25519 key across Aura sealing and BABE VRF — poor hygiene. Instead, operators register **distinct** babe keys, the chain measures uptake, and the flip is gated on it, so the switch happens only when the committee can actually run BABE. The **phantom holder** lets the babe key live in `SessionKeys` (registered, migrated, and rotated like any key) without making `pallet_babe` a session handler — which matters because partner-chains feeds session handlers empty `queued_validators`. The aura fallback keeps the transition non-blocking (unregistered candidates stay valid) without being the end state.

### Tuning c

`c` is a direct tradeoff between DDoS resilience and fork rate. At `c = 1/4`, ~25% of slots are VRF-hidden and the collision rate is ~3.4%. Lowering `c` reduces forks but also the fraction of attack-resistant slots. `c` is governance-adjustable; the initial value should be informed by testnet fork-cost measurements under realistic conditions.

## Path to Active

### Acceptance Criteria

- An in-place Aura → BABE switch (no re-genesis) demonstrated on a testnet with the current validator set.
- The staged rollout executed end-to-end: vanilla Aura → BABE-capable binary → measured babe-key adoption → arm → schedule → automatic flip.
- Babe-key **adoption is observable**, and the flip is shown to be gated on it (the chain does not switch before a sufficient share of the committee carries babe keys).
- No regression in block-production rate or GRANDPA finality across the transition; partner-chains inherent integration (mc-hash, Ariadne committee selection, bridge) intact before, during, and after.
- Committee rotation continues correctly under BABE — the next epoch's authorities are announced at each boundary (two-deep staging), including across a Cardano-epoch boundary.
- Fork rate and collision cost at `c = 1/4` measured and consistent with the model.

### Implementation Plan

**Build.** The runtime, node, and partner-chains changes specified in this MIP are implemented by the engineering team; the design phase completes by end of June 2026.

**Deploy to reach Active.** Reaching Active means performing the in-place migration procedure defined in *Migration Architecture* — binary upgrade → measure babe-key adoption → arm (upgrade #1) → schedule (upgrade #2) → automatic flip — on each network in turn: **devnet → testnet → mainnet**. Each environment must satisfy the Acceptance Criteria before promotion, and the flip on each is gated on sufficient babe-key adoption. The first end-to-end validation is a ≥4-node testnet of the current validators (partner-chains data mocked where the testbed requires it).

## Backwards Compatibility Assessment

This is a consensus-mechanism change, but it is performed **in place on the running chain** — no re-genesis and no chain split. Aura and BABE coexist during the transition (blocks routed by pre-digest), giving the network a window to migrate before the switch.

**Node software.** All validators must run BABE-capable node software **before** the flip. Because the binary handles babe-key registrations and the hybrid import/verifier, it is rolled out *ahead of* the runtime upgrades (binary-before-runtime ordering). Non-validator full nodes and light clients that verify blocks must likewise upgrade before the flip; afterward, un-upgraded nodes cannot validate BABE blocks.

**Session keys.** `SessionKeys` gains a `babe` field, changing its on-chain encoding; the `AuthorityKeys` migration translates existing committee and `pallet_session` storage to the new layout. Operators are *expected* to register **distinct** babe keys on Cardano and load them into their nodes — but this is **non-blocking**: until they do, the validator's aura key is reused for babe (the fallback), so unregistered candidates remain valid in committee selection. There is **no flag-day re-registration** — babe-key adoption is the readiness signal, and the flip is gated on it, so the switch activates only once enough of the committee has migrated.

**Unchanged.** GRANDPA finality and the Ariadne committee-selection algorithm are unaffected, and the inherent-data integration (mc-hash, committee selection, bridge) is preserved. The committee pallet's only change is one added look-ahead slot — additive.

## Security Considerations

**Secondary slots remain targetable.** At any `c`, most slots are secondary (deterministic) and predictable like Aura. BABE improves the situation but does not eliminate it — a sustained attacker can still target secondary producers; the VRF-hidden primary slots are what keep the chain progressing under attack.

**Fork depth under collisions.** At `c = 1/4`, collisions are rare (~3.4%) and forks resolve within a slot. Higher `c` raises collision probability and could deepen forks across consecutive collided slots. GRANDPA finality bounds fork depth — a finalized block cannot be reverted.

**VRF randomness.** Epoch randomness derives from the previous epoch's VRF outputs. A small, colluding committee could bias it by selectively withholding blocks — a known BABE property shared with Polkadot, mitigated by committee size.

**Key management.** Compromise of a validator's babe (VRF) key lets an attacker predict that validator's primary-slot assignments for the epoch, enabling targeted attacks against it. Babe keys must be protected like other session keys.

**Transitional key reuse.** Until an operator registers a distinct babe key, the fallback reuses their aura sr25519 key as the babe key — one key then serves both Aura sealing and BABE VRF during the transition. This is acceptable transitionally but is weaker hygiene than distinct keys; the adoption-driven rollout exists precisely to move validators onto distinct babe keys, and the cross-use ends for a validator once it registers one.

## Implementation

The migration touches the runtime, the node, and the vendored partner-chains modules. All work is implemented by the engineering team.

**Runtime (`midnight-node-runtime`):**
- Add `pallet_babe` to `construct_runtime!` (Aura is retained, not removed); implement `sp_consensus_babe::BabeApi`.
- Add **`pallet-consensus-engine`**: tracks `ActiveEngine`; holds the arm/schedule/flip state; seeds BABE epoch state at the flip; drives epoch changes (`EpochChangeTrigger = ExternalTrigger`) and backfills authorities; sources BABE authorities from committee storage.
- Add a `babe` field to `SessionKeys` via a phantom holder; extend `maybe_from` with the aura fallback; add the versioned `AuthorityKeys` migration.
- Route timestamp/slot bookkeeping by active engine (conditional `OnTimestampSet`, `active_current_slot`).
- Self-guarded `OnRuntimeUpgrade` migrations for **arm** and **schedule**.

**Partner-chains committee pallet:**
- Two-deep committee staging: `NextNextCommittee` storage, the inherent that fills it, and rotation (`NextNext → Next → Current`).
- Extend the Ariadne inherent-data provider to compute the further-ahead committee.

**Partner-chains Aura authoring (`substrate-extensions/aura`):**
- Inject a BABE `SecondaryPlain` pre-digest into Aura-authored blocks while the bootstrap is armed (satisfies the `prune_finalized` bootstrap constraint).

**Node (`midnight-node`):**
- Hybrid block import (route by pre-digest; lazily install `BabeBlockImport` over GRANDPA).
- Hybrid import-queue verifier (Aura → Aura verification; BABE → BABE verification).
- Lazy BABE authoring task + an engine guard that stops Aura authoring after the flip.
- Process babe-bearing registrations and expose babe-key **adoption** for the flip gate.

**Dependencies:** `pallet-babe`, `sp-consensus-babe`, `sc-consensus-babe`, `sc-consensus-epochs` from polkadot-sdk (pinned `polkadot-stable2603`), used unmodified — partner-chains–specific logic is confined to `pallet-consensus-engine` and the node wrappers so that a fork of the BABE crates is not required. The adoption metric and the flip-gate threshold are part of this work.

## Testing

Testing spans three layers plus the staged-rollout validation.

**Unit (pallet/runtime).** The consensus-engine state machine: the armed-window guard keeps BABE inert; arm seeds config without flipping; the scheduled flip fires only at the boundary; authority conversion preserves keys; the epoch-change fallback and authority backfill behave. Two-deep staging: `NextNextCommittee` matches the computed committee, rotation shifts correctly, and the Cardano-straddle case uses the newer snapshot — its mocked main-chain data MUST be built to span a mainchain epoch boundary, or the case isn't actually exercised. Key plumbing: `maybe_from` uses a registered babe key when present and the aura fallback otherwise, and still requires aura + grandpa. The `AuthorityKeys` migration: translates `Current`/`Next`/`NextNext` committees and `pallet_session` keys, and is idempotent.

**Node.** Engine detection from the pre-digest; hybrid import routing (Aura vs BABE, with GRANDPA fallback before BABE init); the engine guard (runs when active, stops after the flip); BABE block verification.

**Integration / e2e (the provable rollout).** On a ≥4-node network of current validators (partner-chains data mocked where needed), exercise the full procedure and assert at each stage:
- *Binary upgrade:* Aura undisturbed; babe-key registrations processed; adoption observable.
- *Arm:* a BABE-pre-digest block is finalized; `NextNextCommittee` warms; Aura still healthy.
- *Schedule + flip:* the flip lands on the epoch boundary; BABE + VRF live; both primary and secondary blocks produced; next-epoch authorities announced each boundary; GRANDPA keeps finalizing.
- *Regression:* block interval and finality lag unchanged; mc-hash / Ariadne / bridge inherents intact throughout.
- *Gate:* the flip does not occur below the adoption threshold.
- *Cardano-straddle (authoritative):* run long enough on a Cardano-connected testnet (e.g. Preview) to cross real mainchain epoch boundaries, confirming two-deep staging holds across an actual straddle. The mock above only makes this fast and deterministic in CI; the real run is the authoritative coverage.

**Failure modes to cover** (representative): no finalized BABE pre-digest when the BABE import is constructed; empty `NextAuthorities` after a session rotation; committee/key encoding mismatch after the migration; flip landing off the epoch boundary; a validator missing its babe key at the flip.

## Open Design Questions

**Guaranteeing partner-chains committee rotation coincides with the BABE epoch boundary.**
BABE rejects the first block of an epoch if it lacks a `NextEpochData` digest, so the next committee must be populated at every boundary, and the design currently holds this by equal BABE/sidechain epoch lengths plus an `on_finalize` fallback for rotations that miss. That makes coincidence a config equality patched at runtime, not a structural invariant, so an asymmetric epoch-length change or a rotation cadence not slot-locked to the BABE counter could desync it. Open question: bind both boundaries to a single source so they provably coincide rather than relying on equal configuration and a fallback.

**Driving non-committee verifier upgrades.**
Full nodes and light clients outside the committee cannot validate BABE blocks after the flip unless upgraded, and nothing on-chain gates on their readiness. Short of a well-run upgrade campaign ahead of the flip it is unclear what more can be done — this is an operational problem, not a protocol one, and it grows with the permissionless opening.

**Carrier for the babe session key: phantom holder vs generic session keys.**
In-flight partner-chains work covers two related tracks — removing the custom `pallet_partner_chains_session`, and introducing generic session-key management. If that work is deliverable this quarter it becomes a prerequisite, and babe rides as a plain generic session key with the phantom holder dropping out; if it cannot be guaranteed, the self-contained phantom-holder carrier keeps the migration unblocked. The requirement is unchanged either way (registered, proven, usable babe keys at the flip) — this is only a carrier-and-sequencing choice.

**Permissionless key registration and observability (downstream).**
Federated nodes register keys via a governance action on Cardano, and the chain already observes them. Permissionless operation will additionally require self-registration with proof-of-possession and on-chain observability of those registrations, with Cardano remaining the source of truth. This is downstream work, addressed when block production opens beyond the federation.

- MPS PR #83: Addressing Censorship Vulnerabilities via Block Producer Anonymity
- MPS PR #82: Addressing Consensus-Related Performance Issues
- BABE specification — [Web3 Foundation research](https://research.web3.foundation/protocols/block-production/BABE)
- GRANDPA specification — [Web3 Foundation research](https://research.web3.foundation/protocols/finality/grandpa)
- Sassafras RFC-0026 — [Polkadot Fellowship RFCs](https://polkadot-fellows.github.io/RFCs/approved/0026-sassafras-consensus.html)

## Appendix A: Block-Import Validity Requirements by Migration Phase

The BABE import path (`sc-consensus-babe`, pinned `polkadot-stable2603`) enforces several hard invariants on every block it imports. Three abort the import outright, and one panics: `prune_finalized` reads the **finalized** header and aborts the node if that header lacks a BABE pre-digest. Because the switch is performed in place — on a chain whose history is Aura blocks with no BABE pre-digest — these invariants are not satisfied by construction the way they are on a chain that ran BABE from genesis. This appendix enumerates, per migration phase, the requirements a valid block import must satisfy so the invariants hold throughout the switch.

The upstream checks referenced below:

- **(I1)** the imported block's header carries a BABE pre-digest;
- **(I2)** the imported block's **parent** header carries a BABE pre-digest;
- **(I3)** the first block of an epoch carries a `NextEpochData` digest (a first-in-epoch block without one is rejected as a missing epoch change);
- **(I4)** `prune_finalized` reads the current finalized header and requires it to carry a BABE pre-digest — it runs only while importing a block that announces an epoch change.

The hybrid import and verifier route **by engine-id of the block**, never by the mere presence of a BABE digest.

### Phase 0 — Baseline (BABE-capable binary deployed, bootstrap not armed)

- **0.1** Every block MUST carry engine-id `aura` and a valid Aura pre-digest and route to the Aura import path; no BABE import checks apply.
- **0.2** No engine-id `babe` block exists in this phase. The hybrid import MUST apply a deterministic policy (reject) for any engine-id `babe` block received before `BabeBlockImport` is installed.

### Phase 1 — Armed window (after Upgrade #1, through the flip; Upgrade #2 applied within)

- **1.1** Blocks MUST remain engine-id `aura` and route to the Aura path. The hybrid import/verifier MUST route by engine-id, treating the injected BABE pre-digest as a passenger — a dual-digest Aura block is the normal case in this phase.
- **1.2** Every block authored from the arm point through the flip MUST carry a valid injected BABE `SecondaryPlain` pre-digest, **with no gaps**. This is the invariant that satisfies **I2** and **I4** at the flip: the block immediately preceding the flip becomes the first BABE block's parent, and a finalized block from this window becomes the finalized head at the first epoch-change import. Injection that stops once a single block is finalized is insufficient — it must continue on every block up to the flip.
- **1.3** Each injected pre-digest MUST carry the block's actual slot on the shared 6,000 ms clock, so the strictly-increasing-slot check at the flip holds.
- **1.4** The flip MUST NOT take effect until GRANDPA has finalized at least one injected block; this is the precondition for installing `BabeBlockImport`. Arming MUST precede scheduling with sufficient finality headroom.
- **1.5** Upgrade #2 records `FlipAtSlot` at a future BABE epoch boundary; until that slot `ActiveEngine` remains `aura` and 1.1–1.3 hold.

### Phase 2 — Flip boundary (first BABE block; engine-id `babe`, first-in-epoch for epoch 0)

- **2.1 (I2)** The parent — the last armed Aura block before `FlipAtSlot` — MUST carry a BABE pre-digest (guaranteed by 1.2).
- **2.2 (I1)** The first BABE block MUST carry a valid BABE pre-digest.
- **2.3** `GenesisSlot` MUST be aligned so the boundary slot strictly exceeds the parent's slot (read from the injected pre-digest, per 1.3).
- **2.4 (I3)** The first BABE block MUST carry a `NextEpochData` digest announcing epoch 1's authorities, populated from committee storage via two-deep staging; otherwise the import is rejected. The external epoch-change trigger MUST emit this digest at the seeded epoch-0 genesis block, not only at subsequent boundaries.
- **2.5 (I4)** At import, the finalized head MUST carry a BABE pre-digest (guaranteed by 1.4 plus the install gate).
- **2.6** `BabeBlockImport` MUST be installed before any engine-id `babe` block is routed to it.
- **2.7** `FlipAtSlot` MUST be a BABE epoch boundary coinciding with a committee rotation whose next authority set is already populated, so 2.4's digest is well-defined.
- **2.8** The seeded epoch-0 producing authorities MUST be derived from the current committee at the boundary, so the set that produces in epoch 0 is the live committee, not a stale snapshot.
- **2.9** Every authority in the seeded epoch-0 set and the announced epoch-1 set MUST resolve to a *usable* babe key: key resolution MUST validate the key, not merely find one present, and a present-but-malformed babe key MUST fall through to the transitional fallback rather than leave a member with an unusable key at the flip.

### Phase 3 — BABE steady state (post-flip)

- **3.1** Every block MUST carry engine-id `babe` and a valid BABE pre-digest and route to `BabeBlockImport` with full BABE verification (slot, VRF, seal).
- **3.2 (I2)** The parent of every BABE block carries a BABE pre-digest; the pre-digest chain is unbroken from the last injected Aura block onward.
- **3.3 (I3)** The first block of every epoch carries a `NextEpochData` digest.
- **3.4 (I4)** `prune_finalized` is always safe; the finalized head is at or beyond the 1.4 gate block.
- **3.5** Aura authoring MUST be guarded off at or before `FlipAtSlot`; no engine-id `aura` block with `slot >= FlipAtSlot` may exist, and forks straddling the boundary resolve to the BABE chain.

### Cross-cutting — Historical re-sync

- **R.1 (I4 under sync)** The finalized-head requirement (2.5 / 3.4) MUST hold for *every* import of an epoch-change BABE block, including by a node syncing from genesis whose finality lags behind block import. Pre-arm history carries no injected pre-digests, so a lagging syncing node could run `prune_finalized` against a pre-arm finalized header. The design MUST ensure this cannot occur (e.g., via the relationship between block import and finality import during sync). This requirement is the least likely to be satisfied incidentally and MUST be covered explicitly in testing.

## Acknowledgements

- Tomasz Bartos (@Klapeyron)
- Mike Clay (@m2ux)
- Giles Cope (@gilescope)
- Justin Frevert (@justinfrevert)
- Lech Głowiak (@LGLO)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
