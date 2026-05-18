---
MIP: '?'
Title: Node-Side Surfacing of Ledger Events via frame_system::Events
Authors:
  - Mike Clay (m2ux)
Status: Draft
Category: Core
Created: 2026-05-18
Requires:
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

## Rationale

Four publication mechanisms were considered. Each surfaces the same events; they differ in storage location, retrieval pattern, and operational profile.

- **(a) `frame_system::Events` via a new pallet event variant.** The mechanism specified above. Events ride the standard FRAME event channel, decoded by any Substrate consumer that already reads pallet events.
- **(b) Storage proofs only — events derivable from canonical state.** Add no new persistence; consumers reconstruct events by replaying transactions against state they prove out via `state_getReadProof`. This is essentially the present-day indexer model promoted to a documented pattern.
- **(c) Pallet-controlled storage column with bounded retention.** A dedicated `StorageMap` keyed by block number (or block hash) holding events, pruned on a configurable window inside the pallet rather than at the Substrate node-pruning layer.
- **(d) Typed RPC subscription wrapper.** A `midnight_subscribeBlockEvents` RPC method returning `Vec<LedgerEvent>` per finalised block, layered on top of any of (a)–(c) as a convenience for non-Substrate-native consumers.

The trade-offs:

| Concern | (a) FRAME events | (b) Replay-only | (c) Pallet storage | (d) Typed RPC |
|---|---|---|---|---|
| Per-node disk cost | Low (default-pruned); high on archive nodes | None — no new persistence | Bounded by retention window | Same as the underlying mechanism |
| Network bandwidth | None — events read on demand | None | None | None — events read on demand |
| Light-client friendliness | High — `frame_system::Events` is storage-root-committed; storage proofs available | None — light client cannot replay without full state | High — same storage-proof model | Inherits from underlying |
| Indexer ingest path | `state_subscribeStorage([System.Events])`; standard subxt event decoding | Continue current replay model | Custom storage subscription | Custom RPC subscription |
| Implementation work in midnight-node | One new pallet variant; bridge plumbing; one new host-fn version | None | A new storage column, retention policy, pruning logic, and reads | One RPC method, layered above |
| Ledger-version churn surface | Wrapper-stable across versions; payload is opaque tagged bytes | Indexer rebuild on every event-shape change | Same as wrapper — once defined | Inherits from underlying |
| Retention story | Inherits `frame_system::Events` default; archive opt-in via existing pruning config | None | Pallet-controlled retention window | Inherits from underlying |
| Pallet ABI surface | One appended `Event` variant | None | One appended `Event` variant plus storage items | Inherits from underlying |

The mechanism specified in `## Specification` is **(a)**.

