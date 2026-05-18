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

_Section drafted from the design philosophy and research artifacts during the `implement` activity._

## Path to Active

### Acceptance Criteria

_Drafted during `implement`._

### Implementation Plan

_Drafted during `implement` — references the upstream PR in `midnightntwrk/midnight-node` once opened._

## Backwards Compatibility Assessment

_Section drafted during `implement` — sources the backwards-compatibility analysis already completed in the implementation work package._

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
