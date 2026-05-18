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

_Section drafted from the work-package plan and assumptions log during the `implement` activity._

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
