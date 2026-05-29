---
MPS: 0007  
Title: Node-Side Visibility of Ledger Events  
Authors: Giles Cope <giles.cope@shielded.io>  
Status: Proposed  
Category: Core  
Created: 2026-05-13  
Requires: none  
Replaces: none  
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

The Midnight ledger crate already produces typed event records (`ZswapInput`, `ZswapOutput`, `ContractDeploy`, `ContractLog`, `ParamChange`, dust events, ...) during transaction application. Inside the midnight-node these events are discarded at the ledger-bridge boundary before they reach any observable surface: they do not appear in the node's substrate FRAME event stream, are not visible to polkadot.js, and are not available to light clients or RPC consumers. Today the only consumer that recovers them is the midnight-indexer, which does so by re-running each transaction against its own copy of the ledger crate. This leaves a gap: protocol-level events that the network produces and pays gas for are invisible to anyone who cannot, or does not wish to, run a full indexer alongside their node. This MPS scopes the problem of making ledger events observable from a Midnight node through standard substrate tooling.

## Vision

A developer or operator running a Midnight node — for debugging, light-client integration, block-explorer development, or local DApp work — can see every event produced by the ledger in a given block, in execution order, through the same standard tooling that exposes pallet events today (polkadot.js, substrate RPC, subscription APIs). What the ledger emits and the network pays gas for is what the node makes available.

## Problem

### Current State

The ledger's `state.apply(verified_tx, ctx)` returns a `TransactionResult` whose `Success` and `PartialSuccess` variants carry the vector of events produced during transaction execution. At the midnight-node ledger bridge, the events are pattern-matched into a `_` binding and discarded:

```rust
let (next_state, result) = sp.state.apply(verified_tx, ctx);
match result {
    TransactionResult::Success(_) => Ok((new_sp, AppliedStage::AllApplied)),
    //                         ^ events discarded
    TransactionResult::PartialSuccess(segments, _) => { ... }
    //                                          ^ events discarded
    ...
}
```

The bridge's output type, `TransactionAppliedStateRoot`, carries `state_root`, `tx_hash`, `all_applied`, lists of call/deploy/maintain addresses, claim rewards, and unshielded UTXO deltas — but no events vector. The pallet reads from that struct and deposits FRAME events derived from those fields only.

The result:

- Events that already exist at the ledger layer, are deterministically produced by every validator, are gas-metered, and are accounted against block-level byte budgets, are nevertheless invisible at the node's public surface.
- The only path to observe these events is to run the midnight-indexer alongside a node and let it re-execute each transaction. The indexer holds a parallel ledger state copy, replays transactions, and then itself filters some variants out (`ContractLog` is currently dropped in both v7 and v8 adapters). For the events the indexer does surface, consumers must use its GraphQL API rather than the substrate RPC layer they may already be integrated with.

### Limitations

**Polkadot.js and substrate RPC users see nothing of contract-internal events.** A developer browsing recent blocks in polkadot.js sees pallet-level summary events but no per-event payloads from inside contract execution. There is no way, using standard substrate tooling, to ask "what events did this transaction produce?".

**Light clients have no recourse.** A light client receiving a `TransactionAppliedStateRoot` cannot reconstruct the events of the transaction. To get them, it must either replay the transaction itself (requiring the full ledger crate and state) or query a separately operated indexer.

**Debugging is opaque.** Contract authors and node operators investigating a failing or unexpected interaction must either attach to a debug build or stand up a full indexer. There is no lightweight visibility path.

**The indexer carries protocol-replay duty.** Because the node does not surface events, every consumer that wants them must replay transactions themselves or rely on the indexer to do so. This couples event observability to indexer availability, indexer version compatibility, and indexer operational health, for a class of data the node has already computed.

**Existing tooling integrations break at the Midnight boundary.** Teams that already consume substrate events from other Substrate-based chains through their existing tooling cannot use that pipeline for Midnight; they must add a Midnight-indexer-specific integration for data the node has but does not expose.

## Use Cases

**UC1: Contract author debugging an emit.** A developer writing a Compact contract that emits events (under the framework proposed by the events CoIP) wants to verify, on their local devnet, that events are produced as expected — without operating an indexer. With node-side event visibility they open polkadot.js, send a transaction, and see the events in the Recent Events panel.

**UC2: Block explorer minimum viable surface.** A block explorer maintainer wants to display contract events per block. With node-side surfacing the explorer reads events from the standard substrate event stream rather than maintaining a parallel indexer integration.

**UC3: Light-client DApp.** A mobile DApp wants to subscribe to events from a specific contract without downloading the full ledger state or relying on a centralised indexer. With node-side event visibility the DApp uses substrate subscription APIs to receive events as part of normal transaction result handling.

