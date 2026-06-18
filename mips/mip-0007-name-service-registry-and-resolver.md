---
MIP: 0007
Title: Midnight Name Service — Canonical Registry and Resolver Standard
Authors:
  - Midnames (midnames)
Status: Proposed
Category: Standards
Created: 2026-05-28
Requires: none
Replaces: none
License: Apache-2.0
---

<!--
 This file is part of midnight-improvement-proposals.
 Copyright (C) 2025-2026 Midnight Foundation
 SPDX-License-Identifier: Apache-2.0
 Licensed under the Apache License, Version 2.0 (the "License");
 You may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

This MIP specifies the **Midnight Name Service (MidNS)**: a canonical, on-chain registry and resolver standard that binds human-readable names under the `.night` top-level domain to Midnight accounts and to the family of addresses derived from them. It addresses the problem space defined in [MPS-0012 (Human-Readable Aliasing for Midnight Accounts)](../mps/mps-0012-account-aliasing.md).

The standard adopts a **delegated registry/resolver architecture**, modeled on DNS and on Ethereum's ENS: each name is its own contract instance that both resolves itself (holding its target address, records, and default-resolution rule) and acts as the registrar for its direct children (holding a map of child label → child resolver contract). The `.night` TLD is a single well-known root contract; resolution walks resolver pointers from that root down to the leaf. This gives global name-path uniqueness verifiable from chain state, recovery-safe ownership proven against a wallet-seed-derived commitment, and per-name extensibility (a name's contract can carry additional application logic — e.g. a DAO whose members hold subdomains).

On top of this, MidNS defines a **records (fields) model** — DNS-record-style key/value metadata per name — with a **normative reserved-key schema and operation-to-address routing table**, so a single name routes to the Shielded, Unshielded, Dust, or external-chain address appropriate to the operation. A transport-agnostic resolver interface, reference TypeScript SDK, and HTTP/CLI resolver complete the standard.

