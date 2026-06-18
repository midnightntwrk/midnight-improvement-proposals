---
MIP: X
Title: Lazy Contract State Query RPC
Authors: Rodrigo Quelhas @RomarQ
Status: Draft
Category: Standards
Created: 2026-06-03
Requires: none
Replaces: none
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

This MIP defines `midnight_queryContractState`, a JSON-RPC method for reading specific fields of a deployed contract's state via paths through its `StateValue` tree. It exposes the lazy navigation primitive midnight-ledger already supports, so clients can read single leaves instead of fetching the full serialized state on every call.

## Motivation

A Compact contract's state is stored in midnight-ledger as a content-addressed DAG of `StateValue` nodes: `Array`, `Map`, `Cell`, and `BoundedMerkleTree` variants connected by `Sp<...>` pointers. Sub-trees can be loaded from ParityDB independently of the others, and this is the same structure the on-chain VM walks every time a circuit executes an `idx` instruction.

Today's `midnight_contractState` RPC flattens this structure on every read: it loads the entire DAG, tagged-serializes it, and returns the full blob. dApps typically want a subset of fields at a time, not the whole tree. For contracts with non-trivial state this is also a denial-of-service surface: each read forces the node to serialize and transport the entire state.

Measurements from [midnightntwrk/midnight-node#1069](https://github.com/midnightntwrk/midnight-node/issues/1069) (the originating feature request, devnet, `Map<Field, Bytes<32>>` contract):

| Map entries | RPC time | Response size |
|---|---|---|
| 0 | 5 ms | 2.9 KB |
| 5,000 | 145 ms | 373 KB |
| 10,000 | 672 ms | 905 KB |
| 17,000 | 2,505 ms | 1,641 KB |

## Specification

### Method

| | |
| --- | --- |
| Name | `midnight_queryContractState` |
| Params | positional, see below |
| Result | array of [`RpcStateQueryResult`](#rpcstatequeryresult) |

Parameters:

1. `contract_address`: string. Lowercase hex of a 32-byte `ContractAddress`, **without** `0x` prefix.
2. `queries`: array of [`RpcStateQuery`](#rpcstatequery) objects. The array MUST contain at most `MAX_STATE_QUERIES` entries (see [Limits](#limits)).
3. `at`: optional block hash (`0x`-prefixed hex). When absent or `null`, the server uses the best block.

### `RpcStateQuery`

```json
{ "path": ["0x...", "0x...", "..."] }
```

`path` is an ordered list of `PathKey`s, each a wire encoding of a Midnight `AlignedValue`.

An `AlignedValue` is a pair `(Value, Alignment)`: the `Value` is a sequence of byte-vector atoms, and the `Alignment` describes how to interpret them as a typed value. The common alignments are:

| Rust / Compact type             | Alignment   |
| ---                             | ---         |
| `bool` / `Boolean`              | `Bytes(1)`  |
| `u8` / `Uint<8>`                | `Bytes(1)`  |
| `u16` / `Uint<16>`              | `Bytes(2)`  |
| `u32` / `Uint<32>`              | `Bytes(4)`  |
| `u64` / `Uint<64>` / `Counter`  | `Bytes(8)`  |
| `u128` / `Uint<128>`            | `Bytes(16)` |
| `[u8; N]` / `Bytes<N>`          | `Bytes(N)`  |
| `Fr`                            | `Field`     |
| tuples / structs                | concatenated alignments |

Both halves are part of the encoded form, so a `PathKey` constructed from `u8(0)` is **not** byte-equal to one constructed from `u64(0)`. For `Map` keys this distinction is significant: the server's lookup compares the deserialized `AlignedValue` (alignment included) against the stored key.

For `Array` and `BoundedMerkleTree` indices the alignment is collapsed to a numeric position, but the widths differ:

- `Array` positions are decoded as a `u8`, matching the VM's `idx` instruction. An `Array` is capped at 16 entries by ledger invariant (`onchain-state/src/state.rs`), so 8 bits suffice. Integer keys of any width whose numeric value fits in `u8` are accepted; a value that does not fit (e.g. `u16(300)`) yields a "key did not decode as a `u8`" per-query error distinct from "index out of bounds."
- `BoundedMerkleTree` positions are decoded as a `u64` against `1 << tree.height()`. Out-of-range positions yield a per-query error.

The wire form of a `PathKey` is:

- `"0x" + hex(Serializable::serialize(av))`, lowercase hex, even length, with the leading `0x` prefix REQUIRED.
- `Serializable::serialize` is the untagged binary form defined in `midnight-serialize` (`serialize/src/serializable.rs`). The companion `tagged_serialize` prepends a `midnight:<tag>:` prefix (e.g. `midnight:fab-aligned-value[v1]:`); this method intentionally omits it because the type is fixed by the method name (see [Why untagged wire format?](#why-untagged-wire-format)).
- The hex-decoded byte length of each `PathKey` MUST NOT exceed `MAX_KEY_BYTES` (see [Limits](#limits)).
- A `path` array MUST NOT be empty and MUST contain at most `MAX_PATH_DEPTH` entries (see [Limits](#limits)).

### `RpcStateQueryResult`

```json
{
  "query": { "path": [...] },
  "value": "0x...",      // present on success
  "error": "string"      // present on per-query failure
}
```

Exactly one of `value` and `error` MUST be set on the wire. `value`, when present, is `"0x" + hex(Serializable::serialize(resolved_state_value))`. `query` echoes the input query verbatim so callers can correlate responses without external state.

### Navigation semantics

The server starts at the contract's `StateValue` root (call it `current`). For each `PathKey` in the path, in order:

1. Let `key` be the deserialized `AlignedValue` for this step.
2. Branch on `current`'s variant:
   - **`Array(arr)`**: `key` MUST decode as an integer fitting in `u8` (matching the VM's `idx` instruction). Two distinct per-query errors are possible:
     - "key did not decode as a `u8`": the alignment is non-integer, or the numeric value exceeds `u8::MAX` (e.g. `u16(300)`).
     - "index out of bounds": the decoded `i` satisfies `i >= arr.len()`.

     Otherwise set `current = arr[i]`.
   - **`Map(map)`**: if `map` contains `key`, set `current = map[key]`. **If `map` does not contain `key`, set `current = StateValue::Null` and continue with the remaining steps.** This mirrors the VM `idx` instruction's behaviour: a map miss substitutes `Null` and the path continues. Because `Null` is not an indexable variant, any subsequent step necessarily yields a "step cannot be indexed" per-query error, which is the right signal to the caller.
   - **`BoundedMerkleTree(tree)`**: `key` MUST decode as a `u64` position `p`. Compute `max = 1u64 << tree.height()` (the `0 < height <= 32` ledger invariant guarantees this fits in `u64`; implementations SHOULD use `checked_shl` defensively in case of malformed state). If `p >= max`, yield a per-query error. Otherwise look up the leaf at `p`: if present, set `current` to a `Cell` containing the leaf hash; if the slot is empty, set `current = StateValue::Null` and continue (same rationale as the `Map` miss case).
   - **`Cell(_)` or `Null`** (the leaf variants): yield a per-query error "step cannot be indexed" — there is nothing to navigate into.

After all steps:

- If `current` is `Array`, `Map`, or `BoundedMerkleTree`, yield a per-query error: "path resolves to a collection; provide a deeper path." The intent of this method is to return concrete field values; allowing a collection return would let a single query inflate the response to an unbounded size.
- Otherwise return `value = "0x" + hex(Serializable::serialize(current))`.

#### Example: reading a counter `Cell`

A typical Compact contract's state is an `Array` at the top level whose entries are the ledger's fields, indexed in declaration order. Take a contract that declares three fields:

```
Array(3)
  ├── [0] Cell(AlignedValue: u8)     ← version    (Uint<8>)
  ├── [1] Map<Fr, Cell>              ← balances   (Map<Field, Uint<64>>)
  └── [2] Cell(AlignedValue: u64)    ← counter    (Counter)
```

To read `counter`, the client encodes the field index as `AlignedValue::from(2u8)` and submits:

```json
{
  "method": "midnight_queryContractState",
  "params": [
    "<contract_address_hex>",
    [
      { "path": ["0x<untagged hex of AlignedValue::from(2u8)>"] }
    ]
  ]
}
```

Server-side traversal:

| step | `current` before        | key   | `current` after          |
| ---  | ---                     | ---   | ---                      |
| 1    | `Array(3) [...]`        | `u8(2)` | `Cell(AlignedValue: u64)` |
| end  | `Cell` (leaf)           |       | return `Serializable::serialize(Cell)` |

Response:

```json
{
  "result": [
    {
      "query": { "path": [...] },
      "value": "0x<untagged hex of Cell(AlignedValue: u64)>"
    }
  ]
}
```

The client deserializes `value` back into a `StateValue` via the untagged `Deserializable::deserialize` routine matching the server's `Serializable::serialize`.

Reading an entry from `balances` is a two-step path `[u8(1), Fr(<map_key>)]`: the first step navigates the top-level `Array` to the `Map` at field index 1, the second looks up the requested key inside the `Map`.

### Limits

Implementations MUST enforce each of the protective caps below. The specific values are suggested defaults and may be tightened or loosened by an implementation, but each cap MUST exist.

| Limit                | Default | Scope          |
| ---                  | ---     | ---            |
| `MAX_STATE_QUERIES`  | 100     | per request    |
| `MAX_PATH_DEPTH`     | 16      | per query      |
| `MAX_KEY_BYTES`      | 512     | per path key   |

The defaults are not arbitrary, but only `MAX_KEY_BYTES` is grounded in a structural ledger bound:

- `MAX_PATH_DEPTH = 16` is a conservative policy cap, not a structural one: the ledger imposes no maximum nesting depth, so a contract could in principle build `Map<_, Map<_, ...>>` chains beyond 16 levels. The default is set well above any realistic Compact field nesting (typically two to three levels: top-level `Array` of fields with `Map`/`Array` inside).
- `MAX_KEY_BYTES = 512` accommodates typical Compact key widths (e.g. `HashOutput` or `Fr` at 32 bytes plus alignment overhead, common tuple keys) with comfortable headroom.

Aggregate response size is bounded by the JSON-RPC server's own maximum response size, which `jsonrpsee` and substrate's RPC layer already enforce; the MIP does not redefine it.

### Error model

Top-level JSON-RPC errors (the whole batch fails):

| Cause | Error |
| --- | --- |
| `contract_address` not valid lowercase hex of 32 bytes | `BadContractAddress` |
| Well-formed address with no contract deployed | `ContractNotPresent` |
| `queries.len() > MAX_STATE_QUERIES` | `TooManyQueries` |
| Malformed `PathKey` (bad hex, missing `0x`, oversize, AlignedValue decode failure) | JSON-RPC deserialization error |
| Internal node failure (runtime API unavailable, ledger storage unreachable) | `UnableToGetContractState` |
| Unknown or pruned `at` block hash | `UnableToGetContractState` (matches the existing `midnight_contractState` behavior; a distinct error variant is a possible follow-up) |

Per-query failures (the conditions defined in [Navigation semantics](#navigation-semantics)) are reported in `RpcStateQueryResult.error` as a human-readable string. The exact wording is non-normative.

## Rationale

### Why untagged wire format?

`midnight-serialize` supports both tagged and untagged binary forms. The tagged form (`tagged_serialize`) prepends `midnight:<tag>:`, a single token whose `<tag>` carries the type name and a version discriminator: `AlignedValue` is currently `fab-aligned-value[v1]`, so its tagged prefix is `midnight:fab-aligned-value[v1]:`. `StateValue` is `impact-state-value[v2]`. The untagged form (`Serializable::serialize`) omits the prefix entirely.

This method uses the untagged form for two reasons:

1. **Compactness.** The tag is constant per path key and per result value, so transmitting it on every entry of every batch is dead weight. For a maximum batch (`MAX_STATE_QUERIES × MAX_PATH_DEPTH` keys), dropping the prefix saves on the order of tens of kilobytes.
2. **The type is fixed by the method name.** A `midnight_queryContractState` path key is always an `AlignedValue`; a result value is always a `Cell` or `Null`. There is no parser-side ambiguity for the tag to resolve.

The trade-off is that the wire bytes carry no `[vN]` discriminator. Clients are expected to know the on-chain ledger version they are talking to (discoverable via `midnight_apiVersions` and `get_ledger_version`). An incompatible `AlignedValue` layout change would be introduced under a new method name or a new top-level API version, not silently within this method.

## Path to Active

### Acceptance Criteria

The MIP transitions to Active once the following are met:

- The `midnight_queryContractState` RPC implementation is merged into `midnight-node`;
- The method is registered in the node's OpenRPC document for the corresponding release, so SDKs and tooling can discover and call it.

### Implementation Plan

- `midnight-node`: ship the reference implementation of the JSON-RPC method, register it in the node's OpenRPC document, and add end-to-end coverage.

## Backwards Compatibility Assessment

This MIP adds a new JSON-RPC method and does not modify any existing one. No on-chain, transaction-format, or consensus change.

## Security Considerations

This RPC is a strict reduction of the over-fetch surface introduced by `midnight_contractState`: instead of returning the full serialized contract state on every call, it returns only the leaves the caller asks for.

Work per request is bounded to `O(MAX_STATE_QUERIES × MAX_PATH_DEPTH)` storage lookups. The per-leaf response size is bounded by an existing ledger invariant: `CELL_BOUND = 32 KiB` (`onchain-state/src/state.rs`) is the maximum serialized size of any `Cell`, and Navigation rejects intermediate collections, so every successful result is a `Cell` or `Null`. The aggregate response size (`MAX_STATE_QUERIES × CELL_BOUND = 3.125 MiB` worst case) is bounded by the JSON-RPC server's own maximum response size, which `jsonrpsee` and substrate's RPC layer already enforce.

## Implementation

The reference implementation in `midnightntwrk/midnight-node` consists of:

- A JSON-RPC method on the `MidnightApi` trait (`pallets/midnight/rpc/src/lib.rs`).
- A wrapped wire type `PathKey(AlignedValue)` with serde impls that round-trip through untagged hex.
- A bridge function that walks the contract's `StateValue` tree lazily against ParityDB (`ledger/src/versions/common/mod.rs`).
- OpenRPC entries and JSON schemas (`node/src/openrpc.rs`, regenerated `docs/openrpc.json`).

## Testing

The reference implementation's end-to-end tests cover:

- Successful path navigation to a `Cell` leaf, with the result deserialized client-side and equality-checked against the expected `StateValue`.
- A batched call that exercises all three result kinds in one round-trip: a `Cell` value, a `Map` hit returning `Null`, and an array out-of-bounds (per-query error).
- A top-level error path for a well-formed but never-deployed contract address (`ContractNotPresent`).

Additional reference tests verify the OpenRPC document is in sync with the runtime-registered method set and that the static OpenRPC JSON does not drift from `build_openrpc_document`.

## Acknowledgements

To be filled in when reviewers/co-authors are confirmed.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