The decisive properties are light-client friendliness and tooling reach. `frame_system::Events` is part of the storage trie; light clients verify any event through `state_getReadProof` without needing the full ledger state or trusting any specific indexer ([`substrate/frame/system/src/lib.rs::Events` storage definition](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/frame/system/src/lib.rs); the storage value is committed by the block's storage root). The same record decodes against the runtime's metadata in `polkadot.js`, in `subxt`, and in every Substrate-tooling integration teams have already built — which is the explicit UC5 promise in MPS-0007.

Mechanism **(b)** preserves the present-day arrangement: events remain derivable but only via replay. It demands the indexer (or any equivalent) to be operationally available and version-pinned to the same ledger crate as the node, and it leaves the UC1 / UC2 / UC3 / UC5 use cases unaddressed. It is the implicit baseline for measuring (a); it is not an action-bearing alternative.

Mechanism **(c)** adds a pallet-controlled retention policy. The benefit is that operators get a knob — events are pruned on a window the pallet owns, not on the node's archive setting. The cost is a new storage column to migrate, a new retention parameter to govern, and a custom subscription surface that consumer libraries do not already understand. The wire shape this MIP specifies (`LedgerEvent` + `LedgerEventSource`) is portable to (c) without re-spec — the same bytes can ride a pallet `StorageMap` rather than `frame_system::Events`. If operator experience under (a) shows archive growth is unsustainable, a future MIP can switch the persistence mechanism to (c) without changing the consumer-facing decoder.

Mechanism **(d)** is a convenience wrapper, not an alternative. A typed `midnight_subscribeBlockEvents` RPC method would offer non-Substrate-native consumers a JSON-friendly entry point. Because `state_subscribeStorage([System.Events])` already provides real-time delivery decodable through any Substrate client, (d) is deferred from this MIP's scope: it can ship in a follow-on without altering the wire shape this MIP commits to, on its own merits when the consumer benefit justifies it.

Within mechanism (a), three sub-decisions warrant explicit rationale:

- **Opaque tagged bytes for `EventDetails<D>` rather than a SCALE-mirrored enum.** The ledger crate's `EventDetails<D>` is `#[non_exhaustive]`; mirroring its variants in SCALE creates a maintenance ratchet on every ledger upgrade and risks silent drift if a new variant lands. Carrying the ledger's own `Tagged` byte form makes the wrapper forward-compatible — consumers decode against the ledger version they ship with, and the runtime ABI does not move when the event taxonomy does.
- **One pallet-event variant per ledger event rather than one variant per block carrying a `Vec`.** `subxt` and equivalent clients filter events by `(pallet_index, variant_index)` *before* SCALE-decoding the payload. A block-level `Vec<LedgerEvent>` variant forces clients to decode the whole vec before filtering, defeating the typed event channel's filtering granularity. One variant per event aligns with the existing pallet variants (`ContractCall`, `ContractDeploy`, `ContractMaintain`, ...) which each fire once per occurrence inside a single transaction.
- **A new `TransactionAppliedStateRootV2` sibling struct rather than extending the existing struct in place.** SCALE does not silently ignore trailing bytes on decode. Extending the existing struct would break replay against WASM runtimes compiled against the old encoding. A sibling struct, returned by the new `#[version(3)]` of the host-fn, leaves historical replay paths byte-stable. This follows the precedent already in tree (`ledger/src/host_api/ledger_7.rs::apply_transaction` v1 vs v2).

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

The implementation is prepared in [`midnight-node` PR #1487](https://github.com/midnightntwrk/midnight-node/pull/1487), addressing [`midnight-node` issue #1474](https://github.com/midnightntwrk/midnight-node/issues/1474). The PR is paused pending this MIP's transition to `Accepted` and will be merged once that acceptance is recorded (see §"Path to Active" for the sequencing). It modifies the following components, each of which is sufficient context for the change in isolation:

- **`ledger/src/versions/common/api/ledger.rs`** — `Ledger::apply_verified_transaction` and `Ledger::apply_system_tx` are extended to return the event vector alongside the existing `AppliedStage` / `Sp<Self, D>` outputs. The two call sites in `ledger/src/versions/common/mod.rs` consume the new return shape. The pre-implementation discard sites (events bound to `_` in the `TransactionResult` match arms) are removed.
- **`ledger/src/common/types.rs`** — defines the new `LedgerEvent` and `LedgerEventSource` SCALE types specified in §"Specification", and a new `TransactionAppliedStateRootV2` sibling struct carrying `events: Vec<LedgerEvent>`. The pre-existing `TransactionAppliedStateRoot` is retained for replay through the pre-events host-fn versions.
- **`ledger/src/versions/common/mod.rs`** — `Bridge::apply_transaction` and `Bridge::apply_system_transaction` populate the new `events` field by tagged-serialising each `Event<D>` returned from the ledger API.
- **`ledger/src/host_api/ledger_7.rs` and `ledger/src/host_api/ledger_8.rs`** — `#[version(3)]` of `apply_transaction` and `#[version(2)]` of `apply_system_transaction` return `TransactionAppliedStateRootV2`. The pre-existing `#[version(1)]` and `#[version(2)]` of `apply_transaction` (and `#[version(1)]` of `apply_system_transaction`) remain in tree for historical replay.
- **`pallets/midnight/src/lib.rs`** — the pallet's `Event` enum gains a `LedgerEvent(LedgerEvent)` variant appended at the tail. `send_mn_transaction` calls the new host-fn version, iterates `result.events`, and deposits one runtime event per `LedgerEvent`.
- **`pallets/midnight-system/src/lib.rs`** — symmetric change for the system-transaction surface; system-tx events are deposited on the same `pallet_midnight::Event::LedgerEvent` channel.

There are no new external dependencies: the change reuses Substrate's existing FRAME event machinery, the existing host-fn versioning pattern (already established by the v1 → v2 transition of `apply_transaction`), and the existing pallet event deposit path. The runtime upgrade that activates the change increments the runtime's `spec_version`; no other coordinated rollout is required of operators beyond a standard runtime upgrade.

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

- Substrate `frame_system::Events` storage declaration: [`substrate/frame/system/src/lib.rs` @ `polkadot-stable2603`](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/frame/system/src/lib.rs) — the `StorageValue<_, Vec<EventRecord<...>>, ValueQuery>` and the per-block `reset_events()` reset path called by `frame_executive::Executive::initialize_block_impl`.
- Substrate `frame_executive::Executive::initialize_block_impl`: [`substrate/frame/executive/src/lib.rs` @ `polkadot-stable2603`](https://github.com/paritytech/polkadot-sdk/blob/polkadot-stable2603/substrate/frame/executive/src/lib.rs) — the call site that invokes `System::reset_events()` at the start of each block.

### Consumer-side tooling

- [subxt event filtering docs](https://docs.rs/subxt/latest/subxt/events/index.html) — pattern reference for consumers decoding `pallet_midnight::Event::LedgerEvent(_)` from `frame_system::Events`.

## Acknowledgements

Thanks to Giles Cope (author of MPS-0007: Node-Side Visibility of Ledger Events) for framing the upstream Problem Statement that this MIP responds to, and to Oscar Bailey for early stakeholder feedback on the publication-mechanism trade-offs that informed the choice of `frame_system::Events` (mechanism (a) in the Rationale). Additional acknowledgements — reviewers on the present PR, MIP editors who shape the draft during numbering, and community commenters during the Proposed period — will be added during the PR review cycle.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
