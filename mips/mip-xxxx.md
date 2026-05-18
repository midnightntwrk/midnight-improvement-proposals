---
MIP: ?
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

The decisive properties are light-client friendliness and tooling reach. `frame_system::Events` is part of the storage trie; light clients verify any event through `state_getReadProof` without needing the full ledger state or trusting any specific indexer ([`substrate/frame/system/src/lib.rs::Events` storage definition](https://github.com/paritytech/polkadot-sdk/blob/master/substrate/frame/system/src/lib.rs); the storage value is committed by the block's storage root). The same record decodes against the runtime's metadata in `polkadot.js`, in `subxt`, and in every Substrate-tooling integration teams have already built — which is the explicit UC5 promise in MPS-0007.

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

The implementation work is in flight in [`midnight-node` PR #1487](https://github.com/midnightntwrk/midnight-node/pull/1487) and will land before this MIP's `Accepted` status is reached — the reverse of the typical MIP-0001 sequencing, but explicitly permitted by MIP-0001 §"Implementation". The path to `Active` is therefore: editor numbering of the present draft → community-commentary and editor vote → status transition to `Accepted` → status transition to `Implemented` (which can be marked immediately because the implementation PR has by then merged) → status transition to `Active` when the runtime upgrade that carries the new `pallet_midnight::Event::LedgerEvent` variant and the `apply_transaction` `#[version(3)]` host-fn enters a deployed Midnight network. A typed `midnight_subscribeBlockEvents` RPC wrapper is held as a follow-on that can ship in a subsequent runtime upgrade — or in a node-only release — without further MIP work; the wire shape this MIP specifies is the layered protocol that wrapper would surface.

## Backwards Compatibility Assessment

The change does **not** require a hard fork. It ships in a normal Substrate runtime upgrade. Compatibility is best understood as three distinct layers, each with its own stability commitment.

**Layer 1 — Pallet ABI stability (existing event variants).** None of the eight pre-existing `pallet_midnight::Event` variants change name, position, payload type, or `#[codec(index)]`. `LedgerEvent` is appended at the tail of the enum. Any consumer whose code already decodes `frame_system::Events` against the pallet's metadata — `polkadot.js`, `subxt`, the indexer's pallet-event reader, custom Substrate-tooling integrations — keeps working unchanged on every block produced before the activation point and on every block produced after the activation point. New consumers opt into the `LedgerEvent` variant by recompiling against the new pallet metadata. Confirmed by acceptance criterion M6.

**Layer 2 — Wire-shape stability across ledger-crate versions (the wrapper).** The `LedgerEvent { source: LedgerEventSource, content_tagged_bytes: Vec<u8> }` struct has a fixed outer SCALE shape: the routing header is 32 + 2 + 2 bytes; the payload is a length-prefixed byte vector. The wrapper is stable across `mn-ledger` v7 and v8 — both ledger versions tag their `EventDetails<D>` as `event-details[v9]` and the wrapper carries the tag inside `content_tagged_bytes` rather than reflecting it into the SCALE envelope. The wrapper continues to be stable when `mn-ledger` adds a new `EventDetails` variant (the variant appears as different bytes inside `content_tagged_bytes`, leaving the wrapper untouched), when fields are added to an existing variant (same), or when `mn-ledger` cuts a new major version (the tag string inside the payload changes; the wrapper does not). The wrapper would change only if the slot-identity triple — `(transaction_hash, logical_segment, physical_segment)` — were to gain a fourth dimension, or if a future MIP deliberately chose to abandon the tagged-byte payload encoding. Either is governed by a future MIP, not by ledger-crate evolution. Confirmed by acceptance criterion M3.

**Layer 3 — Pallet host-fn ABI stability (`apply_transaction` versioning).** The new behaviour ships behind `#[version(3)]` of `apply_transaction` and `#[version(2)]` of `apply_system_transaction` on both `host_api/ledger_7.rs` and `host_api/ledger_8.rs`. The pre-existing `#[version(1)]` and `#[version(2)]` of `apply_transaction` (and the pre-existing `#[version(1)]` of `apply_system_transaction`) remain in tree. Substrate's runtime-upgrade machinery routes each WASM blob to the host-fn version it was compiled against, so historical block replay against pre-activation WASM continues to work without any new branch in the pallet itself. This pattern is the same one already established by the v1 → v2 transition of `apply_transaction` documented in the host-API source. Any future version bump can follow the same shape — append a new `#[version(N)]` rather than mutate an existing one. Confirmed by acceptance criterion M7.

**Hard-fork question.** No. The change ships as a runtime upgrade. The runtime's `spec_version` is incremented to flip the pallet's call site from the pre-events host-fn version to `#[version(3)]`. Validators that have not yet upgraded continue producing blocks correctly under their compiled-in host-fn version; their blocks contain no `LedgerEvent` records and replay cleanly. Validators that have upgraded produce blocks carrying `LedgerEvent` records on top of the existing event surface. Block replay across the upgrade boundary works because the pre-events WASM blobs route to their compiled-in `#[version]` (M7).

**Archive-node disk growth.** The one *operational* compatibility concern is archive-node disk consumption. At a typical event envelope of approximately 45 KiB per block and a heavy envelope of 50–200 KiB per block, six-second blocks produce roughly 6.5 MiB per hour on a typical mix and roughly 30 MiB per hour on a sustained-heavy mix — on the order of 57 GiB per year typical and 263 GiB per year sustained heavy. These figures are modest in absolute terms (a one-terabyte archive volume absorbs a decade of typical load), but archive operators should account for them when sizing storage. Default-pruned nodes are unaffected: `frame_system::Events` is reset at every block, and the keep-window dictates when a block's events leave active state. Archive operators opt into the cost via the same Substrate archive configuration that retains every other pallet event; this MIP introduces no new flag. If sustained operational experience indicates the envelope is unsustainable, mechanism (c) from the Rationale — a pallet-controlled storage column with bounded retention — is the documented fallback, and the wire shape this MIP commits to is portable to it without consumer-side churn.

## Security Considerations

_Section drafted during `implement`._

## Implementation

_Drafted during `implement` — pointers to the upstream node PR and runtime changes._

## Testing

_Section drafted from `.engineering/artifacts/planning/2026-05-11-node-expose-ledger-events/05-test-plan.md` during `implement`._

## References

- [MPS-0007: Node Event Visibility](../mps/mps-0007-node-event-visibility.md)
- [Issue #1474 — Node should expose events emitted by the Ledger](https://github.com/midnightntwrk/midnight-node/issues/1474)

## Acknowledgements

_To be populated._

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
