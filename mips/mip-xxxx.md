---
MIP: TBD
Title: Node-Side Surfacing of Ledger Events via frame_system::Events
Authors:
  - Mike Clay (m2ux)
Status: Draft
Category: Core
Created: 2026-05-18
Requires: MIP-???? — Compact ↔ Ledger event-schema sharing protocol [pre-numbering]
Replaces:
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

This MIP proposes a candidate solution to [MPS-0007: Node-Side Visibility of Ledger Events](../mps/mps-0007-node-event-visibility.md). The Midnight node receives a stream of per-transaction events from the ledger crate during transaction application but discards them at the ledger-bridge boundary before they reach any observable surface. This proposal specifies their publication on Substrate's existing `frame_system::Events` channel, wrapped in a new `pallet_midnight::Event::LedgerEvent` variant whose outer shape is stable across ledger-crate upgrades. A consumer using standard Substrate tooling — `polkadot.js`, `subxt`, or the `state_subscribeStorage` RPC — receives the events in the order they were produced, decoded against the pallet's metadata, with the per-event content carried as the ledger crate's own tagged-byte serialisation so that downstream decoders evolve with the ledger crate rather than with the runtime ABI. The change is non-protocol — it adds no new gas costs, no new consensus rules, and no new block-validity constraints — and ships in a normal runtime upgrade alongside a versioned host-fn (`apply_transaction` `#[version(3)]`) that preserves replay against historical WASM runtimes.

## Motivation

[MPS-0007: Node-Side Visibility of Ledger Events](../mps/mps-0007-node-event-visibility.md) is the upstream Midnight Problem Statement that this MIP addresses. MPS-0007 frames the gap in detail: the ledger crate's `state.apply(verified_tx, ctx)` returns a `TransactionResult` whose `Success` and `PartialSuccess` variants carry the typed event records produced during transaction execution, but at the midnight-node ledger-bridge those event vectors are pattern-matched to `_` and discarded before the pallet builds its applied-state-root. The pallet therefore deposits FRAME events derived from the bridge's settled fields (`state_root`, `tx_hash`, call/deploy/maintain addresses, claim rewards, unshielded UTXO deltas) but emits nothing reflecting the contract-internal events the network actually computed.

The consequence for downstream consumers is the one MPS-0007 enumerates as use cases UC1 through UC5: contract authors cannot see the events their own contracts emit through `polkadot.js`, block explorers cannot display per-event payloads without standing up a separate indexer, light-client DApps have no path to subscribe to events without trusting a hosted service, operators have no lightweight visibility for incident response, and cross-tooling integrations break at the Midnight boundary because the standard Substrate event pipeline is empty. Today the only consumer that recovers these events is the `midnight-indexer`, which does so by re-running each transaction against its own copy of the ledger crate.

