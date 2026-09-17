---
MIP: xxxx
Title: On-Chain Token Metadata Emission (`TokenMetadata` Events)
Authors:
  - Edward Alvarado <edward.alvarado@midnight.foundation>
  - Sebastien Guillemot (@SebastienGllmt) <sebastien.guillemot@midnight.foundation>
Status: Draft
Category: Standards
Created: 2026-09-17
Requires: none
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

Midnight has two kinds of contract-issued tokens.
A *native* token is a 32-byte color derived from the issuing contract's address and a 32-byte domain separator; it moves as protocol-level UTXOs, and nothing on chain says what the color means.
A *ledger* token ([MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) style) is a balance in the contract's own state; it has no color, never appears in a transaction's mint effects, and keeps its `name`/`symbol`/`decimals` in state fields that only a client holding that contract's compiled artifact can decode, and even then only on trust, since nothing on chain proves the artifact matches the deployed contract.
In neither case can a wallet or explorer that encounters an unfamiliar token learn what it is from the chain alone.
Wallets show raw hex or nothing, and every application invents its own lookup table.

This MIP specifies **how a contract publishes metadata about its own tokens on chain**, using the contract event mechanism from [MIP-0002](./mip-0002-public-contract-log-emission.md).
A contract emits one `Misc` event, named `mip-xxxx:token-metadata[v1]` and referred to here as a `TokenMetadata` event, per `(domainSep, kind, key, value)` tuple, with a fixed 256-byte payload.
A consumer folds these events, last write wins, into a table keyed by `(contract address, domainSep, kind)`.
The emitting contract is the sole authority for its own tokens; because the color is derived from `(domainSep, contractAddress)`, no contract can describe another contract's token.
The design follows [EIP-7496 (NFT Dynamic Traits)](https://eips.ethereum.org/EIPS/eip-7496): fixed-width keys, opaque values, updates as later events.

**This MIP defines the transport, not the schema.**
It fixes the bytes on the wire and the rules for interpreting them; it does not require any particular key to be present.
A set of common keys (`name`, `symbol`, `decimals`, `tokenUri`, `metadata`) is proposed as an *informative* convention in [Appendix A](#appendix-a-suggested-well-known-keys-informative) so that independently written consumers agree on the fields most tokens will want.

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
The `name()` path is therefore not merely inconvenient for the general case; it is unverifiable in every case.
MIP-0014 recognizes the general-case gap and delegates it to an off-chain registry ("PR #104") keyed by color, with a mandatory `(domain, contractAddress)` recomputation check.

[MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) account-model tokens have the same shape: metadata lives in ledger fields that only a schema-aware client can read.

### Why an off-chain registry alone is not enough

The [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) proposes a CIP-26-style off-chain registry. That is a reasonable design for curated, attested, human-reviewed metadata, and this MIP does not argue against building one.
But an off-chain registry cannot be the *only* layer, for reasons the MPS itself surfaces:

- **An off-chain entry cannot prove who wrote it.** A registry entry claims to speak for a contract's issuer, so the registry has to add signed attestations, sequence numbers, a validating server and a governance process for deciding which signing keys are legitimate, all to establish that the issuer really wrote the entry. But there is nothing on chain for such an attestation to anchor to. A `ContractDeploy` records no creator; the contract's maintenance authority names the committee that may *upgrade* the contract, not who deployed it, and it may be empty; the DUST that paid the deployment fee proves nothing, since anyone may pay for anyone's transaction. The only party that can demonstrably speak for a contract is the contract itself, by executing. An emitted event is exactly that, and it needs no attestation because the transaction already is one.
- **Per-token lookup leaks a shielded holder's portfolio.** If a wallet asks a metadata server about the colors it holds, the server learns which shielded tokens that wallet holds, which is precisely what shielding is meant to hide. The off-chain design can only mitigate this by having the wallet download the whole registry or pad its queries with decoy colors. On-chain events are already part of what every wallet syncs from the chain, so there is no third party to ask and nothing extra to leak.
- **Discovery.** A registry "maps known colors to metadata" and is explicitly "not a search engine". An event stream is discovery by construction: the set of described tokens is the set of contracts that emitted.
- **Registration is a bottleneck.** Every issuer must find, understand and submit to the registry. A contract that emits at deployment has registered by existing.
- **The long tail.** Community tokens, collection pieces, test tokens and LP shares will never be curated. They still need a name.

What an off-chain registry adds, and what this MIP deliberately does not attempt, is *curation*: deciding which of several contracts calling themselves "USDC" is the real one, hosting large logos, recording cross-chain bridge mappings, and attaching third-party trust signals.
The two layers compose: on-chain events are the authoritative, issuer-written base; a registry overlays verification badges, allowlists and rich media, and can itself be seeded from the events.

### Why events, not contract state

Compact contracts already have a public event mechanism ([MIP-0002](./mip-0002-public-contract-log-emission.md), live on Stagenet).
Events are the right substrate for metadata because:

1. **State layout is unverifiable; an event's layout is fixed by this MIP.** The hardest problem with reading metadata out of contract state is not access but proof. Nothing on chain declares what a contract's state fields mean, so a consumer looking at a deployed contract cannot verify that the first field is the name rather than something else. Holding the source code does not help, because nothing binds the deployed contract to that source. An event sidesteps this entirely: its byte layout is defined here, not by the contract, and the event says `name` in its own key field. The consumer needs no knowledge of the contract's state shape and takes nothing on trust except what the chain already verified, namely that this contract's circuit executed and produced these bytes.
2. **They are schema-free for the consumer.** An indexer decodes a `Misc` event by name and byte layout, with no compiled artifact and no per-contract knowledge.
3. **They are an append-only history.** Renames, trait updates and corrections are later events; consumers fold them last-write-wins and can show history.
4. **They cost nothing at rest.** Events are not consensus state ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Event Lifetime"); they do not grow the contract state that every node carries.
5. **The pipeline exists.** `emit` → `Log` opcode → `VersionedLogItem` → indexer `contractEvents` / `MiscContractEvent` is shipping. This MIP adds one convention on top and requires no ledger, compiler or indexer change.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Scope

**Normative** in this MIP: the event envelope [1], the payload layout [2], the `kind` byte [3], token identity [4], key/value handling rules [5], emission rules [6], consumer rules [7] and versioning [8].

**Informative** in this MIP: the suggested well-known keys ([Appendix A](#appendix-a-suggested-well-known-keys-informative)), the circuit-cost notes ([Appendix B](#appendix-b-circuit-cost-informative)), the mapping to EIP-7496 and the Token Registry MPS ([Appendix C](#appendix-c-mapping-to-eip-7496-and-the-token-registry-mps-informative)), and the reference module and contracts ([Implementation Example](#implementation-example)).

This MIP does not require any specific key to be emitted.
A contract that emits zero `TokenMetadata` events is not non-conforming; it is simply undescribed.
A contract that emits only keys of its own invention is fully conforming.

### Terminology

- **Color**: the 32-byte token type the ledger and wallets see: `tokenType(domainSep, contractAddress)` from the Compact standard library. Only *native* tokens (minted via `mintShieldedToken` / `mintUnshieldedToken`) have a color.
- **Domain separator (`domainSep`)**: the 32 bytes that identify a token *within* a contract; the ERC-1155 `id` analogue. For a native token it is exactly the value passed to the mint primitive. For a ledger token it is any 32 bytes the contract chooses.
- **Native token**: value held as protocol-level UTXOs (shielded Zswap coins or unshielded UTXOs). The issuing contract mints; the protocol moves.
- **Ledger token**: value held as balances in the contract's own state (the [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) / OpenZeppelin `FungibleToken` model). No mint effect ever appears in a transcript; no color exists.
- **Declaration**: a `TokenMetadata` event: a *claim* made by a contract about its own token.
- **Observation**: a mint effect in a transaction transcript: a *fact* about which `(domainSep, kind)` a contract has actually minted. Only native kinds can be observed.
- **Consumer**: any party folding `TokenMetadata` events into a token table: an indexer, a wallet, an explorer.

### 1. The event

A `TokenMetadata` event is a [MIP-0002](./mip-0002-public-contract-log-emission.md) `Misc` event with:

```
Misc {
  name:    pad(32, "mip-xxxx:token-metadata[v1]")   // Bytes<32>, NUL-padded
  payload: <the 256 bytes of [2]>        // Bytes<256>
}
```

The event name is namespaced by this MIP's number so that no other contract's `Misc` event can collide with it, following the ledger's own `midnight:derive_token` style of lowercase, colon-separated labels, and it carries its layout version in square brackets the way the ledger's own serialization tags do (`impact-versioned-log-item[v1]`, `midnight:zswap-memo[v1]`).
`xxxx` is a placeholder for the four-digit number assigned when this MIP is merged; the final string is fixed at that point and this document is updated once.
Throughout this document "a `TokenMetadata` event" means a `Misc` event carrying this name.

`Misc` is the Compact standard library's catch-all event type (`LogEventType::Misc`, `onchain-vm` event tag 10); its `payload` field is fixed at `Bytes<256>` by MIP-0002.

A consumer MUST accept an event as a `TokenMetadata` event if and only if:

1. its event type is `Misc`, and
2. `name == pad(32, "mip-xxxx:token-metadata[v1]")`, that is, the bytes `0x6d69702d787878783a746f6b656e2d6d657461646174615b76315d` followed by 5 NUL bytes (to be recomputed for the assigned number), and
3. its payload is exactly 256 bytes.

Any other event MUST be ignored by a `TokenMetadata` consumer.
An event that satisfies (1) and (2) but whose payload fails validation under [2], [3] or [5] MUST be recorded as **rejected** and MUST NOT be applied.
Consumers SHOULD surface rejected events for diagnostics.

A Compact **constructor cannot emit**, directly or through a circuit it calls.
A conforming contract that wants its metadata published at deployment therefore exposes a circuit (conventionally `publishMetadata()`) that the deployer calls immediately after deployment (see [6.7]).

### 2. Payload layout

The payload is 256 bytes, big-endian, with no padding between fields:

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 32 | `domainSep` | The token within the contract. For a native token, exactly the value passed to `mintShieldedToken` / `mintUnshieldedToken`. For a ledger token, any 32 bytes the contract chooses (e.g. `pad(32, "acme:gold")`). |
| 32 | 1 | `kind` | See [3]. |
| 33 | 32 | `key` | UTF-8 key name, NUL-padded. Compared after trimming trailing NULs [5.1]. |
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
| `0` | opaque bytes | none; consumers surface as hex |
| `1` | UTF-8 string | `value[0..val-len]` MUST be valid UTF-8 |
| `2` | unsigned integer, big-endian | `1 ≤ val-len ≤ 16`; no leading-zero requirement |
| `3` | UTF-8 JSON | valid UTF-8; the parse rule (object, array, part) is the key's to state |
| `4` | UTF-8 URI | valid UTF-8, parses as an absolute URI |
| `5` to `255` | reserved | MUST reject the event |

Type validation is part of transport validation: an event whose `value` fails the rule for its declared `val-type` MUST be rejected.
A consumer MUST NOT reinterpret a value under a type other than the one declared; if a well-known key expects `val-type` `2` and the event carries `1`, the projection fails (see [5.3]) but the trait is still stored as declared.
Values `5` to `255` are reserved for future value encodings an amendment may define.

#### 2.2 Validation

- `val-len > 189` MUST reject the event.
- A reserved `val-type` MUST reject the event.
- `value` failing its `val-type` rule MUST reject the event.
- Bytes of `value` at or after `val-len` MUST be ignored by consumers. Emitters SHOULD set them to NUL.
- A `key` consisting entirely of NUL bytes (empty key after trimming) MUST reject the event.

### 3. The `kind` byte

The `kind` byte takes exactly one of four values. Each value fixes two attributes of the token: its **privacy** (whether the value carries the shielded or the unshielded tag) and its **storage** (whether the value lives in protocol-level UTXOs minted by `mintShieldedToken` / `mintUnshieldedToken`, or in balances kept in the contract's own state).

| `kind` | Privacy | Storage | Example | Has a color? |
|---|---|---|---|---|
| `0` | unshielded | native | MIP-0014 native unshielded token | yes |
| `1` | shielded | native | MIP-0011 native shielded token | yes |
| `2` | unshielded | ledger | MIP-0004 / OpenZeppelin `FungibleToken` balances | no |
| `3` | shielded | ledger | contract-state balances the contract keeps confidential | no |

Any other value MUST reject the event.

**Note on `kind = 3`.** The label is the contract's own description of how it keeps its balances; nothing on chain, and nothing in this MIP, verifies that those balances are actually confidential. No standard currently defines a confidential contract-state token, and no reference contract exercises this kind. Consumers MUST treat `3` as purely informative and MUST NOT present a token as private on the strength of it. The value is kept so that a future standard for such tokens has a kind to use without a wire change.
Values `4` to `255` are reserved for future token categories that an amendment to this MIP may define; no other use of the byte (flags, versioning) is planned, since the event name already carries the version and per-token flags belong in ordinary keys.

A color exists only for native kinds (`0` and `1`).
A consumer MUST NOT derive or display a color for a ledger kind (`2` or `3`).

### 4. Token identity

A token is identified by the triple **`(contractAddress, domainSep, kind)`**.

- `contractAddress` is taken from the event's own `contractAddress` field in the indexer/ledger event record, never from the payload [6.1].
- `domainSep` is payload offset 0.
- `kind` is the full byte [3]. Each of the four kinds is a distinct token identity, even under one `domainSep`:
  - A contract MAY mint the same `domainSep` both shielded (kind `1`) and unshielded (kind `0`). The ledger keeps the two apart by tag, not by value: they share one color but are two token types, and a consumer shows two rows sharing one color.
  - A contract MAY hold the same `domainSep` both as contract-state balances (kind `2` or `3`) and as native UTXOs (kind `0` or `1`). A [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md) token is exactly this: balances in state, converted on demand to shielded or unshielded UTXOs under the token's `domain`. It is one asset in up to three representations, and each representation is its own row.

A contract describes each `(domainSep, kind)` it wants described separately, and a consumer MAY link rows that share `(contractAddress, domainSep)` as representations of one asset.

**`domainSep` for ledger kinds.** For a native kind, `domainSep` is grounded: it is the mint argument and the color is derived from it. For a ledger kind nothing on chain ties `domainSep` to any balance structure; it is a label the contract chooses to identify one balance book. Two rules follow:

- A contract that holds the same asset both in state and natively (MIP-0004 style) MUST use its native `domain` as the ledger `domainSep`, so that the rows link.
- A pure ledger contract uses any stable 32 bytes per balance book. Several ledger `domainSep`s from one contract are several declared balance books (the ERC-1155 shape in state); the chain cannot corroborate any of them, and consumers present them as such [7.2].

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
Keys SHOULD be valid UTF-8; consumers MUST NOT reject a key solely for not being valid UTF-8 (they MAY display it as hex).

#### 5.2 Values are typed bytes

At the transport level a value is `val-len` bytes tagged with a `val-type` [2.1].
The type says how to *read* the bytes; this MIP assigns no *meaning* to any key.
A consumer MUST store every accepted `(key, val-type, val-len, value)` verbatim against the token identity, regardless of whether it recognizes the key, and SHOULD render unknown keys according to their `val-type` (text for strings, a number for integers, hex for opaque bytes). This is EIP-7496's `getTraitValue`, and it is what makes the standard forward-compatible: new keys need no protocol change.

#### 5.3 Well-known keys are a convention, not a requirement

[Appendix A](#appendix-a-suggested-well-known-keys-informative) proposes encodings and validation rules for a small set of common keys (`name`, `symbol`, `decimals`, `metadata`, `metadata/<n>`, `tokenUri`).
These are informative.
However, to keep independently written consumers interoperable:

- An emitter that uses one of the Appendix A key names SHOULD use the encoding Appendix A gives for it.
- A consumer that projects an Appendix A key into a dedicated column SHOULD apply Appendix A's validation for that key, and on failure SHOULD keep the raw trait [5.2] and flag the projection as invalid rather than reject the event.

A future MIP MAY promote some or all of Appendix A to normative status without changing the wire format.

#### 5.4 Values longer than 189 bytes

A single event carries at most 189 value bytes.
Longer values are the emitter's problem to split and the consumer's to reassemble; this MIP fixes only one convention for doing so, the `metadata/<n>` part scheme in Appendix A, and does not mandate it.

### 6. Emission rules

#### 6.1 Authority

The emitting contract is the only authority for `(its own address, domainSep, kind)`.
A consumer MUST take the contract address from the event record's `contractAddress`, never from the payload.
Because a native token's color is derived from `(domainSep, contractAddress)`, no contract can describe another contract's color: an event from contract A about `domainSep` X describes `tokenType(X, A)`, which is A's token by construction.

#### 6.2 Last write wins

Per `(contractAddress, domainSep, kind, key)`, the most recent accepted event is the current value.
Ordering is by block height, then by the event's position in the transaction's evaluation order (the indexer's monotonic event `id` provides this).
Earlier values are history, not truth; consumers MAY retain and display history.

There is no delete.
An emitter that wants to "unset" a key emits it with `val-len = 0`; a consumer MUST treat `val-len = 0` as "present, empty" at the transport level. Appendix A keys define their own handling of empty values.

#### 6.3 Observation is independent of declaration

A mint effect in a transcript is a fact; a `TokenMetadata` event is a claim.
Because the full `kind` is part of the identity [4], the two never contradict each other; they populate rows independently:

- A mint of `(domainSep, kind 0 or 1)` creates or confirms that native row whether or not anything was declared for it. A declaration can never hide or relabel a mint.
- A declaration for `(domainSep, kind)` populates exactly that row and no other. Declaring kind `2` says nothing about kind `0`; declaring kind `1` for a `domainSep` only ever minted as kind `0` describes a token that has not been minted yet, not the one that has.
- A ledger declaration (kind `2` or `3`) has no observation that could ever confirm it. It stays a declaration.

#### 6.4 Describing an unminted token is legal

A ledger kind has no mint at all and can only ever be declared.
A native kind MAY be described before, or without, its first mint.
Consumers represent both as the **declared** state [7.2].

#### 6.5 No registration

Nothing is registered with anyone.
A consumer discovers described tokens from the transactions themselves: every contract call's transcript publicly states how many `log` operations it ran, and the consumer fetches the events of exactly those calls [7.3].

#### 6.6 Disclosure

Everything that reaches `emit` is public.
Contracts MUST pass `disclose(...)` for any witness-derived value, exactly as for any other public write; the Compact compiler enforces this.
Emitters SHOULD NOT emit anything they would not write to public ledger state.

#### 6.7 Publication and access control

Because constructors cannot emit, a contract that wants deployment-time metadata exposes a circuit the deployer calls after deployment.
Whether that circuit is callable once (a publish-once guard), owner-gated, or open is the contract's own policy; this MIP takes no position, with one caveat: a circuit that emits `TokenMetadata` for a token whose metadata is meant to be stable SHOULD be access-controlled, because anyone who can call it can rename the token (see Security Considerations).

### 7. Consumer rules

#### 7.1 Acceptance and rejection

A consumer applies [1] to recognize a `TokenMetadata` event, then [2], [3] and [5] to validate it.
Rejected events MUST NOT be applied and SHOULD be recorded with the reason.

#### 7.2 Token states

Folding observations (mint effects) and declarations (events) yields, for every `(contractAddress, domainSep, kind)`, one of three states a consumer SHOULD distinguish:

| State | Observed mint? | Any declaration? | Meaning |
|---|---|---|---|
| **observed** | yes | no | A native kind with a color that has been minted, but nobody has described it. This is every custom token today. |
| **declared** | no | yes | Described, never minted natively. Every ledger kind is permanently here, since nothing on chain can corroborate it; a native kind is here until its first mint. |
| **described** | yes | yes | A native kind, minted and described. |

Only native kinds can reach **described**.
A consumer SHOULD make the difference visible: a **described** row's name is a claim about a token the chain has seen; a **declared** row's name is a claim about a token the chain has not, and for ledger kinds never will.

A contract that declares only kind `2` for a `domainSep` it then mints as kind `0` produces two rows: an **observed** kind `0` row with no name, and a **declared** kind `2` row with one. That is an accurate picture, not an error: the contract described its balance book and not its UTXOs. A consumer MAY hint that the rows share a `domainSep`.

#### 7.3 Reading events from transactions

Event *contents* cannot be read statically from a transaction: `emit` compiles to the VM's `log` opcode and its operand comes from the stack at execution time.
What a transaction does state publicly is **how many `log` ops each contract call's transcript runs**.
A consumer that does not simply trust an indexer therefore:

1. scans every transaction for `ContractDeploy` / `ContractCall` actions and for the mint effects of each call's transcripts (the observed facts);
2. counts `log` ops in the guaranteed transcript and in the fallible transcript of every successful segment;
3. for each call with at least one `log` op, fetches that call's events (from an indexer, or from its own ledger replay) and compares the count with what the transcript promised before applying anything.

Mints in a guaranteed transcript count when the transaction succeeded or partially succeeded; mints in a fallible transcript count only when that intent's segment succeeded.
Events follow the same rule as the transcript that emitted them.

#### 7.4 Untrusted input

Every byte of a `TokenMetadata` payload is attacker-controlled.
Consumers MUST bound-check all offsets and lengths, MUST treat `value` as untrusted for any parser they apply to it (UTF-8, JSON, URL), and MUST NOT dereference a URL found in a value without the same precautions they would apply to any remote content (see Security Considerations).

### 8. Versioning

**The event name is the version.**
The bracketed suffix is the layout version, in the same form the ledger uses for its serialization tags.
A future, incompatible layout uses a new name and never a reinterpretation of `mip-xxxx:token-metadata[v1]`: an amendment within this MIP bumps the bracket (`mip-xxxx:token-metadata[v2]`), and a superseding MIP gets a fresh name for free (`mip-yyyy:token-metadata[v1]`).
A consumer that only knows `[v1]` ignores the other names; a consumer that knows several keeps them apart.

Within version 1, new keys may be introduced freely: unknown keys are already required to be stored verbatim as traits [5.2], so a key registry can grow without any wire change.
Changes to Appendix A are amendments to this MIP, not superseding proposals.

### Out of scope

- **Which metadata a token should expose.** This MIP fixes the transport. Required fields, per-asset-class schemas (fungible, NFT, RWA), localization and media formats belong in separate, layered proposals that use this transport.
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

Appendix A exists because the alternative, every consumer inventing its own encoding for `decimals`, is exactly the fragmentation this MIP is meant to end. SHOULD-level guidance on the common keys gets interoperability without pretending the transport knows what a token is.

### Why one event per `(key, value)` rather than one blob per token?

- The `Misc` payload is fixed at 256 bytes by MIP-0002. A single blob would need a compression or chunking scheme before it could hold `name` + `symbol` + `decimals` + a URL.
- Per-key events make updates cheap and precise: renaming a token is one event, not a re-emission of everything.
- Per-key folding is what EIP-7496's `TraitUpdated` does, and what an indexer wants to store anyway.
- The `metadata/<n>` part scheme (Appendix A) exists precisely for the case where a blob *is* wanted.

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
It also tells a consumer, per row, whether to expect a color at all and whether a mint could ever corroborate the declaration.

### Why observations and declarations are kept independent?

Events are claims ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Event trust model").
Mint effects are facts the ledger verified.
A consumer that let a claim override a fact would let a contract hide its own mints from a token table, so a declaration is never allowed to create, remove or relabel a native row; only a mint does that.
Detecting contradictions between the two (a ledger declaration for a `domainSep` that was minted natively) and flagging the row was considered and rejected.
With the full `kind` in the identity there is nothing to contradict: the declaration and the mint describe different rows, and the honest picture is simply that one of them has a name and the other does not.
The **declared** versus **described** distinction [7.2] carries the remaining warning: it tells the user whether the chain has ever seen the token a name is attached to.

### Why on-chain `name`/`symbol` despite MIP-0014's rejection of it?

MIP-0014 rejects on-chain `name`/`symbol` because "ledger state is public and untrusted for identity, so on-chain `name`/`symbol` would invite impersonation without removing the need for a derivation check."

Both halves are true and neither is an argument against this design:

- **Impersonation is medium-independent.** Anyone can call their token "USDC" in an off-chain registry too; CIP-26 relies on Cardano Foundation review to sort it out. What this MIP adds is that the *claim is cryptographically bound to the claimant*: the event's `contractAddress` is authenticated by the transaction, and the color a consumer derives from it cannot be the color of anyone else's token. That is strictly more than an unattested registry entry offers, and it is the same binding MIP-0014's own registry path requires the wallet to recompute.
- **The derivation check is not removed; it is automatic.** A consumer *only ever* derives the color from `(domainSep, contractAddress)`; there is no transmitted color to check against. The check MIP-0014 mandates is the only way a color enters the table at all.

What on-chain metadata does not do, and MIP-0014 is right that nothing on chain can do, is tell a user *which* USDC to trust. That is curation, and it is out of scope here by design.

### Why events rather than a metadata field in contract state?

The decisive reason is verifiability.
A `name` ledger field is only a name because the contract's source says so, and nothing on chain binds a deployed contract to any source.
A consumer therefore cannot prove that the field it reads is the name, even with the code in hand.
An event carries its own meaning in a layout this MIP fixes: the key field says `name`, and the only trust involved is the chain's own verification that the contract executed and emitted it.

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
One byte of type tag lets any consumer render any trait sensibly (a string as text, an integer as a number) without knowing the key, and lets well-known keys carry a machine-checkable type instead of a documented convention.
It costs one byte of value width, and the value width was never the binding constraint: anything that does not fit in 189 bytes did not fit in 190 either and goes through the `metadata/<n>` part scheme regardless.
The tag is deliberately a closed enum with reserved values, so new encodings are an amendment, not a free-for-all.

### Alternatives considered

- **Off-chain registry only (the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) as proposed).** Rejected as the sole layer: it needs attestation infrastructure and governance to provide the provenance that on-chain emission provides for free, and it cannot solve privacy-safe lookup or discovery. Retained as a complementary curation layer.
- **`tokenUri` pointer only (ERC-721 style).** Rejected as the sole mechanism: it moves every field behind an HTTP fetch, which is a privacy leak for shielded holders and a liveness dependency. Retained as an Appendix A key for tokens that want it.
- **A protocol-level `LogEventType::TokenMetadata` variant.** Deferred; see above.
- **JSON in every event.** Rejected: 189 bytes is too small for most JSON objects, and it makes the common fields more expensive to emit and decode than fixed bytes. Available via the `metadata` / `metadata/<n>` keys for those who want it.

## Path to Active

### Acceptance Criteria

- At least one independent consumer (indexer, explorer or wallet) folds `TokenMetadata` events into a token table and correctly renders all three states of [7.2] against the published fixtures.
- At least one issuer other than the author adopts the convention on a public network.
- The Compact module is published in a form issuers can import (this repository or an ecosystem library such as OpenZeppelin Compact Contracts).
- Community review through the MIP process, including reconciliation with the authors of the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104) on the on-chain/off-chain boundary.

### Implementation Plan

1. **Reference module and contracts**: done; see [Implementation Example](#implementation-example).
2. **Reference deployment**: done on Stagenet (see below); repeat on Preprod when the events pipeline is available there.
3. **Consumer reference**: a byte-exact decoder and the simulator/Stagenet fixture corpus are published; propose `TokenMetadata` decoding to at least one public explorer.
4. **Library adoption**: propose the module (or an equivalent) to OpenZeppelin Compact Contracts as an optional extension of `NativeShieldedToken`, `NativeShieldedTokenFamily` and `FungibleToken`.
5. **Upgrade template**: publish the added-circuit template for pre-v9 contracts and exercise it on Stagenet by upgrading a contract deployed without events, per [Upgrade Path for Existing Contracts](#upgrade-path-for-existing-contracts).
6. **Schema follow-ups**: a fungible-token schema MIP promoting Appendix A's core keys to normative for MIP-0011/0014/0004 tokens, and an NFT content-metadata MIP (per the discussion on PR #104), both using this transport.

## Backwards Compatibility Assessment

No protocol, compiler or indexer change is required; this MIP is a convention over [MIP-0002](./mip-0002-public-contract-log-emission.md)'s existing `Misc` event.
No deployed contract is affected: contracts that do not emit `TokenMetadata` are simply undescribed.

Existing contracts deployed before ledger v9 cannot emit as deployed; [Upgrade Path for Existing Contracts](#upgrade-path-for-existing-contracts) describes how their maintenance authority adds an emitting circuit without redeployment. Only contracts with an empty or unreachable maintenance authority are left to an off-chain registry.

Adopting the convention is additive to [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md), [MIP-0011](./mip-0011-native-shielded-token.md) and [MIP-0014](./mip-0014-native-unshielded-token.md): a conforming token keeps its `name()`/`symbol()`/`decimals()` circuits and additionally emits the same values as events. Nothing in those standards is changed.

Compact currently cannot serialize an `Opaque<"string">` into a circuit, so a contract that stores `name` as `Opaque<"string">` for the MIP-0011 circuits must also hold a byte form for the event path (the reference contracts take both at construction). This is a toolchain limitation, not a design choice, and would disappear if the compiler gained string-to-bytes conversion.

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
3. **Call the new circuit** once. From that transaction on, the token is **described** [7.2].

Because the maintenance authority is a committee with a threshold, the upgrade is exactly as permissioned as any other change the issuer already reserved the right to make.
No new trust is introduced: a consumer folding the resulting events applies the same rules as for a contract that emitted from day one, and the authority rule [6.1] holds because the event still comes from the token's own contract.

### Two practical points for the new circuit

- **Compact cannot serialize `Opaque<"string">` into a circuit.** Contracts that store `name` and `symbol` as `Opaque<"string">` for their MIP-0011 / MIP-0014 / MIP-0004 circuits cannot emit those fields from state. The added circuit takes the byte form as arguments (owner-gated, called once) or bakes it in as compile-time literals, which is also the cheap shape to prove ([Appendix B](#appendix-b-circuit-cost-informative)). The values SHOULD match what `name()` / `symbol()` / `decimals()` return.
- **`kernel.self()` and the color.** The new circuit reads the contract's existing `domain` from state and emits it as `domainSep`; it does not need to compute or emit the color, which a consumer derives [4].

### Limits

- A contract whose maintenance authority is **empty** cannot be upgraded by anyone. Such a token can never be described on chain; an off-chain registry remains the only path for it (see [Out of scope](#out-of-scope)).
- A contract whose authority committee is no longer reachable is in the same position in practice.
- The upgrade adds a circuit; it cannot change what the contract already does. A token whose issuance is finished and whose authority is retired stays undescribed by design.

### Timeline

The upgrade can be prepared before the network upgrade (compile the circuit, agree the metadata, line up the authority signatures) and executed immediately after it.
Because the events pipeline ships with ledger v9 itself, there is no second dependency to wait for: the day the network runs v9, every upgradable token can be described.
The reference repository will publish the added-circuit template alongside the from-scratch templates.

## Security Considerations

### Impersonation

Any contract can emit `name = "USDC"`.
This MIP binds the claim to the emitting contract's address and, for native tokens, to a color no other contract can produce; it does not and cannot say whether that contract is the one a user means.
Consumers MUST display the contract address (or the derived color) alongside any name, and SHOULD integrate a curation signal (allowlist, registry attestation, user confirmation) before presenting a name as trusted.
This is identical to the situation on every EVM chain and to an unattested CIP-26 entry.

### Unauthorized updates

A `setMetadata`-style circuit with no access control lets anyone rename a token.
Emitters SHOULD gate metadata-emitting circuits (the reference contracts use OpenZeppelin `Ownable`) or make them publish-once.
Consumers MAY surface the history of a key so that a hostile rename is visible.

### Spam and resource use

Events are metered by the existing `Log` opcode fee model and compete for the per-block `bytes_written` budget ([MIP-0002](./mip-0002-public-contract-log-emission.md), "Fee Metering").
No new spam vector is introduced.
A consumer's storage is bounded by what emitters pay for; consumers MAY cap traits per token or bytes per contract as a local policy.

### Untrusted payloads

Every payload byte is attacker-controlled [7.4].
Decoders must be bound-checked; UTF-8, JSON and URL parsers applied to `value` must be hardened against malformed input; `metadata/<n>` reassembly must bound total size (Appendix A caps it at 16 parts).
A `tokenUri` or `image` URL MUST NOT be fetched automatically by a wallet on behalf of a shielded holder without considering that the fetch reveals interest in that token to the URL's host.

### Declarations that the chain cannot corroborate

A ledger declaration is a claim with no possible on-chain corroboration: a contract can declare a balance book under any `domainSep`, or several, without any of them existing in its state.
[7.2] keeps such rows permanently **declared**, and consumers SHOULD show that status rather than present a declared name with the same weight as a described one.
A contract that describes its balance book but not its UTXOs (the "Ledger Liar" row of the reference set) yields an undescribed observed row next to a declared one; the mint is never hidden.

### Disclosure

`emit` is a disclosure site and the compiler enforces that emitted values are disclosed.
No new leakage path is introduced; an issuer that emits a value has chosen to make it public.

### Proving cost as a footgun

Emitting runtime-built payloads is expensive (Appendix B).
This is not a security issue for the network, since the emitter pays, but an issuer who packs many runtime-built events into one circuit may find it slow or impractical to prove on ordinary hardware.
It is called out so that the fixed-width layout is not mistaken for a claim that emission is cheap.

## Implementation Example

Reference implementation: [`acedward/mip-erc7496-midnight-contracts`](https://github.com/acedward/mip-erc7496-midnight-contracts) (Apache-2.0). The repository tracks this MIP; its `main` branch is the current reference.

### Components

- **`contracts/TokenMetadata.compact`**: the module a conforming contract imports. Two circuits: `emitTokenMetadata(domainSep, kind, key, valType, valLen, value)` emits one event with the [2] layout; `emitStandardFields(domainSep, kind, name, nameLen, symbol, symbolLen, decimals)` emits the three Appendix A core fields. Three constants: `KIND_UNSHIELDED()` = 0, `KIND_SHIELDED()` = 1, `KIND_LEDGER_FLAG()` = 2.
- **Reference contracts**, one per value domain, each composing an OpenZeppelin token module (where one exists), OpenZeppelin `Ownable` for update gating, and the module above:
  - `NativeShieldedToken.compact`: one static domain, kind 1 (MIP-0011 Fungible profile + events).
  - `NativeUnshieldedToken.compact`: one static domain, kind 0 (MIP-0014 shape + events).
  - `NativeDualToken.compact`: one domain minted both shielded and unshielded; publishes six events across two circuits.
  - `ShieldedCollection.compact`: one address, one domain per piece (MIP-0011 Family profile + per-piece events); the EIP-7496 shape.
  - `LedgerToken.compact`: OpenZeppelin `FungibleToken` balances, kind 2; the case only events can make visible.
  - `contracts/generated/*.compact`: the same contracts with all metadata as compile-time literals (see Appendix B for why).
- **Consumer reference**: `test/token-metadata.ts`, a byte-exact decoder that deliberately reimplements [2] from this document rather than importing anything from the contracts.
- **Fixtures**: `fixtures/simulator/` (offline, reproducible: events, mints, color vectors, expected token rows, negative payloads) and `fixtures/stagenet/` (the same recorded from the public Stagenet indexer, including raw transaction bytes).

### Reference deployment (Stagenet)

The reference set is deployed to Midnight Stagenet: eleven contracts covering all four value domains and all three consumer states [7.2], including a token minted but never described, a token described but never minted, a dual-kind token producing two rows under one color, a collection with one `domainSep` per piece, and a contract that describes its ledger side but mints natively.

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

## Appendix A: Suggested well-known keys (informative)

These keys are a **convention**, not a requirement [5.3].
An emitter that uses one of these key names SHOULD use the encoding given here; a consumer that projects one into a dedicated column SHOULD apply the validation given here, keeping the raw trait on failure.

| Key | `val-type` | Validation | Notes |
|---|---|---|---|
| `name` | `1` string | `1 ≤ val-len ≤ 189` | Display name. For a MIP-0011/0014/0004 token SHOULD equal `name()`. |
| `symbol` | `1` string | `1 ≤ val-len ≤ 32` | Ticker. SHOULD equal `symbol()`. |
| `decimals` | `2` integer | `val-len == 1`, `value[0] ≤ 36` | Display convention only; the protocol operates on integers. SHOULD equal `decimals()`. |
| `metadata` | `3` JSON, single part | `val-len ≥ 2`, parses as a JSON object | Free-form rich metadata (`description`, `image`, `website`, …). |
| `metadata/<n>` | `3` JSON, one part of several, `n = 0, 1, …`, each `≤ 189` bytes | Parts are concatenated in `n` order and applied only when `0..max` are all present and the concatenation parses as a JSON object; `n ≤ 15` (≤ 3 024 bytes). A single part need not parse on its own. | For JSON that does not fit one event. A single-part `metadata` and a multi-part `metadata/<n>` for the same token SHOULD NOT both be emitted; if they are, the most recently completed one wins. |
| `tokenUri` | `4` URI | `val-len ≤ 189`, absolute `http(s)://` URL | The ERC-721 `tokenURI` analogue: where a metadata document for this token can be fetched. See Security Considerations on fetching. |

A well-known key carried with a different `val-type` than listed is stored as a trait [5.2] but not projected into its column.

Suggested trait names, with no encoding fixed here: `description`, `image` (a URL or a `data:` URI, usually inside `metadata`), `website`, `metadataUri` (EIP-7496's collection-level trait-definition document, distinct from the per-token `tokenUri`), `bridge` (the cross-chain mapping of the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)).

Any key not in this table is a **trait**: stored verbatim against `(contractAddress, domainSep, kind)` and rendered according to its `val-type`.

## Appendix B: Circuit cost (informative)

> **Note.** Proving-cost figures for emitting `TokenMetadata` events (constraint rows, circuit size `k`, proving-key size) are still being worked on and are not included in this draft. They will be added, together with the emission strategy the reference contracts settle on, once the numbers are final.

What is already established qualitatively:

- The standard fixes the bytes on the wire, not how a contract assembles them, and the assembly strategy dominates the proving cost. A payload built entirely from compile-time literals is cheap; a payload assembled from runtime values (ledger fields or circuit arguments) is not.
- Cost grows with the number of runtime-built events in one circuit.
- Issuers whose metadata is fixed at deployment (most tokens) SHOULD prefer literal payloads; the reference repository's `contracts/generated/` templates show the shape and emit exactly the same bytes as the parameterised templates.

## Appendix C: Mapping to EIP-7496 and the Token Registry MPS (informative)

### EIP-7496

| EIP-7496 | This MIP |
|---|---|
| `tokenId` | `domainSep` (+ `kind`) |
| `traitKey: bytes32` | `key: Bytes<32>` |
| `traitValue: bytes32` | `value: Bytes<189>` with `val-type` and `val-len`: longer, typed values, no hashing |
| trait types described in the `getTraitMetadataURI` document | `val-type` byte, in-band |
| `TraitUpdated` event | one `TokenMetadata` `Misc` event |
| `getTraitValue(tokenId, traitKey)` | the consumer's folded key/value table |
| `getTraitMetadataURI` | the `metadata` JSON inline, or a `metadataUri` trait |
| ERC-721 `tokenURI(tokenId)` | the `tokenUri` key |
| the contract is the authority | the emitting contract is the authority, enforced by color derivation |

### Fields of the [Token Registry MPS (PR #104)](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/104)

| Token Registry MPS field | Here | Notes |
|---|---|---|
| `subject` (color hex) | derived: `tokenType(domainSep, contractAddress)` | never transmitted |
| `tokenOrigin` (domain, contract) | `domainSep` in payload + event `contractAddress` | authenticated by the transaction |
| `name`, `ticker`, `decimals` | `name`, `symbol`, `decimals` (Appendix A) | `symbol` follows MIP-0011/0014 naming |
| `description`, `url`, `logo` | traits or inside `metadata` | `logo` as a `data:` URI or URL |
| quadrant | `kind` byte | four values |
| `privacy` | not defined | a trait an issuer MAY emit |
| `contractToken.standard`, circuit names | not defined | a trait an issuer MAY emit; a schema MIP could standardize |
| `bridge` | not defined | a trait an issuer MAY emit |
| attestation, sequence number | not needed | the transaction is the attestation; event order is the sequence |
| governance / review | out of scope | the curation layer an off-chain registry adds on top |
