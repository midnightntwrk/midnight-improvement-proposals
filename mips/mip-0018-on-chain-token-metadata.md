---
MIP: "0018"
Title: On-Chain Token Metadata Emission (`TokenMetadata` Events)
Authors:
  - Edward Alvarado <edward.alvarado@midnight.foundation>
  - Sebastien Guillemot (@SebastienGllmt) <sebastien.guillemot@midnight.foundation>
Status: Proposed
Category: Standards
Created: 2026-09-17
Requires: "MIP-0002 Public Contract Log Emission for Compact"
Replaces: none
MPS: "Off-Chain Token Metadata Registry for Midnight (unnumbered; https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)"
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

Midnight wallets and explorers cannot tell what a token is.
A native UTXO carries a 32-byte color that identifies its token but no name, and a ledger token is whatever its contract implements, with no generic way to find or label it.

This MIP lets a contract publish metadata for its own tokens. It defines two parts:

1. **A transport layer.** Metadata travels in [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` events named `mip-0018:token-metadata[v1]`. Each event names one token and carries one or more typed key/value records. Each event is bound to the contract that emitted it, so a contract can describe only its own tokens. A user who has the color of a native UTXO, or a ledger token's contract address, can use it to query that token's authoritative events. Consumers keep the latest value for each key; a Null record withdraws a token's metadata.
2. **Common metadata.** Three common keys (`name`, `symbol`, `decimals`) and an optional `standards` key give wallets and explorers a shared core for displaying tokens.

The transport accepts any other key and is designed as a building block for future standards: later MIPs can define their own metadata on it without changing the transport.
Events are emitted when a token is created and for extraordinary updates, never as part of normal token operation.

## Motivation

### The problem

Every native UTXO has a color, `tokenType(domainSep, contractAddress)`, that identifies its token.
Mint effects are public in the transaction transcript, so anyone can list every color and the contract that minted it, but nothing says what that token is called or how to display it.

Ledger tokens are not visible even at that level.
They have no mint effect, and each contract represents them its own way: public balances, encrypted balances, coins or commitments held by users, a mix of these, or anything else a contract can implement.
A scanner that does not know the contract cannot tell that a token exists.

[MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md), [MIP-0011](./mip-0011-native-shielded-token.md) and [MIP-0014](./mip-0014-native-unshielded-token.md) define `name()`, `symbol()` and `decimals()` circuits, but reading metadata through them does not work for generic clients:

- It requires executing contract code against the ledger.
- There is no direct way for an indexer to learn that a value has changed.
- As of October 2026, there is no way to obtain a contract's executable interface.

With events, a client needs only network calls to an indexer.
This MIP does not replace circuit interfaces: it is expected to be one of the building blocks, alongside standards built on circuit interfaces such as these getters.

The consequences are those the Token Registry MPS lists: wallets show 64-character hex strings, explorers cannot label tokens, and DApps hard-code token lists.

### Why an off-chain registry is not enough on its own

The Token Registry MPS proposes a CIP-26-style off-chain registry.
That is the right place for curation, but it cannot be the only layer:

- **Provenance.** A registry entry claims to speak for an issuer, but nothing on chain identifies one. A `ContractDeploy` records no creator; the maintenance authority controls upgrades rather than naming the deployer, and may be empty; whoever paid the DUST fee proves nothing. Only the contract itself, by executing, can speak for the contract. An emitted event is exactly that.
- **Registration.** Every issuer must find and submit to the registry. Test tokens, community tokens and LP shares never will.
- **Discovery.** A registry maps the colors someone registered; the event stream contains every declaration.

### Why events

- **Self-describing.** The event carries the key and the value in a layout fixed by this MIP, so decoding needs no knowledge of the contract.
- **Bound to the emitter.** Each event is bound to the contract that emitted it; its address comes from the event record, not from the payload.
- **Append-only.** Renames and corrections are new events, applied in chain order.
- **No state growth.** Events are not consensus state ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Event Lifetime").
- **Available now.** `emit` → `Log` opcode → `VersionedLogItem` → indexer `MiscContractEvent` is shipping; this MIP is only a convention on `Misc`.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Terminology

- **Color**: the 32-byte token type carried by every native UTXO, `tokenType(domainSep, contractAddress)` from the Compact standard library. All UTXOs of one native token have the same color.
- **`domainSep`**: 32 bytes identifying a token within its contract, like an ERC-1155 `id`. For a native token it is the value passed to `mintShieldedToken` or `mintUnshieldedToken`. For a ledger token it is a custom identifier the contract chooses for that token.
- **Native token**: value held as protocol-level UTXOs (shielded Zswap coins or unshielded UTXOs) and minted by a contract. Its UTXOs carry a color.
- **Ledger token**: a token represented by contract logic rather than protocol UTXOs. The representation is up to the contract (public balances as in [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md), encrypted balances, user-held coins or commitments, or a mix), and this MIP does not depend on it. It uses no color and has no mint effect.
- **Token identity**: `(network, contractAddress, domainSep, kind)`; `kind` is defined in [Token identity and authority](#token-identity-and-authority).
- **Record**: one typed key/value entry in an event.
- **Field**: a token identity plus a key. A field has at most one current value.
- **Tombstone**: a record of type Null. It withdraws all metadata of its token identity.
- **Consumer**: any reader of these events, such as an indexer, wallet or explorer.

How token identity, field and record relate:

```
|<-------------- token identity -------------->|
|<---------------------- field ----------------------->|
                                               |<--- record ---->|
| network | contractAddress | domainSep | kind |  key  |  value  |
```

`network` is the network the event was read from, `contractAddress` comes from the event record, `domainSep` and `kind` come from the payload header, and `key` and `value` come from a record in the payload.

### Event

A `TokenMetadata` event is a [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` event:

```
Misc {
  name:    pad(32, "mip-0018:token-metadata[v1]")
           // 0x6d69702d303031383a746f6b656e2d6d657461646174615b76315d, then 5 zero bytes
  payload: Bytes<256>   // see Payload
}
```

Consumers MUST ignore `Misc` events with any other name, including other versions of this one.
An event with this name whose payload cannot be decoded as defined in this MIP is invalid: consumers MUST NOT consider it, and none of its records are applied.

**Versioning.** The name carries the version.
New keys and new `standards` identifiers need no new version.
A new value type or `kind`, or a change to the payload format, the validation rules or the common-field meanings, requires a new name such as `mip-0018:token-metadata[v2]`.

### Payload

Each payload is 256 bytes and describes one token identity. It has three parts, in order: a header, one or more records, and zero padding.

```
| domainSep (32 bytes) | kind (1 byte) | record 1 | record 2 | ... | zero padding |
0                      32              33                                       256
```

**Header** (the first 33 bytes):

| Offset | Size (bytes) | Field | Meaning |
|---:|---:|---|---|
| 0 | 32 | `domainSep` | Which token within the contract. |
| 32 | 1 | `kind` | How the token is represented: 1, 2 or 3; see [Token identity and authority](#token-identity-and-authority). |

**Records** start at offset 33 and follow each other with no gaps. Each record has five fields:

| Field | Size (bytes) | Meaning |
|---|---:|---|
| `keyLen` | 1 | Length of `key`, 1–255. |
| `key` | `keyLen` | The key, for example `name`. |
| `valType` | 1 | How to read `value`; see [Value types](#value-types). |
| `valLen` | 1 | Length of `value`, 0–255. |
| `value` | `valLen` | The value. |

For example, `name = "Acme Token"` is the 17-byte record `04 6e616d65 01 0a 41636d6520546f6b656e`: `keyLen` 4, `key` "name", `valType` 1 (UTF-8 string), `valLen` 10, `value` "Acme Token". [Appendix A](#appendix-a-example-event-informative) shows a complete payload.

**Padding:** every byte after the last record, up to byte 256, is zero. A zero `keyLen` therefore marks the end of the records, which is why a key cannot be empty.

- Every record must fit within the 256 bytes; see [Limitations](#limitations).
- Keys and values are exact byte strings. Lengths count bytes, not characters, and zero bytes inside a key or value are significant.

A consumer MUST check a payload as follows.
If any check fails, it MUST reject the whole event and apply none of its records.
Rejecting one event never affects another.

1. `kind` (byte 32) is 1, 2 or 3.
2. Starting at byte 33, read records one after another until reaching either byte 256 or a zero `keyLen`:
   - If `keyLen` is zero, the records have ended, and every remaining byte up to byte 256 is zero.
   - Otherwise the key, `valType`, `valLen` and the value all fit within the 256 bytes, and the value follows the rule for its `valType` (see [Value types](#value-types)). The next record starts right after the value.
3. At least one record was read.

Every emitter MUST produce exactly these bytes, whatever its toolchain.
Compact has no runtime-sized byte strings, so a Compact emitter builds each payload from fixed-size fields (for example `Uint<8>` lengths and `Bytes<K>` keys), using the serialization the Compact compiler provides for those types.

### Value types

| `valType` | Type | Rule |
|---:|---|---|
| 0 | bytes | Any length. |
| 1 | UTF-8 string | Valid UTF-8; may be empty. |
| 2 | unsigned integer | `valLen` 1–31; little-endian, the Compact serialization of `Uint<8 × valLen>`. |
| 3 | JSON | One complete UTF-8 JSON value ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259.html)). |
| 4 | URI | A UTF-8 absolute URI ([RFC 3986](https://www.rfc-editor.org/rfc/rfc3986)). |
| 5 | Null | `valLen` = 0. A tombstone; see [Applying records](#applying-records). |
| 6–255 | reserved | Reject the event. |

A value that breaks its type's rule rejects the event.
Consumers MUST decode integers of every permitted width and MUST NOT reinterpret a value as a different type.
Empty strings, empty bytes, zero bytes and JSON `null` are ordinary values, not tombstones.

### Token identity and authority

`kind` says how the token is represented:

| `kind` | Representation | Example | Color |
|---:|---|---|---|
| 1 | native shielded | MIP-0011 | yes |
| 2 | native unshielded | MIP-0014 | yes |
| 3 | ledger | MIP-0004 | no |

Any other value rejects the event.

Every record in an event belongs to the token identity `(network, contractAddress, domainSep, kind)`:

- `network` is the network the event was read from.
- `contractAddress` is the address of the contract the event is bound to. Consumers MUST take it from the event record, never from the payload, so a contract can describe only its own tokens.
- Each kind is a separate identity with its own metadata events, even when several represent the same asset. A contract may mint one asset both shielded and unshielded and also represent it as a ledger token (as MIP-0004 does), and a user may hold all three at once. The contract describes each representation separately; indexers and UIs combine them, for example through [Symbol grouping](#symbol-grouping).
- For a ledger token, `domainSep` is any stable 32 bytes the contract chooses, such as `pad(32, "acme:gold")`.

**Lookup.** A user finds a token's metadata from what the user holds: the color of a native UTXO (kinds 1 and 2) or a ledger token's contract address (kind 3).
For native tokens, indexers record the contract address and `domainSep` of every mint in each block, compute `color = tokenType(domainSep, contractAddress)`, and keep a table from color to `(contractAddress, domainSep)`.
A color held by a user resolves through that table to a token identity (kind 1 for a shielded coin, kind 2 for an unshielded UTXO), whose metadata events the user then queries.
A color MUST always be computed this way, never read from a value.
For kind 3, querying the contract's kind-3 events lists its ledger tokens. This MIP does not define how a client picks the user's token among several; that is left to the standards the token declares in `standards`.

### Keys

- Keys are compared as exact bytes: case-sensitive, with no trimming or Unicode normalization.
- Keys SHOULD be UTF-8, but consumers MUST NOT reject an event because a key is not UTF-8.
- Any key is allowed. This MIP gives meaning only to the keys in [Common fields](#common-fields); later MIPs can define others.

### Applying records

Consumers apply records from accepted events in chain order: block, transaction within the block, event within the transaction, then record within the event.
For every token it tracks, a consumer's state MUST equal the result of applying, in that order, every accepted event on the canonical chain.
A consumer that follows non-final blocks MUST therefore recompute state when a reorganization removes blocks; alternatively it can follow only finalized blocks.

- **Non-Null record:** sets that field's current value, replacing any earlier value. An event changes only the keys it carries; it is not a snapshot. A replaced value MUST NOT be presented as current or used as a fallback; consumers MAY keep it as clearly marked history.
- **Null record (tombstone):** withdraws the whole token identity, whatever its key. The consumer MUST hide the identity, clear all its fields, and stop serving its earlier values as metadata or metadata history. A repeated tombstone has no effect. The next non-Null record makes the identity visible again with only that field set; nothing from before the tombstone returns. There is no per-key delete; a single key is changed by overwriting it.

### Common fields

For each token identity it describes, a contract SHOULD publish the three common keys `name`, `symbol` and `decimals`, and MAY publish the optional key `standards`.

| Key | Value type | Requirement | Meaning |
|---|---|---|---|
| `name` | UTF-8 string (1), not empty | SHOULD | Display name. |
| `symbol` | UTF-8 string (1), not empty | SHOULD | Ticker. |
| `decimals` | unsigned integer (2) | SHOULD | Number of decimal places: 10^`decimals` base units make one whole token, so a raw amount is shown as `amount / 10^decimals`. Emitters SHOULD use `Uint<8>`, the type MIP-0011 and MIP-0014 use. |
| `standards` | UTF-8 string (1) | MAY | Standards the token claims to implement; see below. |

- A token that also exposes MIP-0004, MIP-0011 or MIP-0014 getters SHOULD emit the same values those getters return.
- A field whose current value lacks the type or form above is **unusable**: consumers show no value for it and MUST NOT fall back to an earlier value. The event that set it remains valid.
- If `name`, `symbol` or `decimals` was never set, there is no value. Consumers MUST NOT assume a default, such as 0 or 18 decimals.
- `standards` is a list of identifiers separated by single spaces (`0x20`). An identifier is non-empty and contains no spaces or control characters (no byte in `0x00`–`0x20` or `0x7f`). Identifiers are compared exactly and are case-sensitive; order and duplicates carry no meaning. An empty value, or no `standards` field, means no standards are claimed; a malformed value is unusable, not empty.
- This MIP defines only the list format, not what an identifier means or what claiming it implies. A MIP is identified as `mip-NNNN` (for example `mip-0011`), and that MIP defines its meaning. A token may also claim standards from elsewhere, such as BIPs or ERCs; their identifiers and what they mean on Midnight SHOULD be defined in a MIP.
- `standards` is self-declared. A consumer MAY use an identifier it recognizes to choose a UI or adapter it already trusts, but MUST NOT treat it as proof of conformance and MUST NOT fetch or run code because of it.

### Symbol grouping

Indexers SHOULD group visible token identities that share `(network, contractAddress)` and have the same usable `symbol`, compared as exact bytes.

- A group never spans contracts or networks.
- Grouping is presentation only. Each identity keeps its own fields, and an update to one member changes no other member.
- An identity with no usable `symbol` is ungrouped. Changing an identity's `symbol` moves only that identity; a tombstone removes it from its group.

### Publishing

- **No events in normal operation.** Normal token operation, such as mints, transfers and burns, MUST NOT emit metadata events. A contract emits them only when a token is created (or first described, for an existing token) and for extraordinary updates, such as a rename.
- A Compact constructor cannot emit. A contract that wants metadata at deployment exposes a circuit, conventionally `publishMetadata()`, which the deployer calls right after deployment.
- Who may call an emitting circuit is the contract's choice. Anyone who can call it can rename or withdraw the token, so it SHOULD be access-controlled or publish-once.
- Records for one token identity MAY be emitted in different events, but SHOULD be grouped into as few events as the size limit allows ([Limitations](#limitations)).
- The producer is responsible for making sure the transaction that emits the event is included and executed in a block.
- Everything emitted is public.

### Consuming

- **Reading events.** How consumers obtain events is defined by [MIP-0002](./mip-0002-public-contract-log-emission.md).
- **Completeness.** Verifying the events a service returned does not prove that no later update or tombstone exists, and a field missing from a filtered response is not proof that it was never set. Indexers MAY index only some tokens or keys.
- **Untrusted input.** Every payload byte is attacker-controlled. Consumers MUST bounds-check every length before slicing and MUST harden their UTF-8, JSON and URI parsers. Consumers MUST NOT fetch a URI from a value without the precautions they apply to any remote content.

### Limitations

- The maximum size of a value is 219 bytes: the 256-byte payload minus the 33-byte header, the 3 bytes of `keyLen`, `valType` and `valLen`, and a 1-byte key. In general, the largest value is `220 − keyLen` bytes.
- Each event describes one token identity; metadata for another identity needs another event.

### Off-chain content

Content that does not fit in an event, such as an image or a document, stays off chain.
If off-chain content is used, the MIP that defines it SHOULD include a way to validate that content, such as a URL together with a hash of the content.

### Out of scope

- **Schemas beyond the common fields**: real-world-asset (RWA) fields, media, localization, privacy properties and cross-chain mappings. Later MIPs can define them as keys on this transport.
- **Data that changes during normal operation**, such as supply, prices or per-item data. It belongs off chain; see [Publishing](#publishing).
- **Curation**: deciding which of several tokens with the same name or symbol is genuine. That belongs to registries and allowlists, which can be seeded from these events.
- **NIGHT and DUST**: their properties are fixed by the protocol.
- **Private metadata**: every event is public. Selective disclosure belongs to later event work ([MPS-0005](../mps/mps-0005-events.md)).

## Rationale

- **`Misc` rather than a new event type.** A dedicated `LogEventType` would need a ledger release and a coordinated node upgrade, and would freeze the format in the protocol. `Misc` works today, the versioned name gives a way to evolve, and the convention can be promoted to a dedicated type later.
- **`domainSep` rather than color.** The color is derivable from `domainSep` and the emitting contract's address. Sending it would add 32 redundant bytes a contract could forge; deriving it means a contract can never claim another contract's color. `domainSep` also covers ledger tokens, which use no color.
- **A `kind` byte.** One asset can exist as shielded, unshielded and ledger tokens at once, and each representation needs its own metadata and lifecycle; combining them is left to indexers and UIs. Ledger tokens get a single kind because their representation is up to the contract and can change without creating a different asset, so it is not a sound identity attribute.
- **Grouping by symbol within one contract.** One contract can represent an asset in several ways (shielded, unshielded, ledger, or under several `domainSep` values), and a wallet should be able to show them together. Limiting a group to one contract stops another contract from joining it by copying the symbol.
- **Typed values.** [EIP-7496](https://eips.ethereum.org/EIPS/eip-7496) stores trait values as untyped `bytes32` and describes their types off chain, which helps only consumers that already know the collection. One type byte lets a generic explorer decode a key it has never seen.
- **Variable-length packed records.** The `Misc` payload is fixed at 256 bytes. One-byte lengths suffice within that limit, variable-length keys avoid spending 32 bytes on every key, and packing fits the three common fields plus `standards` in one event (95 bytes in [Appendix A](#appendix-a-example-event-informative)).
- **Latest value per key.** As in EIP-7496, an update touches one key and costs one record, with no need to re-emit everything.
- **No events in normal operation.** Events are not consensus state, but each one still costs fees, block space and storage in every indexer. A token can see thousands of mints and transfers; emitting metadata with them would create thousands of events for values that consumers reduce to one current value per key. The chain is not the place for that data; it belongs off chain, referenced as described in [Off-chain content](#off-chain-content) if needed.
- **Identity-wide tombstone.** A single key can be changed by overwriting it. Null exists for the one operation overwriting cannot express: retracting a token's metadata entirely.
- **Events rather than a metadata field in state.** As of October 2026, there is no way to read or prove what a value in a contract's ledger state means; an event carries the key and value in one fixed layout defined by this MIP. State also grows every node's storage and keeps no history.
- **On-chain names despite MIP-0014.** MIP-0014 rejects on-chain `name`/`symbol` as inviting impersonation without removing the color-derivation check. Impersonation is just as easy in an off-chain registry. What this MIP adds is that every claim is bound to the contract that made it, and the derivation check is automatic because a consumer only ever derives colors. Which token to trust remains curation, which is out of scope.

**Alternatives considered.**

- *Circuit code only (reading metadata through `name()`, `symbol()` and `decimals()` getters)*: requires executing contract code against the ledger, gives indexers no direct way to learn that a value changed, and, as of October 2026, a contract's executable interface cannot be obtained. Kept as a complementary building block, alongside this MIP.
- *Off-chain registry only*: needs attestations and governance to recover the provenance an emitted event has by construction. Kept as the curation layer.
- *URI only (ERC-721 `tokenURI`)*: every field becomes a remote fetch that adds a liveness dependency, and a plain URL gives no way to check what was fetched. Off-chain content remains possible, with a way to validate it ([Off-chain content](#off-chain-content)).
- *JSON for every record*: costlier to emit and decode than typed bytes for simple values. Type 3 remains available.
- *A protocol-level event type*: deferred, as above.
- *[MIP-0019](./mip-0019-multipart-event.md) multipart events for longer values*: would let one payload span several events, but every consumer would then have to collect and join events before parsing them. Metadata values are short, and larger content stays off chain ([Off-chain content](#off-chain-content)).

## Path to Active

### Acceptance Criteria

- At least two independent consumers (indexer, explorer or wallet) pass every vector in [Testing](#testing).
- At least one issuer other than the authors adopts the convention on a public network.
- A reference Compact module, tested against the vectors, is published in this repository or an ecosystem library such as OpenZeppelin Compact Contracts.
- Community review through the MIP process, including agreement with the Token Registry MPS authors on the on-chain/off-chain boundary.

### Implementation Plan

1. Publish a Compact module and example issuers in [midnight-experiments/mip-0018](https://github.com/midnight-experiments/mip-0018).
2. Publish the test vectors as byte fixtures and a reference decoder, and obtain a second independent reader.
3. Deploy a publisher on a public test network.
4. Propose the module to OpenZeppelin Compact Contracts as an optional extension of `NativeShieldedToken`, `NativeShieldedTokenFamily` and `FungibleToken`.
5. Publish an upgrade template for pre-v9 contracts and exercise it on a public test network.

## Backwards Compatibility Assessment

No protocol, compiler or indexer change is required; this MIP uses MIP-0002's existing `Misc` event.
It adds to MIP-0004, MIP-0011 and MIP-0014 tokens without changing their circuits.
A contract that never emits is simply undescribed.

### Existing contracts

All contracts can emit once MIP-0002 is in effect (ledger v9, Midnight 2.x).
A contract deployed before then has no emitting circuit, but it does not need redeployment: its maintenance authority can add one.

1. Compile a `publishMetadata()` circuit (and optionally `setMetadata(...)`) against the contract's existing ledger layout. It reads the existing `domain` from state and emits it as `domainSep`.
2. Add its verifier key with a `VerifierKeyInsert` maintenance update signed by the maintenance authority.
3. Call it.

The contract address, `domainSep`, color and every existing coin and balance are unchanged.
The event still comes from the token's own contract, so [authority](#token-identity-and-authority) holds, and the update needs no permission beyond what the issuer already reserved.

## Security Considerations

- **Impersonation.** Any contract can emit any `name` or `symbol`, including one another token already uses. A record is bound to its contract and color, not to a brand. Consumers MUST keep each claim attached to its token identity and SHOULD make that identity visible to users. They SHOULD require a curation signal (allowlist, registry attestation or user confirmation) before presenting a token as a known asset. This matches EVM chains and unattested CIP-26 entries.
- **Curation is a separate layer.** These events are an issuer-written base layer. Deciding which of several tokens with the same name or symbol is genuine is curation, which this MIP does not attempt. A registry such as the one proposed in the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) can provide it on top of these events.
- **Shared symbols.** Anyone can copy a symbol. Because a group never spans contracts, copying one does not put a token into another contract's group. Consumers MUST NOT treat tokens from different contracts as the same asset because their `symbol` values match, and membership of a group within one contract does not prove its members are interchangeable.
- **Unauthorized updates.** An unguarded emitting circuit lets anyone rename or withdraw a token. Consumers MAY show a field's earlier values, marked as history, so hostile renames are visible. A tombstone also removes that history from metadata views, although the events remain on chain.
- **Self-declared standards.** Advertising an identifier in `standards` proves nothing; see [Common fields](#common-fields).
- **Untrusted payloads.** See [Consuming](#consuming).
- **Off-chain content.** A URL alone says nothing about what its host serves, and the host can change it at any time. A way to validate the content, such as a hash next to the URL ([Off-chain content](#off-chain-content)), lets consumers detect substituted content.
- **Spam and cost.** Events pay the existing `Log` fee and compete for each block's `bytes_written` budget ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Fee Metering"). Indexers MAY limit what they index.
- **Disclosure.** `emit` is a disclosure point checked by the compiler, so this MIP introduces no new leak path.

## Implementation

A publisher writes the header, its records and zero padding, and emits the payload under the event name.
A reader filters by event type and name, parses and validates the payload ([Payload](#payload)), applies records in order ([Applying records](#applying-records)) and evaluates the common fields ([Common fields](#common-fields)).
Issuers choose their `domainSep` values, kinds, extra keys and access control.

The reference implementation (libraries, examples and tests) lives in [midnight-experiments/mip-0018](https://github.com/midnight-experiments/mip-0018).

### Dependencies

- [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` events: Compact 0.34.0 / language 0.26.0 / runtime 0.19.0, Midnight ledger v9.

## Testing

These vectors are normative.
Unless stated otherwise, the header is `domainSep = 0x11` repeated 32 times and `kind = 3`.

**Must accept:**

- **A1. Common fields and `standards`.** `name = "Acme Token"`, `symbol = "ACME"`, `decimals = 6` as `Uint<8>` and `standards = "mip-0004"`. Records start at offsets 33, 50, 63 and 75; content ends at offset 95, followed by 161 zero bytes. The full bytes are in [Appendix A](#appendix-a-example-event-informative).
- **A2. Capacity.** One type-0 (bytes) record with a 220-byte key and an empty value, or with a 1-byte key and a 219-byte value, fills exactly 256 bytes with no padding.
- **A3. Exact bytes.** `symbol` and `symbol` followed by `0x00` are different keys. The type-0 value `01 00 00` keeps its trailing zeros. A non-UTF-8 key is accepted.
- **A4. Integer widths.** `decimals` as `06` (`Uint<8>`) and as `06` followed by 15 zero bytes (`Uint<128>`) both decode to 6.
- **A5. Non-tombstones.** An empty string (type 1), empty bytes (type 0) and JSON `null` (type 3) are ordinary values.

**Must reject the whole event, applying none of its records:**

- **R1.** A header with no records (bytes 33–255 all zero).
- **R2.** A key, type byte, length byte or value that extends past byte 256, such as a 221-byte key with an empty value, or a 1-byte key with a 220-byte value.
- **R3.** After a valid record, a zero `keyLen` followed by any non-zero byte.
- **R4.** `kind` 0, 4 or 255; `valType` 6 or higher.
- **R5.** Invalid UTF-8 in a type-1, type-3 or type-4 value; invalid JSON; a relative URI; an integer of 0 or 32 bytes; a Null record with `valLen` > 0.
- **R6.** A valid record, including a tombstone, followed by an invalid one.

**Must ignore:** `Misc` events with any other name, including `mip-0018:token-metadata[v2]`, and events of other types.

**State rules:**

- **S1. Order within an event.** For kind 1, records `name = "A"`, Null at key `retire` and `name = "B"` (at offsets 33, 41 and 50) leave the identity visible with only `name = "B"`. Without the Null, `name = "A"` followed by `name = "B"` in one event also leaves `name = "B"`.
- **S2. Latest value wins.** `name = "Alpha"`, `symbol = "ALP"`, `decimals = 2` and `standards = "mip-0004"` produce the same state whether sent as four events or one. A later event with only `name = "Beta"` changes only `name`; `Alpha` is never shown as current or used as a fallback.
- **S3. Tombstone.** Publish `name`, `symbol`, `decimals` and `standards` for kinds 1 and 3 under one `domainSep`, then a Null record for kind 1. Kind 1 is hidden with all fields cleared; kind 3 is unchanged; a second Null changes nothing. A later kind-1 `name = "New"` makes kind 1 visible with only `name`; `standards` reads as empty and the earlier list does not return. A Null record at key `name` has the same effect: it withdraws the whole identity, not only `name`.
- **S4. Reorganization.** Removing the block containing the tombstone in S3 restores the earlier state; re-adding it withdraws the identity again.
- **S5. Unusable fields.** After usable values, an empty `name`, a type-1 `decimals` of `"6"`, or `standards = "mip-0004  mip-0011"` (two spaces) is accepted, makes that field unusable and does not fall back to the earlier value.
- **S6. Separate identities.** The same records under kinds 1, 2 and 3 create three identities, and the same payload from two contracts creates separate identities. Updating one leaves the others unchanged. No color is computed for kind 3.
- **S7. Independent events.** Of two events in one transaction, a malformed one is rejected and the other applies normally.
- **S8. Display.** With `decimals = 2`, the raw amount `123456` displays as `1234.56`.
- **S9. Symbol grouping.** Kinds 1 and 3 of one contract with `symbol = "ACME"` form one group, and an identity of the same contract under another `domainSep` with `symbol = "ACME"` joins it. An identity with `symbol = "ACME"` from another contract, or on another network, does not. Neither do `acme`, ` ACME`, a type-0 value `ACME`, the key `SYMBOL`, or a missing `symbol`. Updating one member's `name` changes no other member; changing one member's `symbol` moves only that member; a tombstone removes it from the group.

## References

- [MIP-0002: Public Contract Log Emission for Compact Smart Contracts](./mip-0002-public-contract-log-emission.md)
- [MIP-0004: Fungible Token Standard with UTXO Conversion](./mip-0004-fungible-token-standard-with-utxo.md)
- [MIP-0011: Native Shielded Token Standard](./mip-0011-native-shielded-token.md)
- [MIP-0014: Native Unshielded Token Standard](./mip-0014-native-unshielded-token.md)
- [MIP-0019: Multipart Event](./mip-0019-multipart-event.md)
- [MPS-0005: Event Emission Support for Compact Smart Contracts](../mps/mps-0005-events.md)
- [Token Registry MPS: "Off-Chain Token Metadata Registry for Midnight", unnumbered at the time of writing, open as PR #104](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)
- [EIP-7496: NFT Dynamic Traits](https://eips.ethereum.org/EIPS/eip-7496)
- [ERC-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155)
- [ERC-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721) (`tokenURI`)
- [RFC 3986: Uniform Resource Identifier (URI): Generic Syntax](https://www.rfc-editor.org/rfc/rfc3986)
- [RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259.html)
- [CIP-26: Cardano Off-Chain Metadata](https://cips.cardano.org/cip/CIP-26)
- [Reference implementation: midnight-experiments/mip-0018](https://github.com/midnight-experiments/mip-0018)
- [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts)

## Acknowledgements

- Robert Blessing-Hartley (@bobblessinghartley), for the Token Registry problem statement this MIP responds to.
- The reviewers on PR #104 (@kapke, @DpacJones, @rongurlavi) for the discussion on the on-chain/off-chain boundary and NFT content metadata.
- Dominik Zajkowski (@dzajkowski) for MIP-0002, without which there is no transport.
- The Wallet Group, for feedback and suggested improvements.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.

---

## Appendix A: Example event (informative)

The payload of test A1: the three common fields plus `standards`, for `domainSep = 0x11` × 32, kind 3.

```
offset  bytes (hex)                                   meaning
0       11 × 32                                       domainSep
32      03                                            kind = ledger
33      04 6e616d65 01 0a 41636d6520546f6b656e        name = "Acme Token"
50      06 73796d626f6c 01 04 41434d45                symbol = "ACME"
63      08 646563696d616c73 02 01 06                  decimals = 6 (Uint<8>)
75      09 7374616e6461726473 01 08 6d69702d30303034  standards = "mip-0004"
95      00 × 161                                      padding
```

Each record reads `keyLen | key | valType | valLen | value`.

## Appendix B: Token Registry MPS fields (informative)

| Token Registry MPS field | This MIP |
|---|---|
| `subject` (color) | Derived for kinds 1 and 2 as `tokenType(domainSep, contractAddress)`; never transmitted. |
| `tokenOrigin` (domain, contract) | `domainSep` from the payload; `contractAddress` from the event record. |
| `name`, `ticker`, `decimals` | Common fields `name`, `symbol`, `decimals`. |
| `description`, `url`, `logo` | Left to a later schema MIP; these can be ordinary keys, with a way to validate any off-chain content ([Off-chain content](#off-chain-content)). |
| quadrant | `kind`. |
| `privacy` | Not defined; a later MIP may add keys. |
| `contractToken.standard` | `standards`. |
| `bridge` | Not defined; an issuer may emit a key for it. |
| attestation, sequence number | Chain execution and chain order. |
| governance / review | Out of scope; the registry's curation role. |