This MIP closes the gap MPS-0007 identifies. It specifies the bridge-layer change that propagates events from `apply_verified_transaction` through a versioned `TransactionAppliedStateRoot`, the new pallet event variant that deposits them onto `frame_system::Events`, the encoding choice for per-event payloads (an opaque tagged-byte field so the wrapper does not churn with the ledger crate's event taxonomy), and the host-fn versioning that preserves replay of historical blocks. It is the "Node-side ledger event surfacing" proposal MPS-0007's `## Recommended MIPs` section anticipates ([MPS-0007 §"Recommended MIPs"](../mps/mps-0007-node-event-visibility.md#recommended-mips)).

The design is independent of the events Compact-language proposal (the events CoIP, [MPS-0005: Events](../mps/mps-0005-events.md)) along the axis MPS-0007 sets out in its goals: node-side visibility ships without depending on Compact-language changes, and the events CoIP — once delivered — can assume node-side visibility as a building block rather than re-prove it. Re-running transactions against a separate copy of the ledger state ceases to be the only path to event observability; it becomes one consumer pattern among several.

## Specification

### Publication channel

Ledger-emitted events are surfaced through Substrate's existing `frame_system::Events` storage value. `pallet_midnight::Event` gains one new variant, appended to the existing enum:

```rust
#[pallet::event]
pub enum Event<T: Config> {
    // ... existing variants retained at their current indices ...
    LedgerEvent(LedgerEvent),
}
```

The pallet's `apply_transaction` host-fn caller iterates the per-transaction event vector returned from the ledger bridge and deposits one runtime event per ledger event via `Self::deposit_event(Event::LedgerEvent(ev))`. The standard `frame_executive` event lifecycle then applies: events are appended to `frame_system::Events`, the storage root commits them, and `initialize_block_impl` calls `reset_events()` to drop the value at the start of the next block (`substrate/frame/system/src/lib.rs::reset_events`, `substrate/frame/executive/src/lib.rs::initialize_block_impl`). Consumers read the resulting per-event records through `state_subscribeStorage([System.Events])`, `state_getStorage(System.Events, blockHash)`, or any client that decodes `frame_system::EventRecord` against the runtime metadata.

### Wire shape

`pallet_midnight::Event::LedgerEvent` carries a single struct, `LedgerEvent`, defined once in the ledger crate's common types module so the bridge and the pallet share it. The struct's outer SCALE shape is part of this specification:

```rust
/// Routing header for a ledger-emitted event. SCALE-mirrored from
/// `mn_ledger_local::events::EventSource`.
#[derive(Encode, Decode, DecodeWithMemTracking, TypeInfo, Clone, Eq, PartialEq, Debug)]
pub struct LedgerEventSource {
    pub transaction_hash: Hash,
    pub logical_segment: u16,
    pub physical_segment: u16,
}

/// One ledger event as observed at the apply-transaction boundary.
/// The `content_tagged_bytes` payload is the ledger crate's `Tagged`
/// serialisation of `EventDetails<D>` (tag `event-details[v9]`,
/// identical across `mn-ledger` v7 and v8). Consumers re-hydrate the
/// payload via `midnight-ledger`'s deserialiser of the matching ledger
/// version.
#[derive(Encode, Decode, DecodeWithMemTracking, TypeInfo, Clone, Eq, PartialEq, Debug)]
pub struct LedgerEvent {
    pub source: LedgerEventSource,
    pub content_tagged_bytes: Vec<u8>,
}
```

`LedgerEventSource` mirrors the ledger crate's `EventSource` triple verbatim: the 32-byte transaction hash plus two `u16` segment indices that identify the call slot within the transaction (logical segment for the call boundary, physical segment for the dispatch slot). `content_tagged_bytes` carries the ledger crate's own `Tagged` byte serialisation of the typed `EventDetails<D>` variant — a `ContractDeploy`, `ContractLog`, `ZswapInput`, `ZswapOutput`, `ParamChange`, dust-event, or any future variant — opaque to SCALE and decoded by the matching `midnight-ledger` version on the consumer side.

### Ordering

The pallet deposits events in the order the ledger bridge returns them, which is the order `state.apply` emits them during transaction execution. Within a block, the ordering across transactions follows extrinsic order. The ordering is part of the storage-root commitment: every honest validator computes the same `frame_system::Events` value for a given block, in the same order.

### Partial success

A `TransactionResult::Success(events)` produces events for the entire transaction. A `TransactionResult::PartialSuccess(segments, events)` produces events only for the guaranteed portion and any successful fallible segments — segments the ledger reports as failed contribute no events. A `TransactionResult::Failure(_)` produces no `LedgerEvent` records at all; the pallet's existing failure dispatch is unchanged. This matches the ledger crate's own event-vector contents at each result variant.

### Dry-run paths

The pallet's pre-dispatch validators (`do_validate_transaction`, `do_validate_guaranteed_execution`) do not commit state and do not deposit events. The host-fn surface for those entry points returns no event vector; dry-run runs of `state.apply` discard the events the same way the pre-implementation code did. Surfacing events from validators would publish records the network never agreed on.

### Cross-version stability

The `LedgerEvent` outer SCALE shape is stable across `mn-ledger` v7 and v8 today and is committed to remain stable across future ledger versions. The ledger crate's `EventDetails<D>` is tagged `event-details[v9]` in both versions, and the wrapper carries the tag inside `content_tagged_bytes` rather than reflecting it in the SCALE envelope. New `EventDetails` variants, additional fields on existing variants, or a ledger-major-version bump appear as different bytes inside `content_tagged_bytes` and require no change to the wrapper, the pallet event variant, or the runtime metadata. The wrapper would only change if the slot-identity triple itself were to gain a fourth dimension, or if the pallet were to abandon the tagged-byte form for a different payload encoding — and a future MIP would govern either.

### Retention

`frame_system::Events` is reset at every block's `initialize_block_impl`. On a default-pruned node, block-N events leave active state when block N falls out of the keep-window. Archive operators retain them via the same Substrate archive configuration that retains every other pallet event; this MIP introduces no new operator-facing flag. The wire shape is portable to a future pallet-controlled storage column with bounded retention should operational experience show archive-disk growth is unsustainable.

### Pallet ABI version transition

The behaviour ships behind a new `#[version(3)]` of `apply_transaction` and a `#[version(2)]` of `apply_system_transaction` on both `host_api/ledger_7.rs` and `host_api/ledger_8.rs`. The pre-existing `#[version(1)]` and `#[version(2)]` of `apply_transaction` stay in tree to support historical replay against WASM runtimes compiled before this MIP's activation. The runtime upgrade that introduces the new variant increments the runtime's `spec_version`; Substrate routes each WASM blob to the host-fn version it was compiled against. The pallet's `Event` enum gains one variant at the tail of the enum; the existing variants retain their declared positions and `#[codec(index)]` values.

### Contract-event namespacing

Contract-emitted events surface inside `EventDetails::ContractLog`, whose `(ContractAddress, EntryPointBuf)` tuple namespaces the event by `(emitting contract, entry-point label)`. The `ContractAddress` is the chain-controlled on-chain identity of the emitting contract; the `entry_point` label is contract-author-supplied. Two distinct contracts cannot collide on the same routing tuple. This MIP does not introduce a node-side `topic` type; the `ContractLog` namespace is the contract-author-facing surface and any future dedicated topic type is an upstream `midnight-ledger` change.

### Cross-references to companion sections

Two adjacent sections detail aspects of the wire shape that consumers commonly ask about. `## Finality Semantics and Consumer Pipeline` (immediately below) defines when in Substrate's block lifecycle a `LedgerEvent` becomes observable and how consumers compose `frame_system::Events` reads with GRANDPA finality. `## Event Schema and Compact Integration` records the boundary between this MIP's transport-level commitment (opaque tagged bytes) and a downstream concern: the schema-sharing surface that a Compact-side transcript verifier would need beyond byte-level transport. `## Performance Impact` quantifies the cost of depositing one `LedgerEvent` per ledger event onto the per-block weight envelope.

## Finality Semantics and Consumer Pipeline

`frame_system::Events` is per-block storage written at block-execution time, **before** finality is signalled. The events for block N are readable from `state_getStorage(System.Events, blockHashN)` the moment block N is imported on a connected full node, regardless of whether that block has been finalised by GRANDPA. This MIP does not alter that lifecycle: the deposit happens inside the pallet's call to `Self::deposit_event(Event::LedgerEvent(ev))` during transaction application, which precedes the storage-root computation and, in turn, the finality vote.

Midnight's downstream consumer model — the indexer, wallets, light-client DApps, cross-chain bridges — operates on **finalised** data. The reconciliation between per-block event publication and the finalised-only consumer model is a consumer-side composition of two existing Substrate primitives. The canonical flow is:

1. **Subscribe to finalised heads** through `chain_subscribeFinalizedHeads` (GRANDPA finality stream).
2. **For each finalised block hash**, fetch the events for that block via `state_getStorage(System.Events, blockHash)` — or, equivalently in `subxt`, follow the finalised-blocks stream and read events from the resulting `Block` handle.
3. **Decode each `EventRecord`** against the runtime metadata, filter for `pallet_midnight::Event::LedgerEvent(...)`, and re-hydrate the `content_tagged_bytes` payload against the matching `midnight-ledger` version.

This is the same shape as a standard Substrate `subxt` finalised-blocks event subscription, applied to the new `LedgerEvent` variant. Consumers that already read pallet events on finalised heads — the existing indexer is one such consumer — keep their architecture; the change is that `pallet_midnight::Event::LedgerEvent` now carries content the indexer previously had to recover by re-running `state.apply` against its own copy of the ledger crate.

Consumers that want to observe events on the best chain rather than the finalised chain can subscribe to `state_subscribeStorage([System.Events])` directly and accept the reorg risk that comes with reading pre-finality state. That is the existing Substrate trade-off for any state read on non-finalised blocks; the MIP does not invent a new path and does not change it. High-value consumers that require both finality and a streaming surface compose `chain_subscribeFinalizedHeads` with per-block `state_getStorage` reads, as described above.

**Why no new node-side stream is added.** A typed RPC method that filtered events to the finalised chain at the node — for example, a hypothetical `midnight_subscribeFinalizedBlockEvents` returning `Vec<LedgerEvent>` per finalised block — would be option (d) in the Rationale below. It is deferred under `## Path to Active` "Implementation Plan": the wrapper can ship as a follow-on on top of the per-block surface this MIP commits to, without further MIP work. Reframing finality at the node rather than at the consumer would also bind the MIP to a particular consumer policy (finalised-only) when the per-block surface already supports both reorg-tolerant and finality-tracking consumers.

**Cross-reference.** `## Backwards Compatibility Assessment` adds a "Consumer assumption layer — finality" sub-paragraph that records what the existing indexer's operational model relies on and how new consumers SHOULD compose the two primitives. The contract this section defines — events surface at execution time; consumers filter by finality — is the consumer-facing piece of that layer.

## Event Schema and Compact Integration

The wire shape `## Specification` commits to is sufficient for **transport**: a consumer that decodes `pallet_midnight::Event::LedgerEvent` recovers the routing header (`LedgerEventSource`) and the opaque `content_tagged_bytes` payload. To re-hydrate the payload into typed `EventDetails<D>` variants — `ContractDeploy`, `ContractLog`, `ZswapInput`, `ZswapOutput`, `ParamChange`, dust events, future variants — the consumer must hold the `midnight-ledger` crate version that produced the bytes. That is the present-day surface and the one this MIP standardises for node-side observability.

Transport is not sufficient for a separate consumer class: a **Compact-side transcript verifier**. Verifying that a contract execution emitted the events its transcript declares requires evaluating those events against a schema both Compact and the ledger crate agree on. The current Compact ↔ Ledger surface is byte-level only — the ledger crate tagged-serialises typed `EventDetails<D>` values at the boundary, with no shared types crate, no published registry, and no runtime hook the Compact verifier can call. The byte-level boundary is what this MIP exposes; the schema-sharing surface a verifier would compose against it is distinct.

**Routing of the schema-sharing problem.** This MIP records the dependency and points at the work that resolves it; it does not design that work. A sibling MIP — referenced from this MIP's front matter as `Requires: MIP-???? — Compact ↔ Ledger event-schema sharing protocol [pre-numbering]` — is the right venue. That sibling MIP would specify the canonical schema's home (registry or shared-types crate), the acquisition mechanism (compile-time vendoring, host-fn fetch, or in-chain storage), the Compact ↔ Ledger versioning rules, and the security implications of Compact-side reasoning about event content. Those choices touch the Compact compiler, ledger crate, and possibly the runtime ABI; folding them in here would dilute the load-bearing per-block event surface.

**What this MIP does and does not commit to.** The opaque tagged-byte payload is forward-compatible: new `EventDetails` variants or field additions appear as different bytes inside `content_tagged_bytes` and require no changes to the wrapper, the pallet event variant, or the runtime metadata (see `## Backwards Compatibility Assessment` Layer 2), making transport stable across ledger evolution. It does **not** give Compact a mechanism to validate transcript-declared events without the matching `midnight-ledger` version. Consumers needing a Compact-side validation hook are outside this MIP's scope until the sibling MIP lands. This MIP can transition to `Active` independent of the sibling for its committed consumer set (Substrate-tooling consumers reading `frame_system::Events`); editors may gate `Active` on the sibling MIP at numbering time via the `Requires:` field.

## Rationale

Four publication mechanisms were considered; each surfaces the same events but differs in storage location, retrieval, and operational profile.

- **(a) `frame_system::Events` via a new pallet event variant.** The mechanism specified above. Events ride the standard FRAME event channel, decoded by any Substrate consumer that already reads pallet events.
- **(b) Storage proofs only — events derivable from canonical state.** No new persistence; consumers reconstruct events by replaying transactions against state proven via `state_getReadProof`. The present-day indexer model promoted to a documented pattern.
- **(c) Pallet-controlled storage column with bounded retention.** A `StorageMap` keyed by block number holding events, pruned on a configurable window inside the pallet.
- **(d) Typed RPC subscription wrapper.** A `midnight_subscribeBlockEvents` method returning `Vec<LedgerEvent>` per finalised block, layered on (a)–(c) for non-Substrate-native consumers.

The trade-offs:

| Concern | (a) FRAME events | (b) Replay-only | (c) Pallet storage | (d) Typed RPC |
|---|---|---|---|---|
| Network bandwidth | None — events read on demand | None | None | None — events read on demand |
| Light-client friendliness | High — `frame_system::Events` is storage-root-committed; storage proofs available | None — light client cannot replay without full state | High — same storage-proof model | Inherits from underlying |
| Indexer ingest path | `state_subscribeStorage([System.Events])`; standard subxt event decoding | Continue current replay model | Custom storage subscription | Custom RPC subscription |
| Implementation work in midnight-node | One new pallet variant; bridge plumbing; one new host-fn version | None | A new storage column, retention policy, pruning logic, and reads | One RPC method, layered above |
| Ledger-version churn surface | Wrapper-stable across versions; payload is opaque tagged bytes | Indexer rebuild on every event-shape change | Same as wrapper — once defined | Inherits from underlying |
| Pallet ABI surface | One appended `Event` variant | None | One appended `Event` variant plus storage items | Inherits from underlying |

(Per-node disk cost is in `## Backwards Compatibility Assessment` "Archive-node disk growth"; retention is in `## Performance Impact` "Mitigations and operator controls"; both rows are omitted from the table to avoid duplication.)

The mechanism specified is **(a)**. The decisive properties are light-client friendliness and tooling reach: `frame_system::Events` is storage-trie-committed, so light clients verify any event via `state_getReadProof` without full ledger state or a trusted indexer, and the same record decodes against runtime metadata in `polkadot.js`, `subxt`, and every Substrate-tooling integration — the explicit UC5 promise in MPS-0007.

Mechanism **(b)** preserves the status quo: events remain replay-derivable only, requiring an operationally-available version-pinned indexer and leaving UC1 / UC2 / UC3 / UC5 unaddressed. It is the implicit baseline. Mechanism **(c)** adds a pallet-owned retention knob at the cost of a new storage column and a custom subscription surface consumer libraries do not already understand; the wire shape this MIP commits to is portable to (c) without re-spec if archive growth under (a) proves unsustainable. Mechanism **(d)** is a convenience wrapper, not an alternative: `state_subscribeStorage([System.Events])` already provides real-time delivery via any Substrate client, so (d) is deferred to a follow-on without altering the wire shape.

Within mechanism (a), three sub-decisions warrant explicit rationale:

- **Opaque tagged bytes for `EventDetails<D>` rather than a SCALE-mirrored enum.** `EventDetails<D>` is `#[non_exhaustive]`; mirroring its variants in SCALE creates a maintenance ratchet and risks silent drift. The ledger's own `Tagged` byte form is forward-compatible — consumers decode against the ledger version they ship with, and the runtime ABI does not move when the taxonomy does.
- **One pallet-event variant per ledger event rather than one variant per block carrying a `Vec`.** `subxt` and equivalent clients filter events by `(pallet_index, variant_index)` *before* SCALE-decoding. A block-level `Vec<LedgerEvent>` defeats that filtering granularity. One variant per event also aligns with existing per-occurrence variants (`ContractCall`, `ContractDeploy`, `ContractMaintain`, …).
- **A new `TransactionAppliedStateRootV2` sibling struct rather than extending the existing struct in place.** SCALE does not ignore trailing bytes on decode; extending would break replay against WASM runtimes compiled against the old encoding. A sibling struct returned by `#[version(3)]` keeps historical replay byte-stable, following the precedent in `ledger/src/host_api/ledger_7.rs::apply_transaction` v1 vs v2.

The Compact ↔ Ledger event-schema sharing surface — what a Compact-side transcript verifier would need beyond byte-level transport — is cross-referenced (see `## Event Schema and Compact Integration`) rather than designed here, routing that work to the sibling MIP named in the frontmatter `Requires:` field.

## Performance Impact

Depositing one runtime event per ledger event puts a small additional cost on the critical path of block authoring and verification. Numbers below are envelopes drawn from existing sources, not fresh benchmarks; every quantitative claim is marked `[estimate]`.

### Per-event weight envelope

`frame_system::deposit_event` performs a small, near-constant amount of work per call: an allocation, a SCALE-encode of the event payload, and a push onto the per-block `Events` vector that the storage root later commits. Substrate publishes `frame_system::WeightInfo` for `deposit_event`-equivalent operations at the polkadot-sdk pin (`polkadot-stable2603`) the midnight-node depends on. CPU cost is dominated by the SCALE-encode of the payload (linear in encoded byte length); the constant overhead per deposit is hundreds of nanoseconds on a typical validator [estimate]. The `Events` value is reset at every block, so the write touches transient storage only.

The wire shape bounds payload length deterministically: `LedgerEventSource` is `32 + 2 + 2 = 36` bytes; `content_tagged_bytes` is a length-prefixed `Vec<u8>` bounded by the ledger crate's serialiser plus the per-transaction byte limit (`transaction_byte_limit = 1 MiB`).

### Block-budget headroom analysis

The polkadot-sdk runtime exposes a `BlockWeights::max_block` budget bounding the total weight of work a validator will accept on a block. Translating the per-event envelope against that budget against the source kb-research § 1 event-volume figures:

- **Typical envelope** (~45 KiB of events per block; dozens to low hundreds of events): SCALE-encode and deposit cost on the order of low microseconds aggregate per block [estimate]; a small fraction of one per cent of `BlockWeights::max_block` [estimate].
- **Heavy envelope** (50–200 KiB of events per block, sustained; up to mid-thousands of events in a deploy-heavy or contract-log-heavy mix): tens to low-hundreds of microseconds aggregate per block [estimate]; at most a few per cent of `BlockWeights::max_block` [estimate].

Acceptance criterion M5 in `## Path to Active` formalises the upper bounds against which `midnight-node#1487` is measured (≤ 5 % typical, ≤ 15 % deploy-heavy block-time increase). The envelopes here are consistent with M5; the implementation PR is the authoritative source of measured numbers once its benchmark suite is integrated.

### Mitigations and operator controls

Three controls absorb the event-deposit cost without further protocol change. (i) **Per-event payload size bound:** the existing per-transaction byte limit already caps producible bytes; no new bound. (ii) **Archive opt-in pruning:** default-pruned nodes drop block-N events at the keep-window boundary, leaving only archive operators with the long-tail cost. (iii) **Fallback to mechanism (c):** if the envelope proves unsustainable, the wire shape is portable to a pallet-controlled storage column with bounded retention — only the persistence layer moves.

### What is and is not measured

Numbers above are envelopes, not measurements. Event-byte counts and archive-disk-growth figures (`~57 GiB/year typical`, `~263 GiB/year sustained heavy`) come from the source kb-research § 1 envelope analysis. The `deposit_event` constant overhead and SCALE-encode rate come from the polkadot-sdk benchmark documentation at `polkadot-stable2603`. No fresh `deposit_event` micro-benchmark on the midnight-node validator profile, end-to-end TPS regression, or sustained-heavy archive-disk-growth measurement is produced inside this MIP. That evidence is supplied by [`midnight-node` PR #1487](https://github.com/midnightntwrk/midnight-node/pull/1487)'s benchmark suite (criteria M5 and M8) at numbering time; `[estimate]` markers are placeholders for that evidence and may be replaced by measured numbers before this MIP advances beyond `Accepted`.

## Path to Active

### Acceptance Criteria

The MIP becomes `Active` when the following objective milestones hold against a deployed Midnight network upgrade. Each criterion is verified by the test layer named in parentheses.

1. **M1 — Events observable per finalised block.** A finalised block's `frame_system::Events` storage contains one `pallet_midnight::Event::LedgerEvent(_)` record per event the ledger crate produced during transaction application, in deposit order, retrievable via `state_subscribeStorage([System.Events])` and `state_getStorage(System.Events, blockHash)` (pallet runtime test; integration test).
2. **M2 — System-transaction symmetry.** Events produced by system-transaction application (parameter changes, dust-initial-utxo provisioning) surface through the same `Event::LedgerEvent` channel (pallet runtime test).
3. **M3 — Cross-version wire-shape stability.** `LedgerEvent`'s SCALE encoding is byte-identical across builds resolving the ledger crate to either `mn-ledger` v7 or v8 for equivalent event fixtures, and `LedgerEvent::decode()` succeeds on output produced by either build (unit test in `ledger/src/common/types.rs::tests`).
4. **M4 — Namespaced contract events.** A contract emitting through `EventDetails::ContractLog` surfaces an event whose decoded `(ContractAddress, EntryPointBuf)` tuple matches the emitting contract's identity and the chosen entry-point label; two contracts emitting events with identical entry-point labels produce distinguishable records (pallet runtime test; unit test).
5. **M5 — Block-time impact bounded.** End-to-end `apply_transaction` wall time on a representative typical block (≈ 50 shielded transfers) increases by ≤ 5 % relative to the pre-implementation baseline; on a deploy-heavy block the increase is ≤ 15 % (benchmark).
6. **M6 — No regression on existing event surface.** The eight pre-existing `pallet_midnight::Event` variants encode byte-identically on equivalent input; existing pallet-RPC outputs are unchanged; new `LedgerEvent` records are additions, not replacements (pallet runtime test; unit test; metadata-shape check).
7. **M7 — Replay correctness.** Re-running block N against a snapshot at block N − 1 yields the same `LedgerEvent` set, modulo expected digest differences; historical blocks authored under the pre-events `apply_transaction` host-fn version replay cleanly under a node carrying the new version, with no `LedgerEvent` records for pre-activation blocks (integration test).
8. **M8 — No `BlockLength` displacement.** Event volume does not appear in `frame_system::BlockSize`; block-fill rejections under saturation report synthetic-cost saturation rather than `BlockLength` exhaustion (pallet runtime test; integration test).

A typed RPC subscription wrapper for non-Substrate consumers (`midnight_subscribeBlockEvents`) is recorded as future work in the Implementation Plan below; it is not an acceptance criterion for this MIP.

### Implementation Plan

The implementation is prepared in [`midnight-node` PR #1487](https://github.com/midnightntwrk/midnight-node/pull/1487) and is paused pending this MIP's transition to `Accepted` — the standard MIP-0001 sequencing, in which acceptance of the proposal precedes merge of the implementation. The path to `Active` is therefore: editor numbering of the present draft → community-commentary and editor vote → status transition to `Accepted` (which unblocks PR #1487 for merge) → PR #1487 is merged → status transition to `Implemented` → status transition to `Active` when the runtime upgrade that carries the new `pallet_midnight::Event::LedgerEvent` variant and the `apply_transaction` `#[version(3)]` host-fn enters a deployed Midnight network. A typed `midnight_subscribeBlockEvents` RPC wrapper is held as a follow-on that can ship in a subsequent runtime upgrade — or in a node-only release — without further MIP work; the wire shape this MIP specifies is the layered protocol that wrapper would surface.

## Backwards Compatibility Assessment

The change does **not** require a hard fork. It ships in a normal Substrate runtime upgrade. Compatibility is best understood as three distinct layers, each with its own stability commitment.

**Layer 1 — Pallet ABI stability (existing event variants).** None of the eight pre-existing `pallet_midnight::Event` variants change name, position, payload type, or `#[codec(index)]`. `LedgerEvent` is appended at the tail of the enum. Any consumer whose code already decodes `frame_system::Events` against the pallet's metadata — `polkadot.js`, `subxt`, the indexer's pallet-event reader, custom Substrate-tooling integrations — keeps working unchanged on every block produced before the activation point and on every block produced after the activation point. New consumers opt into the `LedgerEvent` variant by recompiling against the new pallet metadata. Confirmed by acceptance criterion M6.

**Layer 2 — Wire-shape stability across ledger-crate versions (the wrapper).** The `LedgerEvent { source: LedgerEventSource, content_tagged_bytes: Vec<u8> }` struct has a fixed outer SCALE shape: the routing header is 32 + 2 + 2 bytes; the payload is a length-prefixed byte vector. The wrapper is stable across `mn-ledger` v7 and v8 — both ledger versions tag their `EventDetails<D>` as `event-details[v9]` and the wrapper carries the tag inside `content_tagged_bytes` rather than reflecting it into the SCALE envelope. The wrapper continues to be stable when `mn-ledger` adds a new `EventDetails` variant (the variant appears as different bytes inside `content_tagged_bytes`, leaving the wrapper untouched), when fields are added to an existing variant (same), or when `mn-ledger` cuts a new major version (the tag string inside the payload changes; the wrapper does not). The wrapper would change only if the slot-identity triple — `(transaction_hash, logical_segment, physical_segment)` — were to gain a fourth dimension, or if a future MIP deliberately chose to abandon the tagged-byte payload encoding. Either is governed by a future MIP, not by ledger-crate evolution. Confirmed by acceptance criterion M3.

**Layer 3 — Pallet host-fn ABI stability (`apply_transaction` versioning).** The new behaviour ships behind `#[version(3)]` of `apply_transaction` and `#[version(2)]` of `apply_system_transaction` on both `host_api/ledger_7.rs` and `host_api/ledger_8.rs`. The pre-existing `#[version(1)]` and `#[version(2)]` of `apply_transaction` (and the pre-existing `#[version(1)]` of `apply_system_transaction`) remain in tree. Substrate's runtime-upgrade machinery routes each WASM blob to the host-fn version it was compiled against, so historical block replay against pre-activation WASM continues to work without any new branch in the pallet itself. This pattern is the same one already established by the v1 → v2 transition of `apply_transaction` documented in the host-API source. Any future version bump can follow the same shape — append a new `#[version(N)]` rather than mutate an existing one. Confirmed by acceptance criterion M7.

**Layer (d) — Consumer assumption layer (finality).** The first three layers describe stability commitments inside the node. A fourth concern, raised by the operational shape of the existing Midnight consumer set, applies outside the node: what does a consumer assume about *when* a `LedgerEvent` becomes safe to act on? The pre-MIP indexer (and any equivalent re-execution consumer) re-runs `state.apply` on **finalised** blocks only and therefore implicitly observes finalised events. That consumer keeps working unchanged after activation — its finality model is unaffected, and the events it produces by re-execution match the events `pallet_midnight::Event::LedgerEvent` deposits on the same block (criterion M7). New consumers that read `frame_system::Events` directly receive events at execution time, before finality, and SHOULD filter by finality if their use case requires finalised-only data. The canonical composition — subscribe to `chain_subscribeFinalizedHeads`, then read events by the finalised hash via `state_getStorage(System.Events, blockHash)` — is documented in `## Finality Semantics and Consumer Pipeline`. The implication for backwards compatibility is conservative: existing consumers' assumptions hold; new consumers compose two existing Substrate primitives; the node itself adds no finality-filtered stream and incurs no new finality-coupled obligation. No acceptance criterion is added; the section is a consumer-facing contract, not a node-side gate.

**Hard-fork question.** No. The change ships as a runtime upgrade. The runtime's `spec_version` is incremented to flip the pallet's call site from the pre-events host-fn version to `#[version(3)]`. Validators that have not yet upgraded continue producing blocks correctly under their compiled-in host-fn version; their blocks contain no `LedgerEvent` records and replay cleanly. Validators that have upgraded produce blocks carrying `LedgerEvent` records on top of the existing event surface. Block replay across the upgrade boundary works because the pre-events WASM blobs route to their compiled-in `#[version]` (M7).

**Archive-node disk growth.** The one *operational* compatibility concern is archive-node disk consumption. At a typical event envelope of approximately 45 KiB per block and a heavy envelope of 50–200 KiB per block, six-second blocks produce roughly 6.5 MiB per hour on a typical mix and roughly 30 MiB per hour on a sustained-heavy mix — on the order of 57 GiB per year typical and 263 GiB per year sustained heavy. These figures are modest in absolute terms (a one-terabyte archive volume absorbs a decade of typical load), but archive operators should account for them when sizing storage. Default-pruned nodes are unaffected: `frame_system::Events` is reset at every block, and the keep-window dictates when a block's events leave active state. Archive operators opt into the cost via the same Substrate archive configuration that retains every other pallet event; this MIP introduces no new flag. If sustained operational experience indicates the envelope is unsustainable, mechanism (c) from the Rationale — a pallet-controlled storage column with bounded retention — is the documented fallback, and the wire shape this MIP commits to is portable to it without consumer-side churn.

## Security Considerations

Four concerns arise from the change. Each is addressed by mechanism already in place; none is novel to this MIP. A short note at the end of the section records claims the MIP deliberately does **not** make.

**Archive growth as a resource-pricing concern.** A pathological transaction — for example, a large `ContractDeploy` near the per-transaction byte ceiling, or a contract emitting high-volume `ContractLog` events — inflates archive-node disk usage at the heavy end of the envelope documented in Backwards Compatibility. The mitigation is the existing resource-pricing model: the per-transaction byte limit (`transaction_byte_limit = 1 MiB`) and the synthetic block-usage budget (`block_usage = 200_000`) already bound per-block event volume. This MIP does not change those bounds and does not let an adversary bypass any existing limit. The cost an adversary already pays at `state.apply` time is the same cost; the difference is that the produced events are now persisted (transiently on default-pruned nodes, optionally permanently on archive nodes) rather than discarded at the bridge. Default-pruned nodes have no new exposure. Archive operators inherit the cost via the same retention configuration they already use for every other pallet event; the option-(c) escape hatch (pallet-controlled storage column with bounded retention) is documented as the remediation path if sustained operational experience surfaces an unmanageable envelope.

**Light-client trust assumptions.** A light client that subscribes to `state_subscribeStorage([System.Events])` over a single full-node connection trusts that node not to omit or fabricate event records. The Substrate trust model already addresses this: `frame_system::Events` is part of the storage trie, and the trie's root is committed in the block header. Any subscriber that requires more than the connecting node's word can request a Merkle proof via `state_getReadProof` for the `System.Events` key against a specific block hash, and verify the events against the header-committed storage root. This is the same trust model already in place for every other pallet event; the MIP introduces no new trust assumption. A cross-chain bridge or any high-value consumer should consume events through storage proofs rather than bare subscription — the same guidance that applies to other Substrate event channels.

**Contract-event topic schema (namespace collisions).** A consumer reading `ContractLog` events needs to know that two distinct contracts cannot collide on the same routing tuple. The mitigation is the namespace design itself: the `ContractLog` variant's routing identity is `(ContractAddress, EntryPointBuf)`, not the entry-point label alone. `ContractAddress` is the chain-controlled on-chain identity of the emitting contract; the chain assigns it at deploy time. Two contracts with identical entry-point labels are distinguished by their addresses. Pallet events live under a different `(pallet_index, variant_index)` namespace at the outer SCALE envelope of `frame_system::Events`. A contract event is always wrapped in `pallet_midnight::Event::LedgerEvent(...)`; a pallet event is never wrapped in a `ContractLog`. There is no cross-namespace collision the consumer needs to disambiguate. Confirmed by acceptance criterion M4.

**Replay correctness as a security property.** If an attacker could produce blocks whose surfaced events differ from what `state.apply` actually emits, the event channel would become a new trust attack surface for any consumer that conditions behaviour on event content. The mitigation is that events are derived from the canonical state. `state.apply(verified_tx, ctx)` is deterministic; every honest validator computes the same per-transaction `Vec<Event<D>>`. The pallet deposits exactly what `state.apply` returns, in the order returned. The deposited records are part of `frame_system::Events`, and the storage root commits them along with the rest of the state. A would-be attacker forging a different event set would have to author a non-canonical block, which other validators reject as having a wrong storage root. The event channel is therefore as trustworthy as the consensus mechanism itself; it adds no new trust assumption. Confirmed by acceptance criterion M7.

**Claims this MIP does not make.** Three over-claims are easy to drift into in this section and would be incorrect.

- This MIP **does not** improve privacy. It surfaces events that the ledger crate already produces; it does not encrypt them, anonymise them, or otherwise reduce information leak. Any privacy concern that applied at the ledger layer remains a privacy concern at the node layer, surfaced or not. Privacy-preserving event patterns are the subject of [MPS-0005](../mps/mps-0005-events.md) and would belong in a follow-on MIP that addresses them directly.
- This MIP **does not** fix any past discrepancy between indexer-replayed events and ledger-emitted events. If such a discrepancy existed, it would be a bug in the ledger crate or in the indexer, not an artefact of node-side visibility.
- This MIP **does not** improve bandwidth efficiency. Events were not previously transmitted; they were discarded. They are now persisted on the producing node and read on demand by consumers. The aggregate network cost is roughly flat across the publication mechanisms enumerated in the Rationale; the difference is observability, not bandwidth.

## Implementation

The implementation is prepared in [`midnight-node` PR #1487](https://github.com/midnightntwrk/midnight-node/pull/1487), addressing [`midnight-node` issue #1474](https://github.com/midnightntwrk/midnight-node/issues/1474). It is paused pending this MIP's transition to `Accepted` (see §"Path to Active"). Components modified:

- **`ledger/src/versions/common/api/ledger.rs`** — `Ledger::apply_verified_transaction` and `Ledger::apply_system_tx` return the event vector alongside existing outputs. The two call sites in `ledger/src/versions/common/mod.rs` consume the new shape; pre-implementation discard sites are removed.
- **`ledger/src/common/types.rs`** — defines `LedgerEvent` and `LedgerEventSource` (per §"Specification") and a `TransactionAppliedStateRootV2` sibling struct carrying `events: Vec<LedgerEvent>`. The pre-existing `TransactionAppliedStateRoot` is retained for replay through pre-events host-fn versions.
- **`ledger/src/versions/common/mod.rs`** — `Bridge::apply_transaction` and `Bridge::apply_system_transaction` populate `events` by tagged-serialising each `Event<D>` from the ledger API.
- **`ledger/src/host_api/ledger_7.rs` and `ledger/src/host_api/ledger_8.rs`** — `#[version(3)]` of `apply_transaction` and `#[version(2)]` of `apply_system_transaction` return `TransactionAppliedStateRootV2`. Prior versions remain in tree for historical replay.
- **`pallets/midnight/src/lib.rs`** — the pallet's `Event` enum gains a `LedgerEvent(LedgerEvent)` variant appended at the tail; `send_mn_transaction` iterates `result.events` and deposits one runtime event per `LedgerEvent`.
- **`pallets/midnight-system/src/lib.rs`** — symmetric change for system transactions; system-tx events deposit on the same `pallet_midnight::Event::LedgerEvent` channel.

No new external dependencies: the change reuses Substrate's existing FRAME event machinery, the established host-fn versioning pattern (v1 → v2 of `apply_transaction`), and the existing pallet event deposit path. Activation requires only a standard runtime upgrade incrementing `spec_version`.

Measured performance evidence — `deposit_event` micro-benchmarks, block-time deltas, archive-disk-growth — comes from PR #1487's benchmark suite (criteria M5 and M8) and backs the envelopes in `## Performance Impact`.

## Testing

Verification spans four layers of midnight-node's existing test harness: unit tests inside `ledger/src/common/types.rs::tests` and the pallet's inline `#[cfg(test)]` modules for type round-trips and code-path logic; pallet-runtime tests against the substrate test runtime (`pallets/midnight/src/mock.rs` + `tests.rs`) that exercise the pallet end-to-end with the new `Event::LedgerEvent` variant; integration tests in `node/tests/` and chain-spec-based smoke tests that exercise full block production, RPC, and replay; and benchmarks in `pallets/midnight/src/benchmarking.rs` that measure block-time impact.

Each acceptance criterion in `## Path to Active` is covered by at least one test case in at least one layer:

| Criterion | Test layers |
|---|---|
| M1 — Events observable per finalised block | Pallet runtime, Integration |
| M2 — System-transaction symmetry | Pallet runtime |
| M3 — Cross-version wire-shape stability | Unit |
| M4 — Namespaced contract events | Pallet runtime, Unit |
| M5 — Block-time impact bounded | Benchmark, Integration smoke |
| M6 — No regression on existing event surface | Pallet runtime, Unit, Compile-time metadata check |
| M7 — Replay correctness | Integration |
| M8 — No `BlockLength` displacement | Pallet runtime, Integration |

In addition, negative-path tests cover the partial-success and failure semantics specified in §"Specification": a `TransactionResult::Failure` produces zero `LedgerEvent` records; a `TransactionResult::PartialSuccess` produces records only for the guaranteed portion and any successful fallible segments; and the pallet's pre-dispatch validators do not deposit events. The full test enumeration, layer-by-layer assertions, fixtures, and pass conditions live alongside the implementation in [`midnight-node` PR #1487](https://github.com/midnightntwrk/midnight-node/pull/1487).

## References

### Midnight Improvement Proposals corpus

- [MPS-0007: Node-Side Visibility of Ledger Events](../mps/mps-0007-node-event-visibility.md) — the upstream Problem Statement this MIP responds to.
- [MIP-1: MIP Process](mip-0001-mip-process.md) — process, lifecycle, and section conventions.
- [MPS-0005: Events](../mps/mps-0005-events.md) — the Compact-language event-emission proposal. Complementary to but independent of this MIP per MPS-0007's "Be independent of the events CoIP" goal.

### Source issue and implementation PR

- midnight-node issue [#1474 — Node should expose events emitted by the Ledger](https://github.com/midnightntwrk/midnight-node/issues/1474) — the engineering issue that scoped the work.
- midnight-node PR [#1487 — feat: expose ledger events via frame_system::Events](https://github.com/midnightntwrk/midnight-node/pull/1487) — the implementation PR. `[pre-merge — re-pin at editor numbering time]`

### Implementation reference points

Line-anchored citations below point at the implementation-branch HEAD at the time this draft was authored (`58a709c8` on `feat/1474-expose-ledger-events`). They are marked for re-pinning to the merge commit of [#1487](https://github.com/midnightntwrk/midnight-node/pull/1487) once it merges. `[pre-merge — re-pin at editor numbering time]`

- The new `LedgerEvent` and `LedgerEventSource` SCALE types in [`ledger/src/common/types.rs`](https://github.com/midnightntwrk/midnight-node/blob/58a709c85c913f98b8df80f271b6c35f57baa0d2/ledger/src/common/types.rs) — the wire shape this MIP commits to.
- The bridge plumbing in [`ledger/src/versions/common/mod.rs`](https://github.com/midnightntwrk/midnight-node/blob/58a709c85c913f98b8df80f271b6c35f57baa0d2/ledger/src/versions/common/mod.rs) — where `Bridge::apply_transaction` and `Bridge::apply_system_transaction` populate the `events` field.
- The host-API versioning in [`ledger/src/host_api/ledger_7.rs`](https://github.com/midnightntwrk/midnight-node/blob/58a709c85c913f98b8df80f271b6c35f57baa0d2/ledger/src/host_api/ledger_7.rs) and [`ledger/src/host_api/ledger_8.rs`](https://github.com/midnightntwrk/midnight-node/blob/58a709c85c913f98b8df80f271b6c35f57baa0d2/ledger/src/host_api/ledger_8.rs) — `#[version(3)]` of `apply_transaction` and `#[version(2)]` of `apply_system_transaction`.
- The new pallet event variant and deposit loop in [`pallets/midnight/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/58a709c85c913f98b8df80f271b6c35f57baa0d2/pallets/midnight/src/lib.rs) — where `pallet_midnight::Event::LedgerEvent(LedgerEvent)` is deposited from `send_mn_transaction`.

### Pre-implementation discard sites (mirror MPS-0007 references)

These citations document the prior state — the discard points this MIP removes. Pinned at `b49ba64` matching MPS-0007's references.

- [`ledger/src/common/types.rs` L46 @ `b49ba64`](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/ledger/src/common/types.rs#L46) — `TransactionAppliedStateRoot` without an `events` field.
- [`ledger/src/versions/common/api/ledger.rs` L142–L158 @ `b49ba64`](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/ledger/src/versions/common/api/ledger.rs#L142-L158) — the ledger-bridge discard site: events bound to `_` in `apply_verified_transaction`.
- [`ledger/src/versions/common/mod.rs` L414 @ `b49ba64`](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/ledger/src/versions/common/mod.rs#L414) — `TransactionAppliedStateRoot` construction site: events not forwarded.
- [`pallets/midnight/src/lib.rs` L241 @ `b49ba64`](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/pallets/midnight/src/lib.rs#L241) — the pallet's pre-existing eight `Event` variants, none of which carry ledger-event payloads.

### Substrate / FRAME source

Pinned to `polkadot-stable2603`, the polkadot-sdk tag the midnight-node depends on.

- Substrate `frame_system::Events` storage declaration: [`substrate/frame/system/src/lib.rs` @ `polkadot-stable2603`](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/frame/system/src/lib.rs) — the `StorageValue<_, Vec<EventRecord<...>>, ValueQuery>` and the per-block `reset_events()` reset path called by `frame_executive::Executive::initialize_block_impl`. Also the source of the per-block-pre-finality lifecycle documented in `## Finality Semantics and Consumer Pipeline` and of the `frame_system::WeightInfo` weights cited in `## Performance Impact`.
- Substrate `frame_executive::Executive::initialize_block_impl`: [`substrate/frame/executive/src/lib.rs` @ `polkadot-stable2603`](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/frame/executive/src/lib.rs) — the call site that invokes `System::reset_events()` at the start of each block.
- Substrate `frame_system::limits::BlockWeights` (configuration of `max_block` and per-class limits): [`substrate/frame/system/src/limits.rs` @ `polkadot-stable2603`](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/frame/system/src/limits.rs) — the block-weight budget referenced by `## Performance Impact` block-budget headroom analysis.
- GRANDPA finality RPC documentation (`chain_subscribeFinalizedHeads`, `chain_getFinalizedHead`): [polkadot-sdk Substrate JSON-RPC reference](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/client/rpc-api/src/chain/mod.rs) — the finality subscription primitive referenced by `## Finality Semantics and Consumer Pipeline`.

### Consumer-side tooling

- [subxt event filtering docs](https://docs.rs/subxt/latest/subxt/events/index.html) — pattern reference for consumers decoding `pallet_midnight::Event::LedgerEvent(_)` from `frame_system::Events`.
- [subxt finalised-blocks streaming API](https://docs.rs/subxt/latest/subxt/blocks/index.html) — the consumer-side composition that `## Finality Semantics and Consumer Pipeline` describes (finalised heads → events by hash → decode `pallet_midnight::Event::LedgerEvent`).

## Acknowledgements

Thanks to Giles Cope (author of MPS-0007: Node-Side Visibility of Ledger Events) for framing the upstream Problem Statement that this MIP responds to, and to Oscar Bailey for early stakeholder feedback on the publication-mechanism trade-offs that informed the choice of `frame_system::Events` (mechanism (a) in the Rationale). Thanks to Dawid Zajkowski (@dzajkowski) for the first-cycle PR review whose feedback prompted the `## Event Schema and Compact Integration`, `## Finality Semantics and Consumer Pipeline`, and `## Performance Impact` sections that this draft now carries. Additional acknowledgements — further reviewers on the present PR, MIP editors who shape the draft during numbering, and community commenters during the Proposed period — will be added during the PR review cycle.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