The delegated architecture is the target design; its canonical deployment is pending Midnight enabling permissionless contract deployment (see [Current Implementation](#current-implementation-transitional)). A behaviorally-equivalent single-contract implementation is live on mainnet today as a transitional measure. This MIP proposes the `.night` registry as the canonical Midnight namespace, on the same basis that every other major chain has converged on a single authoritative name service per top-level domain.

## Motivation

[MPS-0012](../mps/mps-0012-account-aliasing.md) identifies four compounding problems: a single Midnight seed deterministically produces multiple user-facing addresses (Shielded `mn_shield-addr1…`, Unshielded `mn_addr1…`, Dust `mn_dust1…`, and future external-chain addresses); multiple ecosystem providers are independently building uncoordinated naming layers; there is no authoritative on-chain primitive guaranteeing a name is bound to exactly one account globally; and there is no shared guidance for rendering names safely.

Existing Midnight capabilities do not solve this. Bech32m address encodings disambiguate address *type* but do nothing for human readability or for routing a single handle to the right address. A wallet seed's HD derivation is deterministic and recovery-safe, but it is invisible to counterparties — Bob cannot pay "Alice" without Alice first telling him which of her several raw addresses to use. And nothing at the protocol layer prevents two off-chain providers from each minting an `alice` that resolves to different accounts.

The lesson from every comparable ecosystem is that this is solved by a **single canonical name service per chain**, not by a protocol change:

| Chain | Canonical service | Steward | Uniqueness mechanism | Competing services |
|---|---|---|---|---|
| Ethereum | **ENS** (`.eth`) | True Names Ltd + ENS DAO | Registry contract + per-name resolver | Unstoppable Domains (sells *other* TLDs, never `.eth`) |
| Solana | **SNS** (`.sol`) | Bonfida → SNS DAO/token | On-chain name registry program | AllDomains (alternate TLDs) |
| Cardano | **ADA Handle** (`$handle`) | ADA Handle Inc. | Single NFT Policy ID — cryptographically one issuer | CNS (`.ada`) |
| Algorand | **contested** (`.algo`) | NFDomains (TxnLab) *and* ANS both claim `.algo` | none agreed — two registries | each other |

The healthy chains converged early on one registry per TLD, usually ecosystem-aligned, and made uniqueness the core guarantee; competitors differentiate on *other* TLDs or value-added services rather than re-issuing the same name. Notably, ENS — the most mature — uses exactly the registry/resolver delegation model this MIP adopts. Algorand is the cautionary case [MPS-0012](../mps/mps-0012-account-aliasing.md) explicitly warns about: two services claim the same `.algo` TLD, "only one can establish the source of truth," and the conflict is resolvable only by each dApp "voting" through integration. Midnight should not repeat this.

MidNS has shipped a working registry, resolver, SDK, and frontend, live on mainnet. This MIP specifies its architecture as a standard and proposes the `.night` registry as the canonical Midnight namespace, so that wallets, dApps, indexers, and explorers integrate one resolver and a user's identity is consistent across the ecosystem.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be interpreted as described in RFC 2119.

### Terminology

- **Name / domain**: a label registered under a parent, e.g. `alice` under `.night`, rendered `alice.night`.
- **TLD**: the top-level domain. For the canonical namespace this is `night`.
- **Register / resolver**: the contract instance for a single name. It *resolves* that name (its target and records) and *registers* that name's direct children. "Register" and "resolver" denote the same contract in two roles.
- **Account**: the controlling identity behind a name, represented on-chain by a 32-byte **owner commitment** that is a hash over a wallet-seed-derived secret (see [Ownership and Recovery](#ownership-and-recovery-model)).
- **Target**: the primary address a name resolves to.
- **Field / record**: a key/value string pair attached to a name, analogous to a DNS record.

### Architecture: Delegated Registry/Resolver

MidNS is a tree of contracts. Each name is an independent contract instance with two responsibilities:

1. **Resolve itself** — it holds its own `DOMAIN_TARGET`, `fields`, and `DEFAULT_FIELD`.
2. **Register its children** — it holds a `domains` map binding each direct child label to that child's `{ owner commitment, resolver contract address }`.

The `.night` TLD is the **root register**: a single well-known canonical contract whose `domains` map holds all second-level names. `alice.night` is an entry `alice → { owner, resolver_addr }` in the root contract, where `resolver_addr` is the contract instance for `alice.night`; that instance in turn registers `*.alice.night`.

This mirrors DNS delegation and the ENS registry/resolver split. Its decisive advantage over a single monolithic registry is **per-name extensibility and scalability**: each name's state lives in its own contract (no shared `Map` that grows without bound), and a name's contract MAY implement additional application logic — e.g. `dao.night` could be a DAO contract that also issues `member.dao.night` subdomains to its members.

A conforming register/resolver contract holds the following ledger state (names normative; reference contract is `leaf.compact`):

```compact
// Immutable identity of this name (set at construction).
export sealed ledger PARENT_DOMAIN: Maybe<Bytes<32>>;     // parent label
export sealed ledger PARENT_RESOLVER: ContractAddress;    // parent contract
export sealed ledger DOMAIN: Maybe<Bytes<32>>;            // this name's label

// This name's owner and resolution target.
export ledger DOMAIN_OWNER: [Bytes<32>, UserAddress];     // (owner commitment, payout/display address)
export ledger DOMAIN_TARGET: Either<ContractAddress, Either<ZswapCoinPublicKey, UserAddress>>;

// Records for this name.
export ledger fields: Map<Opaque<'string'>, Opaque<'string'>>;
export ledger DEFAULT_FIELD: Maybe<Opaque<'string'>>;

// Children registered under this name, and a reverse index by owner.
export ledger domains: Map<Bytes<32>, DomainData>;            // label -> { owner, resolver }
export ledger domains_owned: Map<Bytes<32>, Set<Bytes<32>>>;  // owner commitment -> labels

// Registrar pricing for selling children.
export ledger BUY_ENABLED: Boolean;
export ledger COIN_COLOR: Bytes<32>;
export ledger COST_SHORT: Uint<128>;   // label length <= 3
export ledger COST_MED: Uint<128>;     // label length == 4
export ledger COST_LONG: Uint<128>;    // label length >= 5

struct DomainData { owner: Bytes<32>, resolver: ContractAddress }
export enum AddressType { ContractAddr, ZswapCPKAddr, UnshieldedAddr }
```

`DOMAIN_TARGET` is an `Either<ContractAddress, Either<ZswapCoinPublicKey, UserAddress>>`, encoding the three target classes: a contract address, a Shielded coin public key, or an Unshielded user address. At construction the target is supplied as `[Bytes<32>, AddressType]` and stored in the corresponding `Either` arm.

### Resolution

Resolution walks resolver pointers from the canonical root to the leaf. To resolve `really.long.example.night`:

1. Split on `.` and drop the TLD: `[example, long, really]` (root-first).
2. Start at the canonical `.night` root contract address.
3. For each label, read the current contract's `domains` map. If the label is absent, resolution fails (name not found). Otherwise take the `resolver` contract address from the entry and make it the current contract.
4. The final contract is the resolver for the full name. Its `DOMAIN_TARGET` (decoded per its `Either` arm), `fields`, and `DEFAULT_FIELD` are the resolution result.

**Default resolution** returns the value of the field named by `DEFAULT_FIELD` if it is set and present in `fields`; otherwise it returns `DOMAIN_TARGET`. This lets an owner make a name resolve, by default, to a human-friendly record (e.g. a profile) or to a chosen address.

### Name Validity and Normalization

A label MUST satisfy the following, enforced **in-circuit** at registration:

- Length `1 ≤ len ≤ 32` bytes.
- Each of the first `len` bytes is a lowercase ASCII letter (`a`–`z`, 0x61–0x7A), an ASCII digit (`0`–`9`, 0x30–0x39), or a hyphen (`-`, 0x2D).
- The first and last bytes MUST NOT be a hyphen.
- Bytes at index `≥ len` MUST be the padding byte `0xFF`.

This is the Letter-Digit-Hyphen (LDH) rule restricted to lowercase ASCII. It is a deliberate, registry-layer phishing-resistance choice: admitting no uppercase, no Unicode, and no mixed scripts structurally eliminates the homoglyph and confusable-script attacks that [MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 4 and Open Question 9 raise — there is no Cyrillic `а` vs Latin `a` problem if only Latin `a` can ever be registered. Clients MUST lowercase and strip a trailing `.night` before submitting a label, and SHOULD reject input that does not normalize to a valid label rather than silently mangling it.

### Field-Based Address Routing

A single name MUST be able to route to the correct address for the operation being performed, abstracting over Shielded, Unshielded, Dust, and external-chain addresses ([MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 1). MidNS achieves this through `DOMAIN_TARGET` plus a set of **reserved field keys**. Conforming resolvers MUST honor the following reserved keys; any other key is freeform metadata (e.g. `avatar`, `bio`, `twitter`) and MUST NOT be used for address routing.

| Key | Value format | Meaning |
|---|---|---|
| *(primary target)* | `DOMAIN_TARGET` | The name's default binding. Typically a Shielded coin public key (`ZswapCPKAddr`); MAY be an Unshielded or Contract address. |
| `epk` | `mn_shield-epk1…` (Bech32m) | Shielded encryption public key. Combined with a `ZswapCPKAddr` target it yields the full Shielded **address** `mn_shield-addr1…`. |
| `unshielded` | `mn_addr1…` (Bech32m) | The account's Unshielded address. |
| `dust` | `mn_dust1…` (Bech32m) | The account's Dust address, for DUST sponsorship. |
| `chain:<caip-2>` | chain-native address string | An externally-derived address on the chain identified by its [CAIP-2](../mps/mps-0003-caip-support.md) identifier, e.g. `chain:eip155:1`. Reserved for the external-chain extension. |

The resolver's **operation-to-address routing table** is normative:

| Operation context | Resolver returns |
|---|---|
| Shielded transfer | Full Shielded address derived from the `ZswapCPKAddr` target and the `epk` field; if the target is already a full Shielded address, that value. |
| Unshielded operation | The `unshielded` field. |
| DUST sponsorship | The `dust` field. |
| Contract call | `DOMAIN_TARGET` when it is a `ContractAddr`. |
| External-chain deposit | The `chain:<caip-2>` field for the requested chain. |
| Unspecified / default | The `DEFAULT_FIELD` value if set, else `DOMAIN_TARGET`. |

If the field required for a requested operation is absent, the resolver MUST report "no address of that class for this name" rather than falling back to a different address class — silently substituting a Shielded address where an Unshielded one was requested is a misdirected-funds risk. The user does not select an address type; the **operation context** determines routing, exactly as envisioned in [MPS-0012](../mps/mps-0012-account-aliasing.md) Use Cases 1, 2, and 5.

This records-based design makes one name route everywhere **without** requiring the owner to pre-publish every address: an owner registers once with a sensible primary target, then adds the `unshielded`, `dust`, and `epk` fields they care about. Wallets SHOULD populate these reserved fields automatically at registration from the user's HD-derived addresses, so the routing table is fully populated by default.

### Registration

`register_domain_for(owner, domain, len, resolver)` records a new child under the calling register. It MUST:

1. Validate the label per [Name Validity](#name-validity-and-normalization).
2. Assert the label is not already present in this register's `domains` map (uniqueness within the parent).
3. Settle payment: if the caller is this register's owner (their derived commitment equals `DOMAIN_OWNER[0]`), registration is free. Otherwise `BUY_ENABLED` MUST be true, and the caller pays, in `COIN_COLOR`, `COST_SHORT` for `len ≤ 3`, `COST_MED` for `len == 4`, or `COST_LONG` for `len ≥ 5`, forwarded to the owner's `DOMAIN_OWNER[1]` address via unshielded send/receive.
4. Insert `domain → DomainData { owner, resolver }` and index it in `domains_owned`.

The `resolver` argument is the contract address of the child's own register/resolver instance, which the registrant deploys for the new name. Registration therefore consists of deploying the child contract and recording the delegation in the parent. Because a child's identity (`PARENT_*`, `DOMAIN`) is sealed at construction, a child that changes hands requires the new owner to deploy a fresh resolver and the parent to re-point the delegation (`set_resolver`); collapsing this into one transaction is a planned V2 improvement.

### Owner Operations

A conforming register/resolver MUST provide, each gated by the ownership check (`assert_is_owner`):

- `update_domain_target(new_target)` — change this name's resolution target.
- `update_default_field(maybe_key)` — set/clear the default-resolution field.
- `add_multiple_fields(kvs)` (batch ≤ 10), `clear_field(k)`, `clear_all_fields()` — manage records.
- `update_color(c)`, `update_costs(short, med, long, enabled)` — configure this name's registrar pricing and `BUY_ENABLED`.
- `change_owner(new_owner, new_address)` — reassign this contract's owner commitment and payout address.

And acting in its **registrar** role over its children:

- `set_resolver(domain, resolver)` — re-point a child label to a new resolver contract (caller must own the child record).
- `transfer_domain(domain, new_owner)` — reassign a child record to a new owner commitment and re-index `domains_owned`.

### Ownership and Recovery Model

Ownership is proven against a commitment, not a wallet public key:

```compact
witness secretKey(): Bytes<32>;

circuit derive_public_key(): Bytes<32> {
  return persistentHash<Bytes<32>>(secretKey());
}

circuit assert_is_owner(): [] {
  assert(derive_public_key() == DOMAIN_OWNER[0], "Not the domain owner");
}
```

`DOMAIN_OWNER[0]` stores `derive_public_key()`. Every owner-gated circuit recomputes it from the caller's witness secret inside the ZK proof and asserts equality. Implementations MUST NOT gate ownership on `ownPublicKey()`: it is a witness supplied by the frontend and is not bound to the proof, so a check against it is bypassable (the same hazard documented in [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md)). Implementations SHOULD additionally bind a fixed domain-separator tag into the hash (e.g. `persistentHash([pad(32, "midnight.domains"), secretKey()])`) to scope the secret to this name service and prevent cross-application replay.

Because `secretKey()` is derived from the wallet seed, ownership is **recovery-safe** ([MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 3): restoring the wallet from its BIP-39 mnemonic reproduces the same secret, so the same names remain controllable. No name becomes re-registrable while the owner is offline, because uniqueness is keyed on the name within its canonical parent, not on any session or device state.

### Uniqueness Guarantee

A full name path is globally unique because (a) there is exactly one canonical `.night` root contract, and (b) each delegation step looks up a label in exactly one parent's `domains` map, which `register_domain_for` guarantees is collision-free. Resolution from the canonical root is therefore deterministic, and the binding `name path → account/target` is verifiable from chain state alone, satisfying [MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 2. Uniqueness depends on the canonical root being agreed; that agreement is what this MIP proposes.

### Resolver Interface

The resolver is **transport-agnostic**; on-chain state is the single source of truth ([MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 5). Any client that walks delegations from the canonical root and applies the resolution and routing rules above is conforming, whether in-process (SDK), over HTTP/JSON, or via GraphQL against an indexer. A conforming resolver MUST expose at least:

- **resolve(name) → target** — apply default resolution; for a Shielded `ZswapCPKAddr` target, derive the full `mn_shield-addr1…` from the `epk` field when present.
- **resolve(name, operation) → address** — apply the routing table for a given operation context.
- **field(name, key) → value** — read a single record.
- **profile(name) → { target, owner, fields, settings }** — read the full record set and registrar settings.

The reference SDK (`@midnames/sdk`) provides these as `resolveDefault`, `resolveDomain`, `getDomainFields`, `getDomainInfo`, `getDomainProfile`, `getSubdomains`, plus write operations mirroring the circuits, and a standalone CLI/HTTP resolver. The same name MUST resolve identically across every conforming client because they all start from the same canonical root.

### Reverse Resolution

Reverse resolution (address → name) is **not** part of this version. Each register maintains `domains_owned` (owner commitment → child labels) for "what does this owner control" within one contract, but global address-to-name lookup is harder in a delegated tree: any number of registers may hold names owned by an address, and the mapping must decide which address class (Shielded, Unshielded, …) anchors it. A follow-up MIP SHOULD specify an ENS-style reverse registrar, consistent with [MPS-0012](../mps/mps-0012-account-aliasing.md) Open Question 3.

### Current Implementation (Transitional)

The delegated architecture above is the target standard. It is **not yet deployed**, because Midnight's mainnet currently permissions contract deployment, so deploying a fresh resolver contract per name is not possible. As a temporary measure, MidNS is live today as a **single equivalent contract** that holds the entire namespace in unified maps keyed by a numeric Zone ID (`name_to_id`, `id_to_data`, `domain_fields`), rather than as one contract per name. The single-contract implementation exposes the **same** resolution semantics, records model, reserved-field routing schema, recovery-safe ownership model, LDH validity rules, registrar pricing tiers, and SDK interface defined in this specification; it differs only in that delegation is expressed as parent-Zone-ID references inside one contract instead of cross-contract resolver pointers. When Midnight enables permissionless deployment, MidNS migrates to the delegated multi-contract architecture; the canonical `name path → account/target` semantics are preserved, so resolvers change only their traversal backend.

## Rationale

### Why a delegated multi-contract registry/resolver?

It is the architecture ENS — the most mature comparable system — uses, and it is the design MidNS targets for three reasons. First, **scalability**: per-name state lives in its own contract, so no single `Map` grows unbounded as the namespace grows (the performance ceiling that motivated moving away from a monolithic registry). Second, **extensibility**: a name's contract can carry application logic beyond naming (a DAO, a payment splitter, an organization that issues employee subdomains), which a shared registry cannot express per-name. Third, **clean delegation semantics**: each owner administers their own subtree in their own contract, matching DNS's mental model. The transitional single-contract deployment exists only because mainnet deployment is currently permissioned; it is behaviorally equivalent and migrates forward without changing resolution semantics.

### Why records/fields instead of automatic HD routing?

[MPS-0012](../mps/mps-0012-account-aliasing.md)'s vision is one name that auto-routes to all HD-derived addresses. We achieve the same *user experience* through explicit records rather than in-contract HD derivation, for three reasons. First, the contract cannot itself derive a user's Shielded/Unshielded/Dust addresses — those come from the wallet seed off-chain; storing them as records is the only way the chain can know them. Second, records are strictly more general: they extend naturally to external-chain addresses (`chain:<caip-2>`) and to addresses the user wants to point elsewhere, which pure HD derivation cannot express. Third, this matches the established ENS records model. The cost is that the routing table is only as complete as the records the owner published; we mitigate this by recommending wallets auto-populate the reserved fields at registration.

### Why lowercase-ASCII LDH only?

Admitting Unicode would import the entire confusables/normalization problem ([MPS-0012](../mps/mps-0012-account-aliasing.md) OQ9) into every wallet's rendering layer. Restricting to lowercase ASCII LDH solves the phishing-resistance goal at the *registry* layer — the strongest place to solve it — at the cost of internationalized names. A future MIP MAY layer a UTS-46/ENSIP-15-style normalization profile on top to admit a vetted Unicode subset; until then, ASCII-only is a safe, simple default.

### Why hash-commitment ownership instead of a stored public key?

It is simultaneously the secure choice (avoids the bypassable `ownPublicKey()` pattern) and the recovery-safe choice (the secret is seed-derived, so device loss does not lose the name). Storing a raw wallet public key would either be insecure (if checked via the witness) or break recovery and multi-device use.

### Why propose midnight.domains as canonical?

Because the alternative — several providers each minting `alice` — is precisely the fragmentation [MPS-0012](../mps/mps-0012-account-aliasing.md) warns about, and the cross-chain record (see [Motivation](#motivation)) shows it is avoided only by early convergence on one registry per TLD. MidNS is already live with an SDK, docs, and a running resolver; designating its `.night` root canonical lets the ecosystem integrate one resolver now. Partner providers retain a healthy role: vanity TLDs, premium tiers, off-chain metadata, and value-added services on top of the shared uniqueness primitive — the ENS/Unstoppable and SNS/AllDomains pattern.

## Path to Active

### Acceptance Criteria

- Midnight Foundation endorsement of `.night` (the canonical root register) as the canonical Midnight namespace, resolving [MPS-0012](../mps/mps-0012-account-aliasing.md) Open Question 5 (namespace governance).
- Midnight enabling permissionless contract deployment, and the canonical multi-contract registry deployed against it.
- Integration of the resolver into at least one major wallet (e.g. Lace) such that users can send to `name.night` without seeing a raw address.
- A published, versioned SDK implementing the resolver interface and routing table (available as `@midnames/sdk`).
- Community review through the [MIP-0001](./mip-0001-mip-process.md) process, including workshop feedback.
- A follow-up MIP specifying reverse resolution and the address class it anchors.

### Reference Deployment

**Canonical multi-contract registry:** deployment **TBD** — pending Midnight enabling permissionless contract deployment. The reference contract (`leaf.compact`) and multi-contract SDK are implemented and tested but not yet deployed, since per-name contract deployment is not currently permitted on mainnet.

**Current transitional single-contract implementation (live):**

- **Mainnet contract:** `83b0d57aba442f92e12b5cdf92642adb9927ccd554a9061b5bd0992fc72596bb`
- **Preprod contract:** `0e6ff3e64e69fcaf513c13f4f66bfe66ab233e554d5c9fef4f2b16795df2e398`
- **TLD:** `.night`
- **Frontend:** [midnight.domains](https://midnight.domains) — search, register, manage, and share profiles (`/p/:name`).
- **SDK:** [`@midnames/sdk`](https://www.npmjs.com/package/@midnames/sdk) on npm.
- **Docs:** [docs.midnight.domains](https://docs.midnight.domains).
- **Resolver:** standalone CLI + HTTP resolver bundled with the SDK.

The live implementation supports registration (free under one's own name, or paid under another's), transfer, target/field updates, default-field resolution, and subdomains, with the same semantics specified here. Because Midnight does not yet expose native registration tokens, a Midnames-operated token-minting server issues a single-use, valueless registration token after confirming a Cardano-side payment ($cNIGHT or $ADA); the token is consumed at registration and cannot be minted by external parties.

### Implementation Plan

1. Maintain the live transitional single-contract registry and SDK as the reference implementation today.
2. On permissionless deployment, deploy the canonical `.night` multi-contract root register and migrate names, preserving the `name path → account/target` semantics.
3. Publish the reserved-field schema and routing table as a stable, versioned interface, and add wallet integration guidance for auto-populating `unshielded`, `dust`, and `epk` at registration.
4. Work with wallet teams to integrate the resolver behind a power-user toggle for raw addresses.
5. Author the follow-up MIPs flagged in [MPS-0012](../mps/mps-0012-account-aliasing.md): reverse resolution, name-rendering/phishing-resistance guidance, partner-registry migration, and the external-chain alias extension.

## Backwards Compatibility Assessment

This MIP introduces an application-layer standard and smart contracts. It requires **no protocol change and no hard fork**. Nothing in the existing ledger, address formats, or wallet derivation changes.

- **Wallets/dApps/explorers** opt in by integrating the resolver; non-integrating clients continue to use raw addresses unaffected.
- **Existing names** on the live transitional contract carry forward to the canonical multi-contract deployment with identical resolution results; the reserved-field schema is additive (owners add `unshielded`/`dust`/`epk` records when convenient; absence simply means "no address of that class").
- **Single-contract → multi-contract migration** preserves the canonical `name path → account/target` semantics, so resolvers need only swap their traversal backend (Zone-ID references → resolver-pointer delegation). Names and ownership commitments migrate as-is.
- **Partner registries** are not broken by this MIP, but consolidating onto the canonical namespace requires a migration path and conflict resolution where two providers issued the same name — deferred to a Partner-Registry Migration MIP ([MPS-0012](../mps/mps-0012-account-aliasing.md) OQ8). Names containing characters outside lowercase-ASCII LDH cannot be represented as-is and need a normalization or transliteration policy at migration time.

## Security Considerations

### Ownership secret = identity

Control of a name equals knowledge of the `secretKey()` witness value. Leaking it is equivalent to leaking a wallet signing key: anyone with the secret can transfer the name or change its target. Deriving the secret from the wallet seed ties name custody to the wallet's own threat model. Wallets MUST treat it as sensitive key material and MUST NOT disclose it on-chain (it is only ever consumed inside `persistentHash`).

### `ownPublicKey()` must not gate ownership

As in [MIP-0004](./mip-0004-fungible-token-standard-with-utxo.md), `ownPublicKey()` is frontend-supplied and unbound to the proof. Ownership MUST be checked by recomputing the commitment from the witness secret inside the circuit and asserting equality against `DOMAIN_OWNER[0]`. Implementations SHOULD bind a fixed domain-separator tag into the commitment hash to prevent a secret from being replayed against an unrelated contract.

### Registration front-running

Registration is a single public transaction with no commit-reveal phase, so a desirable name in a pending transaction can be observed and front-run. This is a known limitation. A future revision SHOULD add a two-phase commit-reveal (`H(label, nonce)` committed first, then revealed) to mitigate it; reliable expiry of stale commitments depends on a usable on-chain time source.

### Sealed identity and transfer

A child's `PARENT_*` and `DOMAIN` are sealed at construction. A transferred name therefore requires the new owner to deploy a fresh resolver and the parent to re-point the delegation via `set_resolver`; until the parent re-points, the old resolver still answers. Resolvers MUST treat the parent's `domains` entry (not the child contract's self-asserted identity) as authoritative for the delegation, so a stale or rogue child contract cannot hijack a name it is no longer delegated.

### Phishing and confusables

The lowercase-ASCII LDH restriction removes mixed-script and homoglyph attacks at the registry layer ([MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 4 / OQ9). Residual look-alike risks within ASCII remain (e.g. `rn` vs `m`, `0` vs `o`); a Name-Rendering MIP SHOULD give wallets guidance (e.g. flagging unverified or visually risky names). The misdirected-funds protection in the routing table — never substituting a different address class when the requested one is absent — is itself a safety property.

### Privacy of registrations

The canonical registry is public, so the existence of `alice.night`, its target, owner address, and records are public. This is a deliberate trade-off: a public registry is what makes uniqueness cheaply verifiable, but it exposes a partial social graph ([MPS-0012](../mps/mps-0012-account-aliasing.md) OQ6). Two mitigations are built in: the controlling identity is a hash *commitment*, not a wallet key, so the registry does not by itself link a name to a spendable wallet key; and storing a Shielded target as a coin public key (with the `epk` in a separate field) means resolving `alice → Shielded address` reveals no more than Alice directly sharing that Shielded address would ([MPS-0012](../mps/mps-0012-account-aliasing.md) Goal 6). Users who do not want a public binding simply do not register or do not publish a given address class.

### Root governance and centralization

The canonical root register's owner controls second-level issuance and pricing, and the transitional registration-token minting server is operated by Midnames. These are centralization points appropriate to a launch phase but inconsistent with a fully canonical, neutral namespace. Path-to-Active includes Foundation endorsement and governance of the root; a production canonical deployment SHOULD migrate root control to a Foundation-aligned or DAO-style steward, mirroring ENS (True Names Ltd + DAO) and SNS (SNS DAO).

## Implementation

### Components

1. **Register/resolver contract** (`leaf.compact`, Compact `>= 0.20`): the per-name ledger state and circuits specified above (`register_domain_for`, `set_resolver`, `update_domain_target`, `update_default_field`, `add_multiple_fields`, `clear_field`, `clear_all_fields`, `update_color`, `update_costs`, `transfer_domain`, `change_owner`).
2. **Resolver SDK** (`@midnames/sdk`, TypeScript): cross-contract delegation traversal from the canonical root, read operations (resolution, fields, profile, subdomains), write operations mirroring the circuits, provider configuration for mainnet/preprod/preview, and Shielded-address derivation from the target + `epk`.
3. **Standalone resolver**: CLI and HTTP server resolving `name.night`, `name.night/<field>`, and full-profile JSON.
4. **Frontend** ([midnight.domains](https://midnight.domains)) and **token-minting server** for Cardano-settled registration.
5. **Transitional single-contract registry** (live today; see [Current Implementation](#current-implementation-transitional)).

### Dependencies

- [Compact](https://docs.midnight.network/compact) language and Compact Standard Library (`persistentHash`, `Map`, `Set`, `Either`, `Maybe`, `sendUnshielded`/`receiveUnshielded`, `UserAddress`, `ZswapCoinPublicKey`, `ContractAddress`).
- `@midnight-ntwrk/wallet-sdk-address-format` for Bech32m encoding and Shielded-address derivation.
- `@midnight-ntwrk/midnight-js-indexer-public-data-provider` and `@midnight-ntwrk/midnight-js-network-id` for reading ledger state.
- Midnight support for **permissionless contract deployment** for the canonical multi-contract registry.
- Optional alignment with [MPS-0003 / CAIP-2](../mps/mps-0003-caip-support.md) for `chain:<caip-2>` external-chain field keys.

## Testing

### Unit Tests (contract)

- **Validity**: reject empty, over-length (`> 32`), leading/trailing-hyphen, uppercase, and non-LDH labels; accept valid LDH labels; verify `0xFF` padding handling.
- **Uniqueness**: a second `register_domain_for` for the same label in the same register reverts.
- **Ownership**: owner-gated circuits revert when the caller's `secretKey()` does not hash to `DOMAIN_OWNER[0]`; pass when it does.
- **Payment**: free registration for the register owner; correct tier (`short`/`med`/`long`) charged to others by label length; revert when `BUY_ENABLED` is false; payment forwarded to the owner's address.
- **Records**: `add_multiple_fields` (bounded at 10), `clear_field`, `clear_all_fields`, `update_default_field`.
- **Delegation**: `set_resolver` re-points a child; `transfer_domain` reassigns a child record and re-indexes `domains_owned`; `change_owner` changes the contract's own owner.

### Integration / Resolver Tests

- Multi-label resolution (`a.b.c.night`) walks delegations from the canonical root to the correct leaf target.
- Default resolution returns the `DEFAULT_FIELD` value when set, else the target.
- Routing table: a name with `unshielded`, `dust`, and `epk` fields returns the correct address for each operation context; an absent field yields an explicit "no address of that class" result, never a fallback.
- Shielded-address derivation: a `ZswapCPKAddr` target + `epk` field reconstructs the expected `mn_shield-addr1…`.
- Stale-delegation safety: a child resolver no longer delegated by its parent is not returned by resolution.
- Recovery: re-deriving `secretKey()` from the same seed re-establishes control of previously registered names.
- Cross-client consistency: SDK, CLI, and HTTP resolver return identical results for the same name.
- Equivalence: the transitional single-contract implementation and the multi-contract implementation return identical resolution results for the same name set.

## References (Optional)

- [MPS-0012: Human-Readable Aliasing for Midnight Accounts](../mps/mps-0012-account-aliasing.md)
- [MPS-0003: CAIP-2 Compliant Network Identifiers](../mps/mps-0003-caip-support.md)
- [MIP-0001: Midnight Improvement Proposal Process](./mip-0001-mip-process.md)
- [MIP-0004: Fungible Token Standard with UTXO Conversion Extensions](./mip-0004-fungible-token-standard-with-utxo.md) — prior art for hash-based caller authentication and the `ownPublicKey()` hazard.
- [Midnight Name Service — midnight.domains](https://midnight.domains) and [docs.midnight.domains](https://docs.midnight.domains).
- [`@midnames/sdk` on npm](https://www.npmjs.com/package/@midnames/sdk).
- [ENS — Ethereum Name Service](https://ens.domains/) (True Names Ltd + ENS DAO) — registry/resolver delegation prior art.
- [SNS — Solana Name Service](https://www.sns.id/) (Bonfida → SNS DAO).
- [ADA Handle](https://adahandle.com/) — single-PolicyID NFT uniqueness on Cardano.
- [NFDomains](https://app.nf.domains/) and [ANS](https://algonameservice.com/) — the contested `.algo` namespace on Algorand.
- [ENSIP-15: ENS Name Normalization](https://docs.ens.domains/ensip/15) and [UTS #46](https://www.unicode.org/reports/tr46/) — prior art for a future internationalization MIP.

## Acknowledgements

Thanks to the authors of [MPS-0012](../mps/mps-0012-account-aliasing.md), Karmel Elshinnawi and Nick Stanford, for framing the account-aliasing problem space; to the Midnight Foundation Identity working group for feedback across the milestone reviews; and to ecosystem partners building naming layers for surfacing the coordination problem this MIP addresses.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