**UC4: Operations and incident response.** An operator investigating an unexpected on-chain state change wants to inspect the events emitted by the relevant transactions. Today this requires standing up an indexer replay or attaching debug tooling. With node-side surfacing the events are visible directly via RPC to any running node.

**UC5: Cross-tooling compatibility.** A team integrating Midnight into a multi-chain dashboard already consumes substrate events from other Substrate-based chains. Today they cannot use that pipeline for Midnight. Node-side event surfacing lets the same tooling work without a Midnight-specific integration layer.

## Goals

- **Make ledger-produced events observable through the standard node surface** — substrate FRAME events, RPC, subscription APIs — for the event types the ledger crate produces today, in the order it produces them.
- **Match event ordering and content to what validators compute deterministically.** No node should produce a different view of events than another node for the same finalized block.
- **Stay within existing protocol constraints.** Add no new on-chain semantics, no new gas costs, no new block-validity requirements. The events to be surfaced are already produced and already paid for.
- **Preserve light-client friendliness.** Whatever surface is added should be reachable by light clients (storage proofs or subscription APIs), not only by archive nodes.
- **Be independent of the events CoIP.** Node-side visibility should ship without depending on Compact language changes, and conversely, the events CoIP should be able to assume node-side visibility as a building block rather than re-prove it.

## Expected Outcomes

- Contract developers can debug events using polkadot.js without operating an indexer.
- Block explorers and analytics tools can consume Midnight events through standard substrate tooling.
- The midnight-indexer is freed from being the sole consumer of ledger events; it becomes one of several consumers of an already-available data stream.
- Light clients and mobile DApps gain a path to event subscriptions that does not require trusting a hosted indexer.
- The events CoIP's Phase 1 immediately becomes useful to a wider audience, because events emitted by Compact `emit` statements land on a surface that DApp developers can already see.

## Open Questions

1. **Event scope.** Should the node surface every `EventDetails` variant the ledger produces (`ZswapInput`, `ZswapOutput`, `ContractDeploy`, `ContractLog`, `ParamChange`, dust events ...) or initially only the subset most relevant to contract-author debugging (`ContractLog`)? The indexer surfaces some variants and filters others; the node could match or diverge from that policy.
2. **Payload encoding for FRAME events.** Ledger events carry richly-typed payloads (e.g. `StateValue` for `ContractLog`). FRAME events use SCALE encoding. Options include: surfacing raw tagged-serialized bytes (opaque to polkadot.js but decodable by midnight-js), exposing typed FRAME variants per event kind, or a hybrid. Each option has different impacts on polkadot.js rendering, metadata size, and future event-shape evolution.
3. **Block-storage impact.** Per-block pallet events are persisted in `System::Events`. The ledger's existing per-transaction limit (currently 512 KB per `Log` opcode invocation) and per-block byte budgets bound the total; only archive nodes would save all events.
4. **Relationship to the events CoIP.** The CoIP currently lists midnight-ledger as "no changes required" and does not list midnight-node as affected at all. If node-side surfacing lands first, the CoIP's "Affected Components" list and its narrative around ephemeral events should be updated.

## Recommended MIPs

The following MIPs are recommended to address the problem domain identified in this MPS:

1. **Node-side ledger event surfacing.** Specify the bridge-layer change (propagating events from `apply_verified_transaction` through `TransactionAppliedStateRoot`), the new pallet event variant(s), encoding choices for payloads, and handling across ledger versions.`.

## References

- Existing events MPS in this repository: [`mps-0005-events.md`](mps-0005-events.md)
- `TransactionAppliedStateRoot` struct definition - no `events` field: [`midnight-node` @ `b49ba64` - `ledger/src/common/types.rs` L46](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/ledger/src/common/types.rs#L46)
- Ledger-bridge discard site: events bound to `_` in `apply_verified_transaction`: [`midnight-node` @ `b49ba64` - `ledger/src/versions/common/api/ledger.rs` L142&ndash;L158](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/ledger/src/versions/common/api/ledger.rs#L142-L158)
- `TransactionAppliedStateRoot` construction: no events forwarded: [`midnight-node` @ `b49ba64` - `ledger/src/versions/common/mod.rs` L414](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/ledger/src/versions/common/mod.rs#L414)
- Pallet FRAME event enum: [`midnight-node` @ `b49ba64` : `pallets/midnight/src/lib.rs` L241](https://github.com/midnightntwrk/midnight-node/blob/b49ba64beae0ede6f7c8a81eff6e840079fcae9b/pallets/midnight/src/lib.rs#L241)
