---
MIP: "0017"
Title: On-Chain Token Metadata Emission (`TokenMetadata` Events)
Authors:
  - Edward Alvarado <edward.alvarado@midnight.foundation>
  - Sebastien Guillemot (@SebastienGllmt) <sebastien.guillemot@midnight.foundation>
Status: Proposed
Category: Standards
Created: 2026-09-17
Requires: MIP-0002: Public Contract Log Emission for Compact
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

Midnight contracts issue *native* tokens, held as protocol-level UTXOs, and *ledger* tokens, represented by balances in contract state. A native token's color is derived from the issuing contract address and a domain separator; a ledger token has no color or native mint effect. Generic consumers have no standardized, authenticated interpretation of a contract's metadata fields or supplied getters.

This MIP defines how contracts publish typed metadata declarations for both token kinds through [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` events. Each `TokenMetadata` event has a fixed 256-byte payload and the name `mip-xxxx:token-metadata[v1]`. Consumers verify events against chain data and apply accepted declarations in execution order, per `(contract address, domainSep, kind, key)`. The event's emitting contract is the authority for its declarations; a native color is derived from that address and `domainSep`.

**This MIP defines the transport, not a metadata schema.** It fixes the bytes on the wire, their transport validation and update order, without requiring or assigning meaning to particular keys. Metadata-specific proposals can define fields and document structure. [Appendix A](#appendix-a-example-keys-informative) gives illustrative examples. The design follows [EIP-7496 (NFT Dynamic Traits)](https://eips.ethereum.org/EIPS/eip-7496) in using keyed values and later events for updates. A verified declaration establishes what a contract emitted, not whether the asset is legitimate.

This MIP responds to the problem statement "Off-Chain Token Metadata Registry for Midnight", referred to below as the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104), with an on-chain, registration-free base layer that an off-chain registry can build on rather than replace.

## Motivation

### The problem

Every native token a wallet can hold is a color: `tokenType(domainSep, contractAddress)`.
Minting is public and static in a transaction (a contract call's transcript carries `effects.shieldedMints` / `effects.unshieldedMints` as `domainSep → amount`), so anyone can enumerate every color ever minted and by which contract.
What nobody can do is say what a color *means*.

Ledger tokens are invisible even at that level.
They never mint, so no transcript effect records them; their `name`, `symbol` and `decimals` do exist on chain, but as fields in a contract-specific state layout that only a client already holding the compiled contract can read.
A scanner that does not know the contract cannot tell that a token is there at all.

Holding the compiled contract is not enough either.
The chain carries no declaration of a contract's ledger layout or of what its pure circuits compute, so a client cannot prove that the artifact it holds describes the deployed contract: that the state field it decodes as `name` is the name, or that `name()` returns what the source it was given says.
It is trusting the provenance of the artifact, not the chain.

The consequences are those listed in the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104): wallets display 64-character hex strings, explorers cannot label tokens, DApps hard-code token lists, and nothing binds a claimed name to the token it claims to describe.

### Why the existing token standards do not solve it

[MIP-0011](./mip-0011-native-shielded-token.md) and [MIP-0014](./mip-0014-native-unshielded-token.md) give issuing contracts `name()`, `symbol()` and `decimals()` circuits.
These read contract *state*, and reading them requires the contract's compiled artifact.
A wallet that encounters an unfamiliar color does not have it.
More fundamentally, a wallet that *does* have it, or even the full source code, still cannot use it as evidence: nothing on chain binds a deployed contract to any source or artifact, so there is no way to verify that the deployed ledger layout and circuits are the ones the source declares.
The supplied `name()` path therefore does not by itself establish a chain-verified metadata declaration.
MIP-0014 recognizes the general-case gap and delegates it to an off-chain registry ("PR #104") keyed by color, with a mandatory `(domain, contractAddress)` recomputation check.

[MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) account-model tokens have the same shape: metadata lives in ledger fields that only a schema-aware client can read.

### Why an off-chain registry alone is not enough

The [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) proposes a CIP-26-style off-chain registry. That is a reasonable design for curated, attested, human-reviewed metadata, and this MIP does not argue against building one.
But an off-chain registry cannot be the *only* layer, for reasons the MPS itself surfaces:

- **An off-chain entry cannot prove who wrote it.** A registry entry claims to speak for a contract's issuer, so the registry has to add signed attestations, sequence numbers, a validating server and a governance process for deciding which signing keys are legitimate, all to establish that the issuer really wrote the entry. But there is nothing on chain for such an attestation to anchor to. A `ContractDeploy` records no creator; the contract's maintenance authority names the committee that may *upgrade* the contract, not who deployed it, and it may be empty; the DUST that paid the deployment fee proves nothing, since anyone may pay for anyone's transaction. The only party that can demonstrably speak for a contract is the contract itself, by executing. An emitted event is exactly that, and it needs no attestation because the transaction already is one.
- **Per-token lookup can reveal a shielded holder's interests.** If a wallet asks a metadata server about the colors it holds, the server can infer interest in those tokens. Synchronizing a broad or curated metadata collection independently of holdings avoids those per-token requests. The same risk returns when a wallet makes token-specific indexer queries or fetches external metadata and media URLs. An indexer may serve a selected collection; neither the stream nor this MIP requires every wallet to download all metadata.
- **Discovery.** A registry "maps known colors to metadata" and is explicitly "not a search engine". A complete on-chain event stream permits discovery of declarations by construction. A filtered indexer response may expose only a selected subset.
- **Registration is a bottleneck.** Every issuer must find, understand and submit to the registry. A contract can publish its own declarations by calling an emitting circuit after deployment, without a separate registration process.
- **The long tail.** Community tokens, collection pieces, test tokens and LP shares will never be curated. They still need a name.

What an off-chain registry adds, and what this MIP deliberately does not attempt, is *curation*: deciding which of several contracts calling themselves "USDC" is the real one, hosting large logos, recording cross-chain bridge mappings, and attaching third-party trust signals.
The two layers compose: on-chain events are the authoritative, issuer-written base; a registry overlays verification badges, allowlists and rich media, and can itself be seeded from the events.

### Why events, not contract state

Compact contracts already have a public event mechanism ([MIP-0002](./mip-0002-public-contract-log-emission.md), live on Stagenet).
Events are the right substrate for metadata because:

1. **The event carries the declaration itself.** Nothing on chain standardizes which contract-state positions mean metadata fields or authenticates a supplied source artifact or pure getter as the deployed contract's metadata interpretation. An event saying only "metadata changed" would still leave consumers dependent on that interpretation. This MIP therefore puts the key and actual value in an event whose layout is fixed here. Verification against chain execution establishes which contract emitted those bytes, without requiring its storage layout or a supplied getter.
2. **They are schema-free for the consumer.** An indexer decodes a `Misc` event by name and byte layout, with no compiled artifact and no per-contract knowledge.
3. **They are an append-only history.** Renames, trait updates and corrections are later events; consumers fold them last-write-wins and can show history.
4. **They cost nothing at rest.** Events are not consensus state ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Event Lifetime"); they do not grow the contract state that every node carries.
5. **The pipeline exists.** `emit` → `Log` opcode → `VersionedLogItem` → indexer `contractEvents` / `MiscContractEvent` is shipping. This MIP adds one convention on top and requires no ledger, compiler or indexer change.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Scope

**Normative** in this MIP: the event envelope [1], the payload layout [2], the `kind` byte [3], token identity [4], key/value handling rules [5], emission rules [6], consumer acceptance, verification and input handling [7.1, 7.3–7.4], and versioning [8].

**Informative** in this MIP: token-state terminology [7.2], example keys ([Appendix A](#appendix-a-example-keys-informative)), the circuit-cost notes ([Appendix B](#appendix-b-circuit-cost-informative)), the mapping to EIP-7496 and the Token Registry MPS ([Appendix C](#appendix-c-mapping-to-eip-7496-and-the-token-registry-mps-informative)), and the reference module and contracts ([Implementation Example](#implementation-example)).

This MIP does not require any specific key to be emitted.
A contract that emits zero `TokenMetadata` events is not non-conforming; it is simply undescribed.
A contract that emits only keys of its own invention is fully conforming.

### Terminology

- **Color**: the 32-byte token type the ledger and wallets see: `tokenType(domainSep, contractAddress)` from the Compact standard library. Only *native* tokens (minted via `mintShieldedToken` / `mintUnshieldedToken`) have a color.
- **Domain separator (`domainSep`)**: the 32 bytes that identify a token *within* a contract; the ERC-1155 `id` analogue. For a native token it is exactly the value passed to the mint primitive. For a ledger token it is any 32 bytes the contract chooses.
- **Native token**: value held as protocol-level UTXOs (shielded Zswap coins or unshielded UTXOs). The issuing contract mints; the protocol moves.
- **Ledger token**: value held as balances in the contract's own state (the [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) / OpenZeppelin `FungibleToken` model). No mint effect ever appears in a transcript; no color exists.
- **Declaration**: a `TokenMetadata` event: a *claim* made by a contract about its own token.
- **Observation**: for a native kind, a verified mint effect in a transaction transcript. Observation criteria for ledger kinds require a separate token specification.
- **Consumer**: any party processing `TokenMetadata` events, such as an indexer, wallet or explorer.

### 1. The event

A `TokenMetadata` event is a [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` event with:

```
Misc {
  name:    pad(32, "mip-xxxx:token-metadata[v1]")   // Bytes<32>, NUL-padded
  payload: <the 256 bytes of [2]>        // Bytes<256>
}
```

The event name is namespaced by this MIP's number to distinguish the convention from unrelated `Misc` events, following the ledger's own `midnight:derive_token` style of lowercase, colon-separated labels. It carries its layout version in square brackets like the ledger's serialization tags (`impact-versioned-log-item[v1]`, `midnight:zswap-memo[v1]`).
`xxxx` is a placeholder for the four-digit number assigned when this MIP is merged; the final string is fixed at that point and this document is updated once.
Throughout this document "a `TokenMetadata` event" means a `Misc` event carrying this name.

`Misc` is the Compact standard library's catch-all event type (`LogEventType::Misc`, `onchain-vm` event tag 10); its `payload` field is fixed at `Bytes<256>` by MIP-0002.

A consumer recognizes a v1 `TokenMetadata` event if and only if:

1. its event type is `Misc`, and
2. `name == pad(32, "mip-xxxx:token-metadata[v1]")`, that is, the bytes `0x6d69702d787878783a746f6b656e2d6d657461646174615b76315d` followed by 5 NUL bytes (to be recomputed for the assigned number).

Events of another type or name, including unsupported versions, MUST be ignored by a v1 consumer. A recognized v1 event MUST have exactly 256 payload bytes and satisfy [2], [3] and [5] before it is accepted. A recognized event that fails any transport rule MUST be rejected and MUST NOT be applied. Consumers SHOULD make the reason for rejection available for diagnostics.

A Compact **constructor cannot emit**, directly or through a circuit it calls.
A conforming contract that wants its metadata published at deployment therefore exposes a circuit (conventionally `publishMetadata()`) that the deployer calls immediately after deployment (see [6.7]).

### 2. Payload layout

The v1 payload MUST be the result of `serialize<TokenMetadataPayload, 256>(payload)` for this ordered Compact struct:

```compact
struct TokenMetadataPayload {
  domainSep: Bytes<32>,
  kind: Uint<8>,
  key: Bytes<32>,
  valType: Uint<8>,
  valLen: Uint<8>,
  value: Bytes<189>
}
```

This MIP uses the canonical [Compact 0.34.0 release](https://github.com/midnightntwrk/compact/releases/tag/compactc-v0.34.0) / language 0.26.0 serialization semantics, with runtime 0.19.0, for v1. The Compact types and their declaration order determine the bytes; a host-language object layout or ledger-state encoding does not. An implementation using another toolchain MUST reproduce these v1 bytes, or emit under a new event version. The struct identifiers `valType` and `valLen` correspond to the wire labels `val-type` and `val-len` below. The resulting payload is exactly 256 bytes, with no padding between fields:

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 32 | `domainSep` | The token within the contract. For a native token, exactly the value passed to `mintShieldedToken` / `mintUnshieldedToken`. For a ledger token, any 32 bytes the contract chooses (e.g. `pad(32, "acme:gold")`). |
| 32 | 1 | `kind` | See [3]. |
| 33 | 32 | `key` | Key identifier bytes, NUL-padded. Compared after trimming trailing NULs [5.1]; `/metadata/` keys require UTF-8. |
| 65 | 1 | `val-type` | How to interpret `value`. See the table below. |
| 66 | 1 | `val-len` | Number of meaningful bytes in `value`. `0 ≤ val-len ≤ 189`. |
| 67 | 189 | `value` | The value bytes. Bytes at offset `≥ val-len` carry no meaning. |

`32 + 1 + 32 + 1 + 1 + 189 = 256`.

The first three fields (`domainSep`, `kind`, `key`) say *which token* and *which attribute* the event is about.
The last three (`val-type`, `val-len`, `value`) carry the attribute's value and are prefixed `val-` to keep them visually separate from the fields that describe the token.

#### 2.1 The `val-type` byte

`val-type` tells a consumer how to interpret `value` even when it does not recognize `key`.
It takes exactly one of the following values:

| `val-type` | Meaning | `val-len` and `value` rules |
|---|---|---|
| `0` | opaque bytes | `Bytes<N>`; `0 ≤ val-len ≤ 189` |
| `1` | UTF-8 string | `Bytes<N>`; `value[0..val-len]` MUST be valid UTF-8 |
| `2` | unsigned integer | Byte-aligned `Uint<8>` through `Uint<248>`; `1 ≤ val-len ≤ 31`, equal to the selected type's serialized size. Emitters SHOULD use `Uint<128>` (`val-len = 16`) by default. |
| `3` | UTF-8 JSON | `Bytes<N>`; `value[0..val-len]` MUST be one complete valid UTF-8 JSON value as defined by [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259.html); object, array and scalar values are allowed |
| `4` | UTF-8 URI | `Bytes<N>`; `value[0..val-len]` MUST be valid UTF-8 and parse as an absolute URI |
| `5` | Null | Empty tuple `[]`; `val-len` MUST be zero; consumers MUST ignore all 189 `value` bytes; emitters SHOULD fill them with NUL |
| `6` to `255` | reserved | MUST reject the event |

For types `0`, `1`, `3` and `4`, `N = val-len`, and the meaningful prefix `value[0..N]` MUST equal `serialize<Bytes<N>, N>(bytes)` for the value's bytes. Each `N` selects a concrete compile-time `Bytes<N>` type; `N` is not a runtime-sized Compact type.

For type `2`, `N = val-len` MUST be 1 through 31, and the bit width `W = 8 × N` selects the concrete native `Uint<W>` type. The meaningful prefix MUST be its canonical Compact serialization in exactly `N` bytes. Consumers MUST decode using the corresponding Compact deserialization semantics for the width indicated by `val-len` and MUST accept every permitted width; `Uint<128>` is an emitter default, not a decoder fallback. Emitters MAY use any permitted width and need not choose the smallest width that holds the number. For example, `serialize<Uint<24>, 3>(6)` and `serialize<Uint<128>, 16>(6)` are both valid type-`2` values when `val-len` is 3 and 16 respectively.

For type `5`, the meaningful payload is `serialize<[], 0>([])`, which contains zero bytes. These backing types define the encoding; the UTF-8, JSON and URI rules in the table additionally constrain the bytes where applicable. The bytes of the 189-byte `value` field after the meaningful prefix remain ignored as specified in [2.2].

Type validation is part of transport validation: an event whose `value` fails the rule for its declared `val-type` MUST be rejected.
A consumer MUST NOT reinterpret a value under a type other than the one declared. Key-specific schemas and handling of schema mismatches belong to metadata-specific MIPs. Type `5` is an explicit null value, distinct from an empty string or byte sequence and from the JSON literal `null` carried under type `3`. Null changes the current value of the exact key without erasing history. Values `6` to `255` are reserved for a future event version.

#### 2.2 Validation

- `val-len > 189` MUST reject the event.
- A reserved `val-type` MUST reject the event.
- `value` failing its `val-type` backing-type or semantic rule MUST reject the event.
- Bytes of `value` at or after `val-len` MUST be ignored by consumers, including all 189 bytes for Null. Emitters SHOULD set ignored bytes to NUL.
- A `key` consisting entirely of NUL bytes (empty key after trimming) MUST reject the event.

An empty string or opaque byte sequence is valid. An empty integer, URI or JSON payload is invalid under the rules above. JSON validity is a transport rule; requirements that a particular key hold an object, array or other shape belong to a metadata-specific MIP.

### 3. The `kind` byte

The `kind` byte takes exactly one of four values. Each value fixes two attributes of the token: its **privacy** (whether the value carries the shielded or the unshielded tag) and its **storage** (whether the value lives in protocol-level UTXOs minted by `mintShieldedToken` / `mintUnshieldedToken`, or in balances kept in the contract's own state).

| `kind` | Privacy | Storage | Example | Has a color? |
|---|---|---|---|---|
| `0` | unshielded | native | MIP-0014 native unshielded token | yes |
| `1` | shielded | native | MIP-0011 native shielded token | yes |
| `2` | unshielded | ledger | MIP-0004 / OpenZeppelin `FungibleToken` balances | no |
| `3` | shielded | ledger | contract-state balances the contract keeps confidential | no |

Any other value MUST reject the event.

**Note on `kind = 3`.** The label is the contract's declaration about how it keeps balances; this MIP does not verify that those balances are confidential. Consumers MUST NOT present a token as private solely on the strength of this byte. Values `4` to `255` are reserved for a future event version; the event name carries the version and per-token flags belong in ordinary keys.

A color exists only for native kinds (`0` and `1`).
A consumer MUST NOT derive or display a color for a ledger kind (`2` or `3`).

### 4. Token identity

A token is identified by the triple **`(contractAddress, domainSep, kind)`**, scoped to a particular Midnight network. Equal triples on different networks are not the same identity.

- `contractAddress` is taken from the event's own `contractAddress` field in the indexer/ledger event record, never from the payload [6.1].
- `domainSep` is payload offset 0.
- `kind` is the full byte [3]. Each of the four kinds is a distinct token identity, even under one `domainSep`:
  - A contract MAY mint the same `domainSep` both shielded (kind `1`) and unshielded (kind `0`). The ledger keeps the two apart by tag, not by value: they share one color but are two token types, and a consumer shows two rows sharing one color.
  - A contract MAY hold the same `domainSep` both as contract-state balances (kind `2` or `3`) and as native UTXOs (kind `0` or `1`). A [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) token is exactly this: balances in state, converted on demand to shielded or unshielded UTXOs under the token's `domain`. It is one asset in up to three representations, and each representation is its own row.

A contract describes each `(domainSep, kind)` it wants described separately, and a consumer MAY link rows that share `(contractAddress, domainSep)` as representations of one asset.

**`domainSep` for ledger kinds.** For a native kind, `domainSep` is grounded: it is the mint argument and the color is derived from it. For a ledger kind nothing on chain ties `domainSep` to any balance structure; it is a label the contract chooses to identify one balance book. Two rules follow:

- A contract that holds the same asset both in state and natively (MIP-0004 style) MUST use its native `domain` as the ledger `domainSep`, so that the rows link.
- A pure ledger contract uses any stable 32 bytes per balance book. Several ledger `domainSep`s from one contract declare several balance books (the ERC-1155 shape in state). Whether the underlying balance structures can be observed is defined by an applicable ledger-token specification [7.2].

For native tokens the color is derived, never transmitted:

```
color = tokenType(domainSep, contractAddress)
```

where `tokenType` is the Compact standard library function.
(In the current ledger this is `persistentCommit([domainSep, contractAddress], pad(32, "midnight:derive_token"))`; the standard library function is the normative reference, not the underlying hash.)
A consumer MUST compute the color itself from `(domainSep, contractAddress)` and MUST NOT accept a color supplied in a `value`.

### 5. Keys and values

#### 5.1 Key comparison

Trailing NUL bytes are trimmed from `key`, then the remaining bytes are compared exactly.
Keys are case-sensitive.
Except for `/metadata/` keys as specified below, keys SHOULD be valid UTF-8; consumers MUST NOT reject another key solely for invalid UTF-8 (they MAY display it as hex).

Keys whose trimmed bytes begin with the UTF-8 prefix `/metadata/` MUST be valid UTF-8 JSON Pointer strings under [RFC 6901](https://www.rfc-editor.org/rfc/rfc6901.html), within the same 32-byte key limit. In particular, a literal `~` in a reference token is encoded as `~0`, and `/` within a token as `~1`; any other `~` escape is invalid and MUST cause rejection. The `*` in `/metadata/*` is a literal property token, not a wildcard. The token `0` in `/metadata/0` may identify an array element or an object property named `"0"`, depending on a document structure defined elsewhere.

This MIP does not define the target document, whether one exists, or how a value is applied at a path. In particular, pointer syntax does not require JSON assembly, nested updates, prefix replacement or concatenation. Last write wins only for the same exact key [6.2]. Other keys remain independent identifiers under the byte-comparison rule above.

#### 5.2 Values are typed bytes

At the transport level a value is a `val-len`-byte Compact-serialized prefix tagged with a `val-type` [2.1].
The type says how to *read* the bytes; this MIP assigns no *meaning* to any key. A transport-valid declaration is accepted even if its key is unknown. Consumers MAY select which tokens, fields and history to retain or serve; they MUST NOT treat an omitted entry in a selected service as proof that no declaration exists on chain. Metadata-specific MIPs define key meanings, required fields, schema validation, projections and handling of schema mismatches. A schema mismatch alone does not make a transport-valid event malformed under this MIP.

#### 5.3 Metadata schemas

[Appendix A](#appendix-a-example-keys-informative) illustrates possible keys and values without standardizing their encodings or meanings. A metadata-specific MIP MAY define a schema for some keys using this transport. Such a schema can impose requirements on its own implementations without changing transport acceptance of other keys.

#### 5.4 Values longer than 189 bytes

A single event carries at most 189 value bytes.
This MIP defines no multipart representation or reassembly rule. A metadata-specific MIP may define one; a URI value may point to an external document under that MIP's rules.

### 6. Emission rules

#### 6.1 Authority

The emitting contract is the only authority for `(its own address, domainSep, kind)`.
A consumer MUST take the contract address from the event record's `contractAddress`, never from the payload.
Because a native token's color is derived from `(domainSep, contractAddress)`, no contract can describe another contract's color: an event from contract A about `domainSep` X describes `tokenType(X, A)`, which is A's token by construction.

#### 6.2 Last write wins

Per network and `(contractAddress, domainSep, kind, key)`, the last accepted event is the current value. Apply events in canonical block order, then transaction execution position within the block, then the order produced by ledger execution within that transaction. An indexer's monotonic event ID may serve as its cursor but does not define the normative order or token identity. Consumers using provisional blocks MUST roll back values from blocks removed by a reorganization; consumers may instead wait for finalized chain data.

Earlier values remain history, which consumers MAY retain. Type `5` sets the current value of the exact key to Null without erasing its history. A zero-length string or opaque byte sequence is present and empty, not Null.

#### 6.3 Observation is independent of declaration

A mint effect in a transcript is a fact; a `TokenMetadata` event is a claim.
Because the full `kind` is part of the identity [4], the two never contradict each other; they populate rows independently:

- A mint of `(domainSep, kind 0 or 1)` establishes a native observation whether or not anything was declared for it. A declaration can introduce a declared native identity but cannot manufacture, hide or relabel a mint.
- A declaration for `(domainSep, kind)` populates exactly that row and no other. Declaring kind `2` says nothing about kind `0`; declaring kind `1` for a `domainSep` only ever minted as kind `0` describes a token that has not been minted yet, not the one that has.
- Observation criteria for ledger kinds (`2` or `3`) belong to an applicable ledger-token specification. This MIP alone does not establish them.

#### 6.4 Describing an unminted token is legal

A ledger kind has no native mint effect. A native kind MAY be declared before, or without, its first mint. Under the informative state terminology in [7.2], such a native token is **declared** until observed. A ledger declaration is also **declared** unless an applicable ledger-token specification establishes an observation.

#### 6.5 No registration

Nothing is registered with anyone. A consumer can discover declarations from verified on-chain events [7.3]. Indexer selection and service coverage are implementation-dependent.

#### 6.6 Disclosure

Everything that reaches `emit` is public.
Contracts MUST pass `disclose(...)` for any witness-derived value, exactly as for any other public write; the Compact compiler enforces this.
Emitters SHOULD NOT emit anything they would not write to public ledger state.

#### 6.7 Publication and access control

Because constructors cannot emit, a contract that wants deployment-time metadata exposes a circuit the deployer calls after deployment.
Whether that circuit is callable once (a publish-once guard), owner-gated, or open is the contract's own policy; this MIP takes no position, with one caveat: a circuit that emits `TokenMetadata` for a token whose metadata is meant to be stable SHOULD be access-controlled, because anyone who can call it can rename the token (see Security Considerations).

### 7. Consumer rules

#### 7.1 Acceptance and rejection

A consumer applies [1] to recognize a v1 `TokenMetadata` event, then [2], [3] and [5] to validate it:

| Input | Transport outcome |
|---|---|
| Unrelated event type or unsupported name/version | Ignore. |
| Recognized v1 event with invalid payload size or another transport-rule violation | Reject; MUST NOT apply. |
| Recognized v1 event satisfying transport rules, including one with an unknown key | Accept as a typed declaration. |

Acceptance here does not impose metadata-specific schema interpretation; authoritative use also requires chain verification [7.3]. Consumers SHOULD make rejection reasons available for diagnostics.

#### 7.2 Token states

The following terminology is informative. It does not require a consumer UI or database schema. A declaration is an accepted `TokenMetadata` event verified against chain data. Native observation comes from a verified mint; ledger observation depends on criteria in an applicable ledger-token specification.

| State | Native token | Ledger token |
|---|---|---|
| **observed** | A verified mint exists; no metadata declaration has been accepted. | Observation criteria are specification-dependent; no metadata declaration has been accepted. |
| **declared** | A metadata declaration has been accepted; no mint has been observed. | A metadata declaration has been accepted; no observation has been established under an applicable ledger-token specification. |
| **described** | Both a verified mint and an accepted metadata declaration. | An accepted metadata declaration plus an observation established under an applicable ledger-token specification. |

This MIP does not define ledger-token observation criteria. Without an applicable mechanism, accepted ledger metadata is **declared**; that status does not establish that its balance structure is absent. A declaration of an unminted native token is likewise **declared**, and cannot establish that a mint occurred.

For example, a contract that declares only kind `2` for a `domainSep` it then mints as kind `0` has two distinct identities: kind `0` is **observed** without a declaration, while kind `2` is **declared** without an observation established by this MIP. An applicable ledger-token specification could establish a ledger observation separately. A consumer may relate identities sharing a `domainSep`.

#### 7.3 Reading events from transactions

Event contents are produced by execution: `emit` compiles to the VM's `log` opcode and its operand comes from the stack at runtime. Consumers MUST verify indexer-supplied metadata against authenticated on-chain data before treating it as authoritative. Verification establishes the actual event bytes, emitting contract, successful execution and inclusion in canonical order; an indexer response alone does not establish those facts.

Guaranteed transcript effects apply when the transaction succeeds or partially succeeds; fallible transcript effects apply only for successful segments. Events and native mint observations follow the execution outcome of the transcript that produced them. Discovery and retrieval algorithms are implementation-dependent.

Indexers MAY choose which tokens, fields and history they provide. Omission from a filtered response is not evidence that no declaration exists on chain. A claim that a value is current requires verifying that no later applicable update supersedes it over the relevant chain range. A consumer cannot infer completeness or currentness merely from authenticating the events that a service returned.

#### 7.4 Untrusted input

Every byte of a `TokenMetadata` payload is attacker-controlled.
Consumers MUST bound-check all offsets and lengths, MUST treat `value` as untrusted for any parser they apply to it (UTF-8, JSON, URI), and MUST NOT dereference a remote URI found in a value without the same precautions they would apply to any remote content (see Security Considerations).

### 8. Versioning

**The event name is the version.**
The bracketed suffix is the layout version, in the same form the ledger uses for its serialization tags.
A future, incompatible layout uses a new name and never a reinterpretation of `mip-xxxx:token-metadata[v1]`: an amendment within this MIP bumps the bracket (`mip-xxxx:token-metadata[v2]`), and a superseding MIP gets a fresh name for free (`mip-yyyy:token-metadata[v1]`).
A consumer that only knows `[v1]` ignores the other names; a consumer that knows several keeps them apart.

After finalization, a version fixes its payload layout and Compact serialization, accepted `val-type` and `kind` values, and transport-validation rules. Assigning a reserved datatype or kind, or changing those rules, requires a new event version. New keys may be introduced without a new event version because the transport assigns no fixed key registry or schema. Appendix A may change without changing transport rules.

### Out of scope

- **Which metadata a token should expose.** This MIP fixes the transport. Required fields, per-asset-class schemas (fungible, NFT, RWA), localization and media formats belong in separate, layered proposals that use this transport.
- **How metadata documents and ledger observations work.** Metadata-specific MIPs define document structure, path application, multipart representation, schema-mismatch outcomes and projections. An applicable ledger-token MIP defines evidence for observing a ledger balance structure.
- **Curation and trust.** Which of several contracts calling themselves "USDC" is real is not a question on-chain data can answer. That is the job of an off-chain registry, allowlist or attestation layer, which MAY be seeded from these events.
- **Metadata for tokens whose issuer does not emit.** A contract deployed without this convention cannot be described on chain by anyone else [6.1]. Most such contracts can be upgraded by their own maintenance authority to emit (see [Upgrade Path for Existing Contracts](#upgrade-path-for-existing-contracts)); for the rest, an off-chain registry is the only path. This is a deliberate consequence of the authority rule, not an oversight.
- **NIGHT and DUST.** The protocol's native asset has protocol-defined properties that clients hard-code; DUST is not a token. Neither is described by this mechanism.
- **Private or encrypted metadata.** All `TokenMetadata` events are public. Selective-disclosure metadata is a Phase-2-events concern ([MPS-0005](../mps/mps-0005-events.md)).
- **Cross-chain identity mapping** (the `bridge` property of the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)). A `bridge` key MAY be emitted as a trait; its schema is not defined here.

## Rationale

### Why transport-only, with fields informative?

Three reasons.

First, the fields a token needs depend on what the token is. A fungible token wants `decimals`; an NFT wants per-piece traits and a content pointer; an RWA wants issuer and jurisdiction fields ([MPS-0023](../mps/mps-0023-rwa-standard-interface.md)). Baking one schema into the transport would either bloat it or leave asset classes out.

Second, EIP-7496's lesson is that an open key/value space with a *conventional* well-known subset ages better than a closed struct. ERC-20's `name`/`symbol`/`decimals` succeeded as a convention over a generic ABI, not as a wire-format requirement.

Third, layering keeps this MIP small and stable. A schema MIP for fungible tokens can be argued, revised and superseded without touching how bytes reach the chain.

Appendix A illustrates potential keys without assigning binding encodings or projections. Metadata-specific MIPs can establish interoperable schemas for the assets that need them, while retaining this common event transport.

### Why one event per `(key, value)` rather than one blob per token?

- The `Misc` payload is fixed at 256 bytes by MIP-0002. A single blob would need a compression or chunking scheme before it could hold `name` + `symbol` + `decimals` + a URL.
- Per-key events make updates cheap and precise: renaming a token is one event, not a re-emission of everything.
- Per-key folding is what EIP-7496's `TraitUpdated` does, and what an indexer wants to store anyway.
- A larger document can be addressed by a metadata-specific MIP without changing this event envelope.

### Why `Misc` rather than a new `LogEventType` variant?

A dedicated `TokenMetadata` variant in MIP-0002's enum would give the indexer built-in schema knowledge and field indexing.
It would also require a ledger release, a bumped serialization tag and a coordinated node upgrade, and would freeze the payload layout at the protocol level.
`Misc` is available today on Stagenet, costs nothing to adopt, and the event-name-as-version rule [8] gives the evolution path a protocol enum would not.
If the convention proves out, promoting it to a standard variant is a follow-up this design does not preclude.

### Why `domainSep` in the payload and not the color?

The color is a function of `(domainSep, contractAddress)`, and `contractAddress` is already authenticated by the event record.
Transmitting the color would add 32 bytes of redundant, forgeable data; transmitting `domainSep` lets the consumer *derive* the color and makes it impossible for a contract to claim someone else's.
It also covers ledger tokens, which have a `domainSep` but no color.

### Why the `kind` byte?

Midnight has four value domains (shielded/unshielded × native/ledger), and one 32-byte value can legitimately name up to four distinct token types: the same `domainSep` minted shielded and unshielded, and the same `domainSep` held as state balances and converted to UTXOs (MIP-0004).
A consumer that keyed only on color would merge the first pair; one that keyed on color plus privacy would merge the second, and would then read a MIP-0004 token's ledger declaration as contradicting its own mints.
Making the whole byte part of the identity removes both collisions, and it removes the need for any "which declaration wins" rule: every `(domainSep, kind)` is its own row, populated by its own mints and its own declarations.
It also tells a consumer, per identity, whether to derive a color and whether native mint observations apply.

### Why observations and declarations are kept independent?

Events are claims ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Event trust model").
Mint effects are facts the ledger verified.
A consumer that let a claim override a fact would let a contract hide its own mints. A declaration can introduce a declared native identity, but it cannot create, remove or relabel an observed mint.
Detecting contradictions between the two (a ledger declaration for a `domainSep` that was minted natively) and flagging the row was considered and rejected.
With the full `kind` in the identity there is nothing to contradict: the declaration and the mint describe different identities, one with an accepted declaration and the other without.
The **declared** versus **described** terminology [7.2] records whether an applicable observation has also been established.

### Why on-chain `name`/`symbol` despite MIP-0014's rejection of it?

MIP-0014 rejects on-chain `name`/`symbol` because "ledger state is public and untrusted for identity, so on-chain `name`/`symbol` would invite impersonation without removing the need for a derivation check."

Both halves are true and neither is an argument against this design:

- **Impersonation is medium-independent.** Anyone can call their token "USDC" in an off-chain registry too; CIP-26 relies on Cardano Foundation review to sort it out. What this MIP adds is that the *claim is cryptographically bound to the claimant*: the event's `contractAddress` is authenticated by the transaction, and the color a consumer derives from it cannot be the color of anyone else's token. That is strictly more than an unattested registry entry offers, and it is the same binding MIP-0014's own registry path requires the wallet to recompute.
- **The derivation check is not removed; it is automatic.** A consumer *only ever* derives the color from `(domainSep, contractAddress)`; there is no transmitted color to check against. The check MIP-0014 mandates is the only way a color enters the table at all.

What on-chain metadata does not do, and MIP-0014 is right that nothing on chain can do, is tell a user *which* USDC to trust. That is curation, and it is out of scope here by design.

### Why events rather than a metadata field in contract state?

The decisive reason is verifiability.
A `name` ledger field is only a name because a supplied contract schema says so, and nothing on chain standardizes or authenticates that interpretation. A supplied pure getter and source code likewise do not themselves establish a chain-verified metadata declaration. A change notification would leave this gap in place. An event instead carries the actual key and value in a fixed layout; verified execution establishes that the emitting contract declared those bytes. A metadata-specific MIP supplies the key's meaning.

Two lesser reasons point the same way.
State grows the set every node carries; events do not.
And state has no natural history: a rename overwrites, whereas events append.

### Why 189 value bytes and fixed widths?

The 256-byte `Misc` payload is given.
32 bytes of `domainSep` and 32 of `key` are the natural widths (EIP-7496 uses `bytes32` for trait keys), leaving 192 for `kind`, `val-type`, `val-len` and `value`.
Fixed widths make the decoder trivial and byte-exact, which matters when every consumer independently reimplements it.

### Why a `val-type` byte?

EIP-7496 leaves trait values as untyped `bytes32` and describes their types in an off-chain trait-metadata document.
That works when a consumer already knows the collection; it fails for the generic explorer that meets an unknown key and has nothing but bytes.
One byte of type tag identifies the value's transport encoding without requiring knowledge of the key. A metadata-specific MIP can use that information when defining fields.
It costs one byte of value width. Larger-value representation is left to a metadata-specific MIP.
The tag is deliberately a closed enum within each event version.

### Alternatives considered

- **Off-chain registry only (the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) as proposed).** Rejected as the sole layer: it needs attestation infrastructure and governance to provide the provenance that verified on-chain emission provides directly. Broad or curated synchronization can avoid token-specific lookups. Retained as a complementary curation layer.
- **URI pointer only (ERC-721 style).** Rejected as the sole mechanism: it moves every field behind an external fetch, which can reveal interest in shielded holdings and adds a liveness dependency. Appendix A includes an illustrative URI key for future schemas.
- **A protocol-level `LogEventType::TokenMetadata` variant.** Deferred; see above.
- **JSON in every event.** Rejected: 189 bytes is too small for many JSON documents, and it makes simple values more expensive to emit and decode than typed bytes. A complete JSON value can still be carried under type `3` when it fits.

## Path to Active

### Acceptance Criteria

- At least one independent consumer (indexer, explorer or wallet) verifies and processes `TokenMetadata` events, and demonstrates the three informative states of [7.2] using fixtures that exercise this MIP's rules.
- At least one issuer other than the author adopts the convention on a public network.
- The Compact module is published in a form issuers can import (this repository or an ecosystem library such as OpenZeppelin Compact Contracts).
- Community review through the MIP process, including reconciliation with the authors of the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) on the on-chain/off-chain boundary.

### Implementation Plan

1. **Reference module and contracts**: illustrative examples exist; see [Implementation Example](#implementation-example).
2. **Reference deployment**: done on Stagenet (see below); repeat on Preprod when the events pipeline is available there.
3. **Consumer reference**: an illustrative decoder and simulator/Stagenet fixtures are published; propose `TokenMetadata` decoding to at least one public explorer.
4. **Library adoption**: propose the module (or an equivalent) to OpenZeppelin Compact Contracts as an optional extension of `NativeShieldedToken`, `NativeShieldedTokenFamily` and `FungibleToken`.
5. **Upgrade template**: publish the added-circuit template for pre-v9 contracts and exercise it on Stagenet by upgrading a contract deployed without events, per [Upgrade Path for Existing Contracts](#upgrade-path-for-existing-contracts).
6. **Schema follow-ups**: a fungible-token schema MIP defining common fields for MIP-0011/0014/0004 tokens, and an NFT content-metadata MIP (per the discussion on PR #104), both using this transport. Multipart documents and ledger observation criteria are separate follow-ups.

## Backwards Compatibility Assessment

No protocol, compiler or indexer change is required; this MIP is a convention over [MIP-0002](./mip-0002-public-contract-log-emission.md)'s existing `Misc` event.
No deployed contract or protocol state is changed by this proposal. Contracts that do not emit `TokenMetadata` have no declarations under it.

Existing contracts deployed before ledger v9 cannot emit as deployed; [Upgrade Path for Existing Contracts](#upgrade-path-for-existing-contracts) describes how their maintenance authority adds an emitting circuit without redeployment. Only contracts with an empty or unreachable maintenance authority are left to an off-chain registry.

Adopting the convention is additive to [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md), [MIP-0011](./mip-0011-native-shielded-token.md) and [MIP-0014](./mip-0014-native-unshielded-token.md): a conforming token can retain its existing circuits and emit declarations as events. This MIP does not require matching values for particular keys; metadata-specific MIPs may define such consistency rules.

## Upgrade Path for Existing Contracts

### The gap

Contract events exist only from Midnight 2.x (ledger v9) onward.
Every contract deployed on Midnight 1.x (ledger v8), including every token contract live on mainnet today, was compiled without `emit` and cannot produce a `TokenMetadata` event as deployed.
Without an upgrade path, this MIP would describe only tokens issued after the network upgrade, and the tokens users already hold would stay as raw hex.

### The proposal

Existing contracts do not need to be redeployed or migrated.
A Midnight contract's circuits are a set of verifier keys held in its on-chain state, and the contract's maintenance authority can change that set after deployment through a maintenance update: `VerifierKeyInsert(operation, vk)` adds a circuit, `VerifierKeyRemove` retires one.
The token itself, its address, its `domainSep`, its color and every UTXO or balance already issued are untouched by such an update.

The upgrade is therefore, per contract, once the network runs ledger v9:

1. **Compile a new circuit** against the contract's existing ledger layout, conventionally `publishMetadata()` (and, if updates are wanted, `setMetadata(...)`), that emits the `TokenMetadata` events of [1] and [2] for each `(domainSep, kind)` the contract issues.
2. **Insert its verifier key** with a maintenance update signed by the contract's maintenance authority.
3. **Call the new circuit** once. An already-observed native token with an accepted declaration is then **described** under the informative terminology in [7.2]. An unminted native token or a ledger token without an observation established by another specification is **declared**.

Because the maintenance authority is a committee with a threshold, the upgrade is exactly as permissioned as any other change the issuer already reserved the right to make.
No new trust is introduced: a consumer folding the resulting events applies the same rules as for a contract that emitted from day one, and the authority rule [6.1] holds because the event still comes from the token's own contract.

### Token identity in the new circuit

The new circuit reads the contract's existing `domain` from state and emits it as `domainSep`; it does not need to compute or emit the color, which a consumer derives [4].

### Limits

- A contract whose maintenance authority is **empty** cannot be upgraded by anyone. If it does not already emit under this convention, an off-chain registry remains the path for metadata (see [Out of scope](#out-of-scope)).
- A contract whose authority committee is no longer reachable is in the same position in practice.
- The metadata-only procedure described here adds an emitting circuit while preserving the contract's existing operations and state. A token whose authority is retired before publication stays without an on-chain declaration under this convention.

### Timeline

The upgrade can be prepared before the network upgrade (compile the circuit, agree the metadata, line up the authority signatures) and executed immediately after it.
Because the events pipeline ships with ledger v9 itself, there is no second dependency to wait for: once the network runs v9, every upgradable token can publish a declaration.
The reference repository will publish the added-circuit template alongside the from-scratch templates.

## Security Considerations

### Impersonation

Any contract can emit a `name` key with the value `"USDC"`.
This MIP binds the claim to the emitting contract's address and, for native tokens, to a color no other contract can produce; it does not and cannot say whether that contract is the one a user means.
Consumers presenting a declaration as issuer metadata MUST preserve its `(network, contractAddress, domainSep, kind)` provenance. They SHOULD integrate a curation signal (allowlist, registry attestation or user confirmation) before presenting a claimed asset identity as trusted.
This is identical to the situation on every EVM chain and to an unattested CIP-26 entry.

### Unauthorized updates

A `setMetadata`-style circuit with no access control lets anyone rename a token.
Emitters SHOULD gate metadata-emitting circuits (the reference contracts use OpenZeppelin `Ownable`) or make them publish-once.
Consumers MAY surface the history of a key so that a hostile rename is visible.

### Spam and resource use

Events are metered by the existing `Log` opcode fee model and compete for the per-block `bytes_written` budget ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Fee Metering").
No new spam vector is introduced.
An indexer or consumer MAY select tokens, fields, retained history and service scope as local policy. A filtered service must not imply that omission establishes on-chain absence or that an authentic returned event is still the latest event.

### Untrusted payloads

Every payload byte is attacker-controlled [7.4].
Decoders must be bound-checked; UTF-8, JSON and URI parsers applied to `value` must be hardened against malformed input. This MIP does not define multipart reassembly.
Fetching a token-specific metadata or image URI can reveal interest in that token to the remote host. Wallets SHOULD account for this before fetching such content for a shielded holder.

### Declarations that the chain cannot corroborate

A contract can declare a ledger balance book under any `domainSep` without this MIP establishing that such a structure exists in its state. An applicable ledger-token specification may define evidence for observation [7.2]. Until then, the declaration is **declared** under the informative terminology here. A contract that declares its ledger side but not its native UTXOs yields a separate observed native identity; the mint is never hidden by the declaration.

### Disclosure

`emit` is a disclosure site and the compiler enforces that emitted values are disclosed.
No new leakage path is introduced; an issuer that emits a value has chosen to make it public.

### Proving cost

The measured publication shapes and their circuit-row counts are listed in [Appendix B](#appendix-b-circuit-cost-informative).

## Implementation Example

Illustrative implementation: [`acedward/mip-erc7496-midnight-contracts` at `d4d6d0b`](https://github.com/acedward/mip-erc7496-midnight-contracts/tree/d4d6d0b773adaf29426ecf533716b809c51aa654) (Apache-2.0). Its contracts, decoder and fixtures show possible integration patterns. This MIP defines conformance; the linked material is not a substitute for its requirements.

### Components

- **`contracts/TokenMetadata.compact`**: an illustrative module issuers can adapt. Two circuits: `emitTokenMetadata(domainSep, kind, key, valType, valLen, value)` emits one event with the [2] layout; `emitStandardFields(domainSep, kind, name, nameLen, symbol, symbolLen, decimals)` emits three example fields. Three constants: `KIND_UNSHIELDED()` = 0, `KIND_SHIELDED()` = 1, `KIND_LEDGER_FLAG()` = 2.
- **Reference contracts** illustrate several token representations, each composing an OpenZeppelin token module (where one exists), OpenZeppelin `Ownable` for update gating, and the module above:
  - `NativeShieldedToken.compact`: one static domain, kind 1 (MIP-0011 Fungible profile + events).
  - `NativeUnshieldedToken.compact`: one static domain, kind 0 (MIP-0014 shape + events).
  - `NativeDualToken.compact`: one domain minted both shielded and unshielded; publishes six events across two circuits.
  - `ShieldedCollection.compact`: one address, one domain per piece (MIP-0011 Family profile + per-piece events); the EIP-7496 shape.
  - `LedgerToken.compact`: OpenZeppelin `FungibleToken` balances, kind 2; an example of a ledger declaration.
  - `contracts/generated/*.compact`: variants with metadata as compile-time literals.
- **Consumer reference**: `test/token-metadata.ts`, an illustrative event decoder.
- **Fixtures**: `fixtures/simulator/` (offline events, mints, color vectors, expected token rows, negative payloads) and `fixtures/stagenet/` (recorded from the public Stagenet indexer, including raw transaction bytes).

### Reference deployment (Stagenet)

The illustrative reference set is deployed to Midnight Stagenet: eleven contracts showing native and ledger declarations, including a token minted without a declaration, a token declared before minting, a dual-kind token producing two identities under one color, a collection with one `domainSep` per piece, and a contract that declares its ledger side but mints natively.

Contract addresses, colors and deployment details are published and maintained in [`effectstream/staging-tokens-addresses`](https://github.com/effectstream/staging-tokens-addresses), which is the authoritative list; this document does not repeat them so that redeployments do not leave stale addresses in a merged MIP.
The recorded events, expected token rows and raw transaction bytes for the same deployment live in the reference repository's `fixtures/stagenet/` directory.

### Dependencies

- [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` events (Compact 0.34.0 / language 0.26.0 / runtime 0.19.0, Midnight ledger 9, as deployed on Stagenet).
- OpenZeppelin Compact Contracts `0.4.0-alpha.1` for the reference contracts' token and access-control modules. The module itself depends only on the Compact standard library.

## References

- [MIP-0002: Public Contract Log Emission for Compact Smart Contracts](./mip-0002-public-contract-log-emission.md)
- [MIP-0004: Fungible Token Standard with UTXO Conversion](./mip-0004-fungible-token-standard-with-utxo.md)
- [MIP-0011: Native Shielded Token Standard](./mip-0011-native-shielded-token.md)
- [MIP-0014: Native Unshielded Token Standard](./mip-0014-native-unshielded-token.md)
- [MPS-0005: Event Emission Support for Compact Smart Contracts](../mps/mps-0005-events.md)
- [Token Registry MPS: "Off-Chain Token Metadata Registry for Midnight", unnumbered at the time of writing, open as PR #104](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)
- [EIP-7496: NFT Dynamic Traits](https://eips.ethereum.org/EIPS/eip-7496)
- [ERC-1155: Multi Token Standard](https://eips.ethereum.org/EIPS/eip-1155)
- [ERC-721: Non-Fungible Token Standard](https://eips.ethereum.org/EIPS/eip-721) (`tokenURI`)
- [RFC 6901: JavaScript Object Notation (JSON) Pointer](https://www.rfc-editor.org/rfc/rfc6901.html)
- [RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259.html)
- [CIP-26: Cardano Off-Chain Metadata](https://cips.cardano.org/cip/CIP-26)
- [Reference implementation: acedward/mip-erc7496-midnight-contracts](https://github.com/acedward/mip-erc7496-midnight-contracts)
- [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts)

## Acknowledgements

- Robert Blessing-Hartley (@bobblessinghartley), for the Token Registry problem statement this MIP responds to.
- The reviewers on PR #104 (@kapke, @DpacJones, @rongurlavi) for the discussion on the on-chain/off-chain boundary and NFT content metadata.
- Dominik Zajkowski (@dzajkowski) for MIP-0002, without which there is no transport.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.

---

## Appendix A: Example keys (informative)

The examples below demonstrate transport encodings only. They do not define required fields, key meanings, schema types, validation beyond [2] and [5.1], projections or display behavior. Metadata-specific MIPs may define those rules. Each `Encoded` line shows exactly the meaningful `val-len` bytes; the ignored remainder of the 189-byte `value` field is omitted.

| Example key | Example `val-type` | `val-len` | Example value | Transport interpretation |
|---|---|---:|---|---|
| `name` | `1` string | 10 | `Acme Token`<br>Encoded: `0x41636d6520546f6b656e` | A UTF-8 value under the exact key `name`. |
| `symbol` | `1` string | 4 | `ACME`<br>Encoded: `0x41434d45` | A UTF-8 value under the exact key `symbol`. |
| `decimals` | `2` integer | 16 | `6` as `Uint<128>`<br>Encoded: `0x06000000000000000000000000000000` | The recommended default uses `serialize<Uint<128>, 16>(6)`. |
| `count` | `2` integer | 3 | `6` as `Uint<24>`<br>Encoded: `0x060000` | Another permitted width uses `serialize<Uint<24>, 3>(6)`. |
| `metadata` | `3` JSON | 25 | `{"description":"Example"}`<br>Encoded: `0x7b226465736372697074696f6e223a224578616d706c65227d` | One complete JSON value fitting in this event. |
| `/metadata/0` | `1` string | 11 | `hello world`<br>Encoded: `0x68656c6c6f20776f726c64` | An RFC 6901 pointer key with a UTF-8 value; no array or assembly behavior follows from the path alone. |
| `tokenUri` | `4` URI | 30 | `https://example.org/token.json`<br>Encoded: `0x68747470733a2f2f6578616d706c652e6f72672f746f6b656e2e6a736f6e` | An absolute URI value. |

For `/metadata/0`, the entire pointer string is the key. A future metadata schema can define the target document and whether token `0` denotes an array element or an object property. Keys such as `description`, `image`, `website`, `metadataUri` or `bridge` can also be used, with meaning supplied by a separate schema or application convention.

## Appendix B: Circuit cost (informative)

The five publication shapes and row counts below come from the [pinned historical MinoCrab-v3 benchmark](https://github.com/acedward/mip-erc7496-midnight-contracts/blob/d4d6d0b773adaf29426ecf533716b809c51aa654/benchmarks/token-metadata-shapes.md), rather than the current deployed reference contracts. The fixtures were compiled with Compact 0.34.0 and measured with its bundled `zkir-v3 mock-compile` oracle, with matching MinoCrab-v3 cost-model results.

| Measured publication shape | Events | Circuit rows |
|---|---:|---:|
| Fixed literal publisher (`name`, `symbol`, `decimals`) | 3 | 120 |
| Publisher assembling payloads from ledger fields | 3 | 3,715 |
| One fully runtime typed event | 1 | 1,914 |
| Two independent runtime-built events in one circuit | 2 | 3,789 |
| Three independent runtime-built events in one circuit | 3 | 5,672 |

Metadata emission is inexpensive in these measured circuit shapes.

## Appendix C: Mapping to EIP-7496 and the Token Registry MPS (informative)

### EIP-7496

| EIP-7496 | This MIP |
|---|---|
| `tokenId` | `domainSep` (+ `kind`) |
| `traitKey: bytes32` | `key: Bytes<32>` |
| `traitValue: bytes32` | `value: Bytes<189>` with `val-type` and `val-len`: longer, typed values, no hashing |
| trait types described in the `getTraitMetadataURI` document | `val-type` byte, in-band |
| `TraitUpdated` event | one `TokenMetadata` `Misc` event |
| `getTraitValue(tokenId, traitKey)` | a consumer may derive the latest value for a retained key |
| `getTraitMetadataURI` | a metadata-specific MIP may define a corresponding document or pointer |
| ERC-721 `tokenURI(tokenId)` | a metadata-specific MIP may define a URI key, such as the Appendix A example `tokenUri` |
| the contract is the authority | the verified event's emitting address establishes the authority for both native and ledger declarations; native color is derived from that address and `domainSep` |

### Fields of the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)

| Token Registry MPS field | Here | Notes |
|---|---|---|
| `subject` (color hex) | derived: `tokenType(domainSep, contractAddress)` | never transmitted |
| `tokenOrigin` (domain, contract) | `domainSep` in payload + event `contractAddress` | authenticated by the transaction |
| `name`, `ticker`, `decimals` | example keys `name`, `symbol`, `decimals` (Appendix A) | A metadata-specific MIP must define interoperable meanings and encodings. |
| `description`, `url`, `logo` | possible keys or schema-defined document fields | A metadata-specific MIP must define their meaning. |
| quadrant | `kind` byte | four values |
| `privacy` | not defined | a trait an issuer MAY emit |
| `contractToken.standard`, circuit names | not defined | a trait an issuer MAY emit; a schema MIP could standardize |
| `bridge` | not defined | a trait an issuer MAY emit |
| attestation, sequence number | chain execution and event order | These establish a contract's declaration and its order, not third-party endorsement. |
| governance / review | out of scope | the curation layer an off-chain registry adds on top |
