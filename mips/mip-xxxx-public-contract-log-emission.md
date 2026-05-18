---
MIP: xxxx
Title: Public Contract Log Emission for Compact Smart Contracts
Authors:
  - Dominik Zajkowski (@dzajkowski)
Status: Draft
Category: Core
Created: 2026-05-15
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

Compact smart contracts on Midnight have no mechanism to emit structured notifications about contract activity.
Consumers must poll contract state, diff serialized values, and reverse-engineer meaning from internal state layouts.

The infrastructure for log emission already exists but is disconnected: the Impact VM provides a `Log` opcode (0x09), the ledger wraps logged values as `EventDetails::ContractLog`, and the indexer discards them.

This MIP connects the pipeline.
The Compact language gains a `log` expression — proposed separately as [CoIP-xxxx](https://github.com/LFDT-Minokawa/compact/pull/442) — which compiles to the existing `Log` opcode.
The ledger wraps payloads in a versioned envelope (`VersionedLogItem`), extending the `Log` opcode's data field while maintaining backwards compatibility.
The indexer surfaces events via GraphQL.

Phase 1 restricts emission to a fixed set of standard event types (`LogEventType` enum), with a `Misc` escape hatch for custom data.
The indexer derives indexed fields from the enum definition — no explicit marking by the contract author.
A standard events package covers common operations (shielded/unshielded transfers, mints, burns).
Events are capped at 1 KB serialized.

No new VM opcodes are introduced.
Events are not consensus state — retention is a downstream policy.

This MIP addresses the problem defined in [MPS-events](../mps/mps-0005-events.md).

## Motivation

DApp developers on Midnight have no notification mechanism for contract activity.
To track what a contract has done, a consumer must:

- Poll the contract's serialized state via RPC (`midnight_contractState`), or subscribe to contract state snapshots via the indexer's GraphQL API (`CONTRACT_STATE_SUB`), diff the full serialized state after each action, and decode what changed.
- Or replay every `ContractCall` transaction targeting the address, diff the `ChargedState` before/after, and interpret the deltas.

Both approaches are expensive and introduce polling latency.

The infrastructure for log emission already exists but is disconnected:

- The Impact VM provides a `Log` opcode (0x09) that captures a `StateValue` during execution.
- The ledger wraps logged values as `EventDetails::ContractLog`.
- The indexer discards `ContractLog` entries before they reach storage (`ContractLog { .. } => None`).

Structured events improve access — consumers subscribe to labeled, typed notifications rather than diffing raw state.
They do not eliminate the need to understand a contract's schema, and consumers must treat events as the contract author's representation of what happened, not as independently verifiable facts.

## Specification

### Versioning

The specification evolves along two axes:

- **Event payload schemas** — the `version` field in `VersionedLogItem` allows the field layout of a given `LogEventType` variant to change across ledger releases. Consumers use the version to select the correct decoder.
- **Standard event set** — adding a new `LogEventType` variant requires a ledger release and a bumped serialization tag. New variants are amendments to this MIP and do not require a superseding proposal.

Changes to the structure of `VersionedLogItem` itself, to the `EventDetails` envelope, or to the fee model require a new MIP that supersedes this one.

### VersionedLogItem

The compiler emits `Log` with a `StateValue` payload.
The VM parses it into a typed wrapper at event construction time, extracting version, event type, and data.
This requires a bumped `EventDetails` serialization tag.

```rust
pub enum LogEventType {
    ShieldedSpend,
    ShieldedReceive,
    ShieldedMint,
    ShieldedBurn,
    UnshieldedSpend,
    UnshieldedReceive,
    UnshieldedMint,
    UnshieldedBurn,
    Paused,
    Unpaused,
    Misc,
}

#[tag = "impact-versioned-log-item[v1]"]
pub struct VersionedLogItem<D: DB> {
    pub version: u32,
    pub event_type: LogEventType,
    pub data: StateValue<D>,
}
```

- `version` — payload schema version. Allows consumers to distinguish between versions as the structure evolves.
- `event_type` — a `LogEventType` enum variant. Phase 1 restricts emission to these variants. Adding a new variant requires a ledger release.
- `data` — the serialized struct as `StateValue`.

Non-conforming log payloads (e.g. from hand-crafted bytecode) degrade gracefully to version 0 with `Misc` event type.

### Standard Events

Standard events are protocol-level enum variants in `LogEventType`.
Each variant has a fixed schema enforced by the compiler, the indexer knows the field layout without a descriptor.
The `Misc` variant is the escape hatch for custom data.
Contract authors choose when to emit standard events, no automatic emission.

Details on proposed Standard Events are in [Appendix A](#appendix-a-standard-events-definition).

### Schema Sizing

For this iteration, a single log event must not exceed 1 KB serialized.
The 1 KB limit applies to the uncompressed payload — v1 does not compress event data.
The `event_type` field is an enum variant reference, taking a fixed small amount of space in the serialized payload.
The size limit can be loosened in future updates if the need for larger events is merited.

**Open Question?:** Should there be a hard cap on the number of events a contract can emit?

### Fee Metering

The existing `Log` opcode gas model already implements a `base+variable` fee structure.
No new fee metering is needed.

The `Log` opcode charges:
- A **constant** per payload type (e.g., 610,437 ps for Array, 314,563 ps for Cell)
- A **per-byte coefficient** proportional to serialized size (e.g., 18,027 ps/byte for Array)
- Both `bytes_written` and `bytes_deleted` equal to serialized size, counted as churn

These costs feed into the multi-dimensional fee formula:
```
total_fee = (max(read_cost, compute_cost, block_cost) + write_cost + churn_cost) * overall_price
```

Per-block throughput is bounded by shared block budgets: `bytes_written: 50 KB`, `bytes_churned: 1 MB`.
Events compete with state writes for the same budget.

### Event Lifetime

Events are not consensus state — the node discards them after verification, and chain replay does not depend on them.
The serialized form is part of the transactions transcript. 
In that form events carry no meaning, and are stored as long as a particular block is stored.
Parsed event retention is a downstream policy, set per use case by the consumer (indexer, application, archival service).
The protocol takes no position on how long parsed events are retained.
Events are deterministically reproducible by replaying chain history, so short retention does not lose data — it moves the cost of historical access from the indexer to the consumer.
See [Appendix B](#appendix-b-indexer-storage-impact) for storage projections under different retention windows.

### Event Indexing

Standard events provide hints on which fields could be indexed as part of their definition.
The indexer knows the field layout from the `LogEventType` enum variant — no descriptor or explicit marking needed.
`Misc` fields require custom processing rules per application.

The indexer stores indexed field values in a sidecar table, following the same pattern used for dust and zswap nullifiers.
Consumers query by prefix and filter to exact match client-side — same pattern as `dustNullifierTransactions`.

The `prefix` filter can be empty or carry some prefix bytes of the requested value set.
This allows the DApp implementer to choose the level of disclosure at the cost of the data returned.

The indexer exposes contract events via both GraphQL query (historical lookup) and subscription (real-time stream).
The query is the higher priority for the initial implementation.

```graphql
input FieldPrefixFilter {
    fieldName: String
    prefix: HexEncoded!
}

input ContractEventFilter {
    contractAddress: HexEncoded!
    eventType: String!
    fieldPrefixes: [FieldPrefixFilter!]
}

contractEvents(
    filter: {
        contractAddress: "0xabc...",
        eventType: "ShieldedSpend",
        fieldPrefixes: [{ fieldName: "nullifier", prefix: "a3b7" }]
    }
)
```

The indexer resolves `fieldName` for all standard events.
`Misc` events do not support indexing by default.
If indexing is needed for `Misc`, it must come from a descriptor or be configured by policy.

Each event receives a monotonic `id` (backed by `BIGSERIAL`), used for ordering, resumption, and deduplication.
Subscriptions accept an `id` parameter to resume from a known position — same mechanism as `dustLedgerEvents(id: Int)` and `zswapLedgerEvents(id: Int)`.

The `Log` opcode gas calculation accounts for the full `VersionedLogItem` serialized size, but it does not compensate downstream applications for the upkeep cost of indexed items.

## Rationale

**Why `VersionedLogItem` at the ledger level (Option A)?**

Three approaches were considered for schema versioning:

- **Option A (chosen):** `VersionedLogItem` wrapper parsed at the ledger level. No VM or opcode changes. Consumers get typed `version` and `event_type` fields. Standard event types are enforced at the protocol level. Non-conforming logs degrade gracefully to version 0.
- **Option B:** Explicit `version` and `event_type` fields on `ContractLog`, propagated through the VM. Highest impact — touches VM opcodes, `ResultMode` trait, `EventDetails` serialization tag, gas model, and compiler code generation.
- **Option C:** Convention-based versioning inside `StateValue`. Zero protocol changes, but convention-only — hand-crafted bytecode can bypass it.

Option A was chosen because it provides protocol-level enforcement with minimal blast radius — no VM changes, no new opcodes, one bumped serialization tag.

**Why `LogEventType` enum instead of string-based event names?**

String-based names create a naming collision risk and require the indexer to carry TypeScript descriptors to understand field layouts.
An enum makes standard events a protocol-level concern, the indexer ships with built-in schema knowledge, no descriptor sharing mechanism needed at this point.
The `Misc` variant preserves flexibility for custom data.

**Why indexed fields are derived from the enum, not specified by the contract?**

For standard events the indexer already knows the schema.
Requiring the contract to redundantly declare indexed fields adds complexity for no benefit.
`Misc` events have no indexing hints available, the escape hatch trades convenience for flexibility.

**Why not IR-based field resolution?**

Standard events solve the field resolution problem for the initial implementation — the indexer knows every field layout from the `LogEventType` variant alone.
For custom events (`Misc`) and future user-defined event types, a more flexible approach would use the compiler's IR to derive field schemas on-chain, removing the need for out-of-band descriptor sharing.
This requires ZKIR-on-chain integration that is not yet in place.
The current design is compatible with this path — `VersionedLogItem` and `LogEventType` do not preclude IR-based resolution in a future iteration.

**Why events are not consensus state?**

Events are a notification mechanism, not a state commitment.
As such there is no practical way of enforcing that an event is emitted correctly.
A user still needs to inspect the contract to make sure that the event they are looking for was emitted from the place in the code that they expect.

On the other hand, the event is part of the transaction transcript and as such is guaranteed to be exactly what the user posted to the chain for verification and emission.

**Why no domain-level event interpretation?**

This proposal covers two layers of the event data flow:

1. **Contract events** — the emission mechanism.
A contract decides what happened and emits a structured record.
Covered by `log`, `VersionedLogItem`, and disclosure rules.
2. **Event stream access** — the query and subscription API.
Consumers fetch or subscribe to events by contract, type, or indexed prefix.
Covered by the indexer's GraphQL surface.

A third layer — **derived domain models** — sits above these.
Application-level entities materialized by folding events from one or more contracts into stateful views (e.g. a DeFi position aggregated from deposit, borrow, price, and liquidation events across multiple contracts).
This is what DApp builders ultimately code against.

The domain representation is out of scope for the protocol.
Domain models are application-specific — no universal schema covers DeFi positions, NFT collections, governance state, and identity attestations at once.
On other ecosystems this layer is built by indexer frameworks (The Graph subgraphs, Subsquid, Goldsky), application backends, or domain-specific libraries.
The same pattern is expected here.

Two decisions in this proposal support domain-level data even though it lives outside the on-chain representation:
- **Deterministic replay** — DApp stores can be rebuilt by replaying chain history, so retention at the indexed layer does not bound what a specific application can offer.
- **Schema versioning** — a DApp may hold events emitted under older schemas long after the contract has upgraded; `VersionedLogItem` lets it disambiguate them.

**Why not node-side event emission?**

A separate problem statement proposes having the node emit events directly to subscribers, rather than requiring the indexer to replay transactions independently.
See [MPS: Node Event Visibility](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-node-event-visibility.md).
This is a significant change to how the indexer operates and can be pursued independently of the current user-oriented feature.

## Path to Active

### Acceptance Criteria

- Ledger parses `Log` payloads into `VersionedLogItem` with `LogEventType` enum.
- Compiler maps standard event structs to `LogEventType` variants and rejects unknown variants in Phase 1.
- Indexer stores `ContractLog` entries (currently discarded) and exposes them via GraphQL query and subscription.
- Standard events are queryable by contract address, event type, and indexed field prefix.
- SDK (`compact-js`, `midnight-js`) surfaces event queries and subscriptions.
- Toolkit surfaces events to DApp developers and decodes event payloads.
- Non-conforming log payloads degrade to version 0 without breaking the pipeline.

### Implementation Plan

The workstreams are:

- **Ledger**: adds `LogEventType` enum and `VersionedLogItem` parsing. Bumps `EventDetails` serialization tag. Prototype available in [Midnight Ledger PR](https://github.com/midnightntwrk/midnight-ledger/pull/503/changes).
- **Compact compiler**: adds the `log` expression (proposed separately as [CoIP-xxxx](https://github.com/LFDT-Minokawa/compact/pull/442)). Maps standard event structs to `LogEventType` variants. Compact work tracked in [Compact issue](https://github.com/LFDT-Minokawa/compact/issues/377).
- **Indexer**: removes the `ContractLog { .. } => None` filter, adds `Contract` variant to `LedgerEventGrouping`, adds storage and GraphQL surface. Ships with built-in schema knowledge for all `LogEventType` variants.
- **SDK & Toolkit**: `compact-js` captures log events from circuit execution. `midnight-js` exposes event queries and subscriptions. The toolkit surfaces events to DApp developers and decodes event payloads.

The compiler and indexer have no technical dependency on each other.
Both depend on the ledger change.

If the indexer ships first, it begins storing `ContractLog` entries, but no existing contract produces them — no observable effect.
If the compiler ships first, contracts can emit events, but the indexer continues to discard them — no breakage.
The feature becomes usable once all workstreams are deployed.

## Backwards Compatibility Assessment

No Compact contracts have been found that produce `Log` instructions, so no deployed contract should be affected.

The `Log` opcode (0x09) data field is modified to carry the `VersionedLogItem` structure.
The `EventDetails` serialization tag is bumped. 
New ledger code handles both the old unstructured payload and the new versioned format. 
Old ledger code cannot deserialize the new format, it will panic on encounter.

This requires a coordinated upgrade of all nodes before any new-format events are produced.
This is effectively a hard fork.

The indexer currently discards `ContractLog` entries.
Storing them is a new behavior with no impact on existing queries or subscriptions.

## Security Considerations

**Spam / resource exhaustion:**
Events compete with state writes for the shared per-block `bytes_written` budget (50 KB) and are metered by the existing `Log` opcode gas model.
No new spam vector is introduced beyond what state writes already allow.

**Event trust model:**
Events represent the contract author's claim about what happened, not independently verifiable facts.
Consumers must treat them accordingly, a malicious contract can emit misleading events.
This is inherent to any event system and not specific to this proposal.

**Disclosure:**
`log` is a disclosure site.
The compiler enforces that all emitted fields are already disclosed.
No new private data leakage is introduced, events carry values that are already public.

**Indexer storage growth:**
At sustained saturation, event storage reaches ~1.4 TB/year for standard events.
Archiving does not require indexing, chain history is sufficient to reproduce any event.
Indexing on the other hand can have strict pruning policies without data loss.
See [Appendix B](#appendix-b-indexer-storage-impact) for full capacity projections and retention policy analysis.

**Deserialization of untrusted payloads:**
The VM parses `Log` payloads into `VersionedLogItem`.
Malformed payloads are handled as parsing failures.

## Implementation

**Ledger:**
- Add `LogEventType` enum and `VersionedLogItem` struct to `onchain-vm/src/ops.rs`.
- Bump `EventDetails` serialization tag from `v10` to `v11`.
- Parse `Log` opcode payload into `VersionedLogItem` during transaction application. Malformed payloads are handled as parsing failures.

**Compact compiler** (proposed separately as [CoIP-xxxx](https://github.com/LFDT-Minokawa/compact/pull/442)):
- Add `log` expression to parser, type checker, and code generator.
- Map standard event structs to `LogEventType` variants.
- Reject unknown variants in Phase 1.
- Generate TypeScript descriptors for `Misc` events.

**Indexer:**
- Remove `ContractLog { .. } => None` filter in `indexer-common/src/domain/ledger/ledger_state.rs`.
- Add `Contract` variant to `LedgerEventGrouping`.
- Add storage table for contract events and sidecar table for indexed fields.
- Add `contractEvents` GraphQL query and subscription.
- Ship with built-in schema knowledge for all `LogEventType` variants.

**SDK & Toolkit:**
- `compact-js` captures log events from circuit execution.
- `midnight-js` exposes event queries and subscriptions.
- Toolkit decodes event payloads and surfaces them to DApp developers.

## Testing

**Ledger:**
- `VersionedLogItem` parsing: valid v1 payloads, malformed payloads degrade to version 0, oversized payloads (>1 KB) are silently swallowed (`None returned`).
    > This is to make the event emission limits 'soft', that is, changeable without changing core ledger semantics – the events emitted would change, but the state transition function would not if this were adjusted.
    > By contrast, a hard error would require a hard fork to change the bound.
    [Code link](https://github.com/midnightntwrk/midnight-ledger/pull/503/changes#diff-db969825d360f99ef145fa347e2c782866574dff80ed2ac49225d2501f37b04fR41)
- Serialization tag bump: old-format and new-format payloads coexist correctly.
- `LogEventType` enum: each variant maps to the correct standard event schema.

**Compiler:**
- `log` expression: emits correct `Log` opcode with `VersionedLogItem` payload.
- Disclosure enforcement: undisclosed fields produce compilation error.
- Standard event structs map to correct `LogEventType` variants.
- `Misc` variant accepts arbitrary struct data.
- Unknown variants rejected in Phase 1.

**Indexer:**
- `ContractLog` entries stored instead of discarded.
- GraphQL query: filter by contract address, event type, indexed field prefix.
- GraphQL subscription: real-time delivery, resumption via monotonic `id`.
- Sidecar indexed fields match standard event definitions.
- `Misc` events stored without indexing.

**Integration:**
- End-to-end: contract emits standard event → ledger wraps as `VersionedLogItem` → indexer stores → GraphQL query returns the event.
- Backwards compatibility: `Log` payloads that do not hold `version` information pass through the pipeline as version 0, event type `Misc`.

## References

- [MPS-events: Event Emission Support for Compact Smart Contracts](../mps/mps-0005-events.md)
- [Wallet Engine Specification — Indexing Service for Dust](https://github.com/midnightntwrk/midnight-architecture/blob/main/components/WalletEngine/Specification.md#indexing-service-for-dust)
- [Monero View Tags](https://github.com/monero-project/research-lab/issues/73)
- [Ethereum Logs and Events](https://docs.soliditylang.org/en/latest/abi-spec.html#events)
- [CoIP-xxxx: Compact `log` expression](https://github.com/LFDT-Minokawa/compact/pull/442)

## Acknowledgements

Many thanks for review and input to:
  - Parisa Ataei (@pataei)
  - Oscar Bailey (@ozgb)
  - Kent Dybvig (@dybvig)
  - Thomas Kerber (@tkerber)
  - Andrzej Kopeć (@kapke)
  - Sean Kwak (@cosmir17)
  - Szymon Paluchowski (@sp-io)
  - Darren Oliveiro-Priestnall (@DarrenOliveiro-Priestnall)
  - Giuseppe Salvatore (@GiuseppeSalvatoreShielded)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.

# Appendix A. Standard Events Definition

## Shielded Coin Events

```compact
struct ShieldedSpend {
    nullifier: Bytes<32>          // hint: indexed
}

struct ShieldedReceive {
    commitment: Bytes<32>,        // hint: indexed
    contractAddress: Maybe<ContractAddress>,
    ciphertext: Maybe<Bytes<512>> // hint: indexed
}

struct ShieldedMint {
    commitment: Bytes<32>,       // hint: indexed
    domainSep: Bytes<32>,        // hint: indexed
    amount: Maybe<Uint<128>>
}

struct ShieldedBurn {
    nullifier: Bytes<32>,        // hint: indexed
    amount: Maybe<Uint<128>>
}
```

## Unshielded Token Events

```compact
struct UnshieldedSpend {
    sender: Either<ZswapCoinPublicKey, ContractAddress>,     // hint: indexed
    domainSep: Bytes<32>,        // hint: indexed
    tokenType: Bytes<32>,        // hint: indexed
    amount: Uint<128>
}

struct UnshieldedReceive {
    recipient: Either<ZswapCoinPublicKey, ContractAddress>,  // hint: indexed
    domainSep: Bytes<32>,        // hint: indexed
    tokenType: Bytes<32>,        // hint: indexed
    amount: Uint<128>
}

struct UnshieldedMint {
    domainSep: Bytes<32>,        // hint: indexed
    tokenType: Bytes<32>,        // hint: indexed
    amount: Uint<128>
}

struct UnshieldedBurn {
    sender: Either<ZswapCoinPublicKey, ContractAddress>,     // hint: indexed
    tokenType: Bytes<32>,        // hint: indexed
    amount: Uint<128>
}
```

## Lifecycle Events

```compact
struct Paused {}
struct Unpaused {}
```

## Misc

```compact
struct Misc {
    name: Bytes<32>,
    payload: Bytes<256>
}
```

The indexer stores the raw payload but has no built-in schema knowledge.
Any indexing is entirely on the end user.

# Appendix B. Indexer Storage Impact

The fee model covers on-chain costs (see [Fee Metering](#fee-metering)).
Three downstream cost dimensions are not covered by gas:
1. **Storage** — persisting log items beyond the execution transcript (e.g., indexer database).
2. **Computation** — deserializing and decoding log payloads.
3. **Processing** — consumers must process every event to stay current.
Per-block gas limits bound total throughput but not the load on any individual consumer.

## Capacity Implications

A single event may take up to the 1 KB maximum.
Events compete with state writes for the shared per-block `bytes_written` budget (50 KB).
Upper bound: at 6-second block time with the full write budget consumed by events alone, raw event volume is ~703 MB/day / ~250 GB/year.

Indexed fields are stored separately from the event payload, following the same pattern the indexer uses for dust and zswap nullifiers.
The sidecar cost per row exceeds the event payload at small event sizes, making it the dominant storage cost.


### Calculation methodology

- **Events/day** = `bytes_written` budget (50 KB) / serialized event size × blocks/day (14,400 at 6s block time).
- **Sidecar rows/day** = events/day × indexed fields per event (capped at 3).
- **Sidecar row size** = ~150 bytes (rounded) in PostgreSQL: 76 bytes heap (23-byte tuple header, 8-byte `event_id`, 2-byte `field_index`, 33-byte `BYTEA` field value, alignment, item pointer) + 48-byte B-tree index on field value + 24-byte primary key index.
- **Raw event storage/day** = 703 MB (full write budget sustained).
Constant regardless of event size — it is the block budget itself.

### Storage projections (upper bound, sustained saturation)

| Scenario | Serialized event size | Events/day | Sidecar/day | Raw/day | Total/day | Total/year |
| --- | --- | --- | --- | --- | --- | --- |
| Standard events, all blocks saturated | ~100 B | 7.2M | 3.2 GB | 703 MB | 3.9 GB | ~1.4 TB |
| Maximum event size, all blocks saturated | 1 KB | 720K | 308 MB | 703 MB | 1.0 GB | ~365 GB |

The standard events scenario uses the average standard event (~61 bytes struct data + ~40 bytes `VersionedLogItem` overhead) with 3 indexed fields each.

### Retention policy impact

| Retention | Standard events | Maximum event size |
| --- | --- | --- |
| Indefinite | ~1.4 TB/year, growing linearly | ~365 GB/year, growing linearly |
| 90 days | ~351 GB | ~90 GB |
| 30 days | ~117 GB | ~30 GB |
| 7 days | ~27 GB | ~7 GB |

Events are deterministically reproducible by replaying chain history (see [Event Lifetime](#event-lifetime)), so short retention does not lose data — it moves the cost of historical access from the indexer to the consumer.

Sustained saturation is unlikely: events share the write budget with ledger state writes, and the fee model makes it expensive.

## Indexer Latency

Contract log events flow through the same indexer pipeline as existing event types (Zswap, Dust).
The pipeline itself is unchanged, but the additional data volume will increase storage and query load on the indexer.
The magnitude depends on event adoption — contracts without `log` produce no additional load.

# Appendix C. Zerocash Example

The `spend` circuit from the zerocash example contract (`examples/zerocash.compact`) illustrates the before/after.

## Before

```compact
ledger nullifiers: Set<nullifier>;
ledger commitments: HistoricMerkleTree<32, commitment>;
ledger ciphertexts: Opaque<"Uint8Array">;

circuit spend(dest_public_key: public_key, input_coin: coin_info): [] {
  const source_secret_key = private$zk_secret_key();
  const old_nullifier = derive_nullifier(input_coin, source_secret_key);
  assert(!nullifiers.member(old_nullifier), "spend: Coin already spent");
  nullifiers.insert(old_nullifier);
  const source_public_key = derive_zk_public_key(source_secret_key);
  const old_commitment = commitment_from_coin_info(input_coin, source_public_key);
  const commitment_path = context$path_of(old_commitment);
  assert(commitments.checkRoot(disclose(merkleTreePathRoot<32, commitment>(commitment_path))) &&
         old_commitment == commitment_path.leaf,
         "spend: Illegal state: merkle path not recognized by public state");
  const fresh_coin_info = context$new_coin_info();
  const fresh_commitment = commitment_from_coin_info(fresh_coin_info, dest_public_key.zk);
  commitments.insert(fresh_commitment);
  const ciphertext = disclose(context$encrypt(dest_public_key.encryption, fresh_coin_info));
  ciphertexts.write(ciphertext);
  private$remove_coin(input_coin);
}
```

The values already public on-chain:
- **`old_nullifier`** — disclosed hash, inserted into the public `nullifiers` Set.
- **`fresh_commitment`** — disclosed hash, inserted into the public `commitments` MerkleTree.
- **`ciphertext`** — encrypted coin info, written to the public `ciphertexts` ledger field.

To detect spends, a consumer must poll the full state, diff Merklized structures, and trial-decrypt every ciphertext.

## After

```compact
circuit spend(dest_public_key: public_key, input_coin: coin_info): [] {
  // ... existing logic unchanged ...

  log(ShieldedSpend { nullifier: disclose(old_nullifier) });
}
```

`old_nullifier` is already disclosed.
The `log` adds no new information — the data is already available on-chain.
The difference is that the contract is now explicit about which state changes are important, and consumers can discover them without knowledge of the contract's internal state layout.
