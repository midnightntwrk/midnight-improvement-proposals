---
MIP: "0008"
Title: CAIP-2 Network Identifiers for Midnight
Authors:
  - alba-press (alba-press)
  - Mahesh Sashital (mbs-midnight)
Status: Draft
Category: Standards
Created: 2026-05-30
Requires: none
Replaces: none
MPS: MPS-0003
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

Midnight currently identifies networks using unqualified lowercase strings such as `mainnet`, `preprod`, and `preview`. These values appear in wallet SDKs, indexers, and DApp connector APIs. They are not globally unique: the string `mainnet` is shared by hundreds of blockchains and cannot be used safely in multi-chain wallet registries, CAIP chain-ID fields in wallet policy/audit surfaces, or cross-chain tooling without additional context.

This MIP adopts [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) as the **external** network identification format for Midnight, using the namespace `midnight`.

The reference portion is the existing Midnight network name, for example `midnight:mainnet` and `midnight:preprod`, mapped one-to-one from Midnight's `NetworkId` strings in [midnight-wallet `NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts). Each registered name is paired with a published genesis identifier, which clients use to verify that they are connected to the intended chain instance. The genesis identifier is verification metadata: it does not appear in the CAIP-2 identifier, in addresses or in transactions.

The specification covers CAIP-2 grammar constraints, parsing rules, a canonical network registry, reset and reuse rules for registered names, chain-instance verification, and a backwards-compatible transition strategy (dual acceptance of legacy and CAIP-2 forms, canonicalizing to CAIP-2 in new APIs). Genesis-hash, numeric, and hybrid reference formats were considered and are documented as alternatives. Scope is limited to CAIP-2 chain identifiers; account IDs (CAIP-10), address encoding, bridge semantics, and ledger consensus are out of scope.

This MIP responds to [MPS-0003: CAIP-2 Compliant Network Identifiers for Wallet Ecosystem Integration](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md). The reference-format decision was reached in [Discussion #195](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/195).

**Submission scope:** This MIP is for review in [midnight-improvement-proposals](https://github.com/midnightntwrk/midnight-improvement-proposals) only. ChainAgnostic namespace registration and third-party wallet registry updates (e.g. OWS) are **deferred** until this MIP is Accepted.

## Motivation

### Problem

Midnight network identifiers today are simple strings defined in wallet abstractions and consumed across the stack. Verified against the live wallet SDK source ([`NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts), 2026-05-30):

```ts
export const NetworkId = {
  MainNet: 'mainnet',
  TestNet: 'testnet',
  DevNet: 'devnet',
  QaNet: 'qanet',
  Undeployed: 'undeployed',
  Preview: 'preview',
  PreProd: 'preprod',
} as const;
```

Applications set these via `@midnight-ntwrk/midnight-js-network-id` (`setNetworkId('preprod')`), indexers validate lowercase network strings, and the DApp connector accepts a plain `networkId` hint (e.g. `'mainnet'`). This works within Midnight-only code paths but fails at ecosystem boundaries.

### Why existing capabilities are inadequate

1. **Namespace collision:** `mainnet`, `testnet`, and similar strings are ambiguous without chain context. Multi-chain wallets cannot register Midnight in a chain registry alongside Ethereum (`eip155:1`) or Solana (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) using bare strings.
2. **Standards incompatibility:** [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) (status: Final) defines identifiers as `namespace:reference`. Midnight does not conform, forcing each integrator to build ad hoc translation layers.
3. **Wallet ecosystem friction:** Multi-chain wallet standards use CAIP-2/CAIP-10 in wallet files, policy contexts, audit logs, and API parameters. As of community discussion in [OWS #222](https://github.com/open-wallet-standard/core/issues/222), no `midnight` namespace is registered at [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces).
4. **Developer experience:** DApp and wallet developers familiar with multi-chain patterns expect CAIP-2 identifiers in configuration and chain registries. Midnight-specific string handling increases integration cost and error rates.

### Stakeholder context

Cross-chain wallet interoperability research, including work at **Ura Labs** that informed discussion in [OWS #222](https://github.com/open-wallet-standard/core/issues/222), highlighted integrator friction with unqualified Midnight network strings. That research motivated follow-on specification work; it does not define canonical identifiers.

**Midnight Foundation, MIP editors, and Midnight core engineering retain sole authority** over reference format, registry contents, and acceptance of this MIP.

## Specification

### Scope

This MIP specifies:

- CAIP-2 grammar constraints applicable to Midnight
- The CAIP-2 namespace `midnight`
- The format of the CAIP-2 `reference` portion
- A canonical registry mapping CAIP-2 identifiers to legacy Midnight network strings and genesis identifiers
- Reset and reuse rules for registered names
- How clients verify that they are connected to the intended chain instance
- Parsing, validation, and canonical serialization rules
- The resolution method for a future ChainAgnostic namespace registration
- Backwards-compatible transition requirements for APIs and SDKs

This MIP does **not** specify bridge protocols, CAIP-10 account IDs, address formats, or ledger-embedded `network_id` serialization.

### Non-goals

The following are explicitly out of scope:

- Bridge semantics, cross-chain deposit/claim/refund flows, or application-specific routing
- CAIP-10 account identifier adoption
- Changes to Bech32m address encoding (`mn_addr1…`, `mn_addr_preprod1…`, shielded address formats)
- Ledger consensus, block production, ZK circuit, or transaction-format changes
- Renaming or consolidating Midnight test environments (`qanet`, `devnet`, etc.)
- WalletConnect / CAIP-25 session protocol details beyond registering chain IDs
- **ChainAgnostic namespace PRs**, deferred until this MIP is Accepted
- **Third-party wallet registry updates** (including OWS), deferred until this MIP is Accepted

### CAIP-2 baseline

Per [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) (status: Final):

```
chain_id  = namespace + ":" + reference
namespace = [-a-z0-9]{3,8}
reference = [-_a-zA-Z0-9]{1,32}
```

Properties relevant to Midnight:

- Identifiers are **case-sensitive**.
- Design goals include ecosystem-wide uniqueness, partial human-readability, on-chain storage feasibility, hardware-wallet display, and URL-path safety.
- The reference's **semantics and resolution method** are delegated to a per-namespace specification at [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces).
- CAIP-2 sets grammar only; it does **not** mandate genesis hash vs network name. The namespace specification decides.

**Midnight grammar check:** `midnight` is a legal namespace (8 characters, the CAIP-2 maximum). Every registered and unregistered Midnight network name fits the 1–32 character reference rule.

Implementations MUST reject malformed identifiers at API boundaries (invalid namespace, empty reference, illegal characters).

### Namespace

The CAIP-2 namespace for Midnight blockchains is:

```
midnight
```

Properties:

- Length: 8 characters (maximum allowed by CAIP-2 `[-a-z0-9]{3,8}`)
- Lowercase only
- Refers to the Midnight blockchain platform and its deployed networks

Registration of this namespace at [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces) is a prerequisite for ecosystem-wide interoperability and SHOULD follow **Acceptance** of this MIP. **No ChainAgnostic PR is proposed as part of this submission.**

### Reference format

The reference portion is the existing Midnight network name string, lowercase, matching the `NetworkId` values in [midnight-wallet `NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts).

```
reference = network-name
where network-name matches [-_a-zA-Z0-9]{1,32} per CAIP-2
```

Examples:

```
midnight:mainnet
midnight:preprod
midnight:preview
```

This follows the Cosmos namespace pattern (`cosmos:cosmoshub-4`) and MPS-0003 Appendix B.

**What the name provides.** The network name is already part of ledger state, appears in every Bech32m address, and is checked in every transaction. Using it as the reference keeps the CAIP-2 identifier consistent with everything Midnight already emits, without a mapping table.

**What the name does not provide.** A name does not fingerprint a chain instance. Different genesis instances can carry the same string, for example after a test network reset, or in a private or local deployment that reuses a public name. This MIP closes that gap with published genesis identifiers (*Canonical network registry*), rules for what happens to a name when its network is reset (*Reset and reuse rules*), and a procedure clients follow to check the instance (*Chain-instance verification*).

#### Documented alternatives (not adopted)

The following alternatives from MPS-0003 Open Question #1 were considered and are not adopted. They are kept for the record.

| Option | Example | Ecosystem precedent | Summary |
|---|---|---|---|
| **B. Genesis hash** | `midnight:1941ca8e2bb88146c14dea08…` | Solana, Bitcoin, Polkadot | Truncated genesis hash as the reference. Verifiable, but not human-readable and not aligned with the strings Midnight already uses. This MIP uses the genesis hash as verification metadata instead. |
| **C. Numeric ID** | `midnight:1`, `midnight:109` | EIP-155 (`eip155:1`) | Familiar to EVM developers; no established public Midnight numeric chain ID; internal values risk confusion with unrelated numeric namespaces. |
| **D. Hybrid name + hash prefix** | `midnight:mainnet-1941ca8e` | — | Readability plus verifiability, but complex parsing, and the 32-character limit constrains the hash prefix. |
| **E. Hashed fallback** | `midnight:hashed-0204c92a0388779d` | Cosmos (long `chain_id`s) | Only needed if a reference exceeds 32 characters. No Midnight network name does. |

### Canonical network registry

Registered references:

| CAIP-2 identifier | Legacy `NetworkId` | Chain name (`system_chain`) | Genesis identifier (current instance) | Verified |
|---|---|---|---|---|
| `midnight:mainnet` | `mainnet` | `Midnight Mainnet` | `0x1941ca8e2bb88146c14dea084d3be7eb6e96ca7135429c543848b628124f2854` | 2026-11-20 |
| `midnight:preprod` | `preprod` | `Midnight Preprod` | `0xdf831b09a8baa92badf47762ce5ac439b7e47e3ed3d39600cfdd44fad552361b` | 2026-11-20 |
| `midnight:preview` | `preview` | `Midnight Preview` | `0x3c096de209e06a1a8c52be7bda109ee8c891a42288c0a9db5495f9616dd13796` | 2026-11-20 |

The genesis identifier is the hash of block 0, as returned by the node's `chain_getBlockHash(0)` RPC: 32 bytes, written as `0x` followed by 64 lowercase hex digits.

Superseded instances: when a registered test network is reset under *Reset and reuse rules*, its previous genesis identifier is listed here with the date it stopped being current. No reset under those rules has happened yet.

| CAIP-2 identifier | Genesis identifier | Current until |
|---|---|---|
| — | — | — |

**Pre-MIP exception: Preview, 7 November 2026.** Before this amendment, Preview was relaunched as a new chain on 7 November 2026, with no state carried over from the previous instance. The relaunch reused the previous genesis: the current instance's `chain_getBlockHash(0)` is still `0x3c096de209e06a1a8c52be7bda109ee8c891a42288c0a9db5495f9616dd13796`. So the instance check (step 3 of *Chain-instance verification*) alone cannot tell the pre-relaunch chain from the current one, and the superseded instance cannot be listed by its genesis identifier. A client that must tell them apart, for example one holding Preview state stored before 7 November 2026, SHOULD also compare `chain_getBlockHash(29)` with `0xb05a5b4ce244dd42b827129f3f155dfb13090218b5810bdc69e59edc96e0dbef`. Block 29 is the first block after the relaunch (timestamp 2026-11-07 00:34:47 UTC; hash read 2026-11-20). Blocks 1 to 28 carry timestamps from 16 July 2026, so they do not distinguish the two instances.

Rule 2 of *Reset and reuse rules* applies to every reset after this amendment, so this exception is not repeated.

Unregistered names. These are valid CAIP-2 strings, but no uniqueness is guaranteed for them, and a chain using one is not a registered reference:

| Name | Use |
|---|---|
| `undeployed` | Local and emulated chains. Every local chain uses this name, so it never identifies one instance |
| `stagenet` | Midnight's staging network. It may be reset or redeployed without notice |
| `qanet`, `devnet` | Internal environments |
| `testnet` | Legacy label with no current public network. It MUST NOT be treated as an alias for `preprod` or `preview` |
| `govnet` and other operator environments | Reserved; may be registered by amendment |

Mapping rules:

1. CAIP-2 identifier = `midnight:` + the legacy string, both lowercase.
2. Each legacy string maps to exactly one CAIP-2 identifier, and the reverse.
3. Distinct environments always have distinct identifiers. Silent aliasing (for example `qanet` → `preview`) is forbidden, even when operator tooling maps several environments to shared off-chain infrastructure.
4. A registered name can be added, or an unregistered one promoted, only by amending this MIP. Recording a new genesis identifier for an existing registered name follows *Reset and reuse rules*: it is a registry update to this document, not a new MIP.

### Reset and reuse rules

A registered name identifies a network's role. Its genesis identifier identifies the current chain instance in that role.

1. **A name survives a reset.** When a registered test network is reset, it keeps its name. `midnight:preprod` still means "the Preprod network" after the reset.
2. **A reset MUST produce a new genesis identifier.** Whoever performs the reset changes the genesis, for example the chain specification's genesis timestamp or a reset nonce in the genesis configuration, so that block 0 differs from every previous instance. Relaunching a registered network from an unchanged chain specification reproduces the old genesis hash, so clients cannot detect the reset. That is not a permitted reset of a registered network.
3. **An upgrade is not a reset.** A runtime or ledger upgrade applied in place keeps the chain's history and its block 0, so the genesis identifier does not change. For example, Mainnet and Preprod kept their genesis identifiers through the ledger 9 upgrade in November 2026.
4. **Each reset is recorded.** The Midnight Foundation records the new genesis identifier in the canonical registry, with the date it takes effect, no later than the date the reset network accepts its first public transaction. The previous identifier moves to *Superseded instances* with its end date, so historical references remain interpretable.
5. **Nothing carries across a reset.** Contracts, balances, channels and other state of a superseded instance do not exist on its successor. An application that stores a CAIP-2 identifier alongside state SHOULD also store the genesis identifier it observed, so that a reset is detectable rather than silently misread.
6. **`mainnet` is never reset or reused.** A new production genesis would be a new network. It requires an amendment to this MIP and a new registered name.
7. **Names are never reassigned to another role.** A retired name stays retired. For example, `preview` cannot later be used for the network now known as `preprod`.
8. **Other chains using a registered name are not that network.** Any chain can set its network ID to a registered string, for example a private deployment or a fork. Only the instance whose genesis identifier is the registry's current entry is the registered network. *Chain-instance verification* is how a client tells them apart.
9. **Unregistered names carry no guarantees.** Any number of chains may use `undeployed`, `stagenet` or the other unregistered names at once. Tools MUST NOT present them as registered references, and production builds SHOULD reject them.

### Parsing and validation

Implementations MUST:

1. Split on the first `:` only: `namespace`, `reference`.
2. Verify `namespace === "midnight"`.
3. Verify `reference` matches `[-_a-zA-Z0-9]{1,32}` per CAIP-2.
4. Verify `reference` against the canonical registry. Unregistered names are accepted only in explicit development or local modes.

Canonical serialization:

- Namespace MUST be lowercase `midnight`.
- Reference MUST use the registry spelling (all lowercase).

Legacy parsing (transition period):

- A bare legacy string `mainnet` (no colon) MAY be accepted at API boundaries where historically supported.
- When accepted, implementations SHOULD canonicalize to the CAIP-2 form (e.g. `midnight:mainnet`) for storage in new APIs and external-facing responses.

#### Reference implementation sketch (non-normative)

```ts
// Illustrative API surface for wallet/SDK integrators
parseCaip2Midnight(id: string): { namespace: 'midnight'; reference: string } | Error
legacyToCaip2(legacy: string): string    // e.g. 'preprod' → 'midnight:preprod'
caip2ToLegacy(id: string): string        // inverse when the reference is registry-known
expectedGenesis(id: string): string      // current genesis identifier from the registry
```

### Chain-instance verification

A client resolves `midnight:<name>` to a live chain, and checks that it is the intended instance, as follows:

1. Connect to a node, or to an indexer that reports its node, for the intended network.
2. **Check the network name.** Read the network ID the indexer or connector configuration reports, and check that `midnight:` + that ID equals the expected CAIP-2 identifier. A client with only node RPC access MAY instead check that `system_chain` equals the registry's chain name. This is the same network the ledger checks in every transaction and that appears in addresses.
3. **Check the chain instance.** Read the genesis identifier with `chain_getBlockHash(0)`, and check that it equals the registry's current entry for that name. A client connected through an indexer MAY read the hash of the indexer's block at height 0 instead, and SHOULD check that it matches the node's. An all-zero hash means the read failed: it never matches, and the client treats it as a failed check, not as an unknown network.
4. If either check fails, the client MUST NOT sign or submit value-bearing transactions for that identifier. It SHOULD tell the user which check failed: a name mismatch is a wrong network, and a genesis mismatch on a matching name is a reset, a fork or a private deployment.

Requirements by context:

- Wallets and any service that holds or moves value MUST perform both checks before the first value-bearing operation on a connection, and SHOULD repeat step 3 on reconnect.
- Read-only tools MAY skip step 3.
- For unregistered names there is no registry entry, so step 3 does not apply. Such connections are development use only.

**Existing practice.** Step 3 codifies what Midnight's wallet SDK already does. Its node client compares the node's genesis hash with the indexer's block at height 0 and treats a mismatch as `WrongNetwork` ([`NodeClient.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/node-client/src/effect/NodeClient.ts)). This MIP adds the published registry value as the reference both are checked against. It reuses existing plumbing, with no ledger, address or transaction-format changes.

**What the genesis identifier is not.** It is not part of the CAIP-2 identifier and does not change Bech32m addresses, transaction serialization, `network_id` in ledger state, or proofs.

**For the ChainAgnostic namespace profile:** the reference is the network name, and the resolution method is the two checks above, with the canonical registry as the source of chain names and genesis identifiers.

### API surfaces

The following Midnight components SHOULD accept CAIP-2 identifiers in new API versions while maintaining legacy support during transition:

| Surface | Current behavior | Target behavior |
|---|---|---|
| `@midnight-ntwrk/midnight-js-network-id` | `setNetworkId('preprod')` | Accept CAIP-2; canonicalize internally |
| Wallet SDK / facade | Legacy `networkId` string; genesis check against the indexer | Dual-parse; check the genesis identifier against the registry |
| DApp connector `connect(networkId)` | Plain `'mainnet'` | Accept CAIP-2 and legacy during transition |
| Indexer GraphQL | Lowercase legacy string | Accept CAIP-2 in filters; return CAIP-2 in new schema fields |

Downstream wallet registries and standards such as OWS may use the Midnight CAIP-2 identifiers after this MIP is Accepted. Specific OWS metadata, signer, and wallet integration changes are out of scope for this MIP.

Deprecation:

- Legacy bare strings SHOULD continue to work during a **transition period (suggested minimum: six months, subject to core release planning and migration analysis per MPS-0003)** with deprecation warnings in logs/headers where applicable.
- After the deprecation window, bare strings MAY be rejected in a coordinated major SDK release, subject to core release planning.

### Versioning

The specification version is `0.2.0-draft`. Changes to the reference format, to the set of registered names, or to the reset rules require a MIP revision or superseding MIP. Recording a new genesis identifier after a reset is a registry update under *Reset and reuse rules* and does not change the specification version.

## Rationale

### Why CAIP-2

[CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) is the Final chain-identification standard used by ChainAgnostic wallets, indexers, and multi-chain tooling. Adopting it removes bespoke translation at every integration boundary and prevents namespace collisions with Ethereum, Solana, and other ecosystems. The grammar accommodates Midnight without constraining the reference-format choice.

### Why namespace `midnight`

- Matches the platform name used in documentation, SDK packages (`@midnight-ntwrk/*`), and ecosystem discourse.
- Length 8 fits CAIP-2 namespace rules (`[-a-z0-9]{3,8}`) and is the maximum permitted length.
- No known collision with existing ChainAgnostic namespaces at time of drafting.

### How other ecosystems structure references

| Camp | Examples | Reference style | Resolution |
|---|---|---|---|
| Network name | Cosmos (`cosmos:cosmoshub-4`), Tron, Sui, NEAR | Human-readable name | Node status / registry table |
| Numeric | EVM (`eip155:1`) | Chain ID integer | `eth_chainId` |
| Genesis hash | Solana, Bitcoin (BIP-122), Polkadot | Truncated genesis hash | `getGenesisHash`, block 0 hash, etc. |

Cosmos resolves via Tendermint `/status` → `chain_id`, hashing only if the name exceeds 32 characters. Polkadot deliberately uses the genesis block hash (first 16 bytes, lowercase hex) because the genesis hash is load-bearing in network and consensus.

### Why a network name, with genesis identifiers as metadata

The decision was reached in [Discussion #195](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/195):

- The network ID string points back to the genesis and ledger state, is part of every wallet address, and is checked in every transaction. As a reference it is consistent with why it was introduced and how it is used today, and it is human-readable.
- It is not a genesis fingerprint: different genesis instances can use the same string, particularly across resets or private deployments. Identifying the chain instance needs a separate check.
- Publishing genesis identifiers as verification metadata provides that check without changing Midnight's internal network IDs, address serialization or transaction serialization.

| Criterion | Option A (name + genesis metadata) | Option B (genesis) | Option C (numeric) | Option D (hybrid) |
|---|---|---|---|---|
| Matches existing SDK strings | Yes | No | No | Partial |
| Human-readable | Yes | No | Moderate | Yes |
| Verifies the chain instance | Yes, against the registry | Yes | Only if IDs are published | Partial |
| MPS-0003 alignment | Appendix B recommendation | Open Question #1 | Open Question #1 | Open Question #1 |
| Ledger / ZK impact | None | None | None | None |

### Why a reset must change the genesis

The genesis check only detects a reset if the reset produces a new block 0. Relaunching a Substrate chain from an unchanged chain specification reproduces the same genesis hash even though all history is gone. This has happened on Midnight: Preview was relaunched as a new chain on 7 November 2026, with no state carried over, and its `chain_getBlockHash(0)` result is the same as before the relaunch. That relaunch predates this amendment. It is recorded as a pre-MIP exception under *Canonical network registry*, which tells verifiers to also compare the hash of block 29, the first block after the relaunch. Rule 2 of *Reset and reuse rules* makes a changed genesis part of what a reset of a registered network is, so the existing `WrongNetwork` check keeps working without a second checkpoint.

### Why the name survives a reset

Giving each reset a new name (for example `preprod-2`) would make every reference instance-unique, but it breaks the one-to-one mapping with Midnight's internal network IDs and forces every integrator to re-key configuration after each reset. Keeping the name and changing the genesis identifier preserves the mapping, while the registry history keeps old references interpretable.

### Downstream wallet standards (informational only)

Multi-chain wallet tooling expects `namespace:reference` chain IDs per [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md). Community discussion in [OWS #222](https://github.com/open-wallet-standard/core/issues/222) surfaced integrator interest in Midnight identifiers; that discussion is **context only** and does not define canonical identifiers.

**No OWS or ChainAgnostic PR is proposed as part of this submission.**

### Why dual support during transition

Hard cutover risks breaking shipped wallets, DApps, and operator tooling that persist legacy strings. Dual acceptance with canonicalization balances safety and a clear end state, consistent with optional migration targets discussed in MPS-0003.

### Why decouple from address format

Bech32m address HRPs (`mn_addr_preprod1…`) already encode network context for addresses. CAIP-2 identifies the **chain** for wallet routing and policy, not the address serialization format. Merging the two would break existing address validation and test vectors.

## Path to Active

### Acceptance Criteria

This MIP reaches **Accepted** when:

1. MIP editors assign a number and merge the draft after community commentary (≥2 weeks per MIP-1).
2. Midnight core engineering confirms the reference format. See *Resolved core decisions*.
3. The canonical network registry and the reset and reuse rules are confirmed by core.
4. No unresolved security objections from reviewers regarding chain spoofing or migration windows.
5. The MIP Editors vote to accept.

This MIP reaches **Active** when:

1. The reference format, registry and chain-instance verification are implemented in released wallet SDK and indexer versions designated by Midnight Foundation.
2. At least one first-party wallet connector documents CAIP-2 acceptance.
3. A migration guide is published for wallet and DApp developers.
4. The Midnight Foundation has named the owner responsible for recording genesis identifiers after resets (*Reset and reuse rules*, rule 4).

Downstream ecosystem work may include ChainAgnostic namespace registration, wallet registry updates, and OWS chain metadata. That work is **not** a blocking condition for this MIP reaching Active.

### Implementation Plan

1. **Phase 0 — Specification (this MIP):** Community review in midnight-improvement-proposals; reference format, registry and reset rules confirmed by core.
2. **Phase 1 — Adapter layer:** Add parse/canonicalize helpers and the registry, including genesis identifiers, to `midnight-js-network-id` and wallet abstractions. Extend the wallet's existing genesis check to compare against the registry. No ledger changes.
3. **Phase 2 — API exposure:** Indexer and connector return/accept CAIP-2; deprecation warnings for legacy-only clients.
4. **Phase 3 — Deprecation:** Remove legacy bare-string acceptance in a coordinated major SDK release after the transition period.

Downstream ecosystem work (ChainAgnostic namespace registration, third-party wallet registries, OWS chain metadata) follows **after** acceptance and is out of scope for this MIP's implementation plan.

## Backwards Compatibility Assessment

### Impact summary

| Area | Hard fork required? | Compatibility approach |
|---|---|---|
| SDK `setNetworkId` | No | Dual-parse legacy and CAIP-2 |
| Persisted wallet config | No | Migrate on read; canonicalize on write |
| Wallet genesis check | No | Existing check, compared against the published registry value |
| Indexer queries | No | Additive schema fields; legacy filters supported during transition period |
| Bech32m addresses | No | Unchanged |
| ZK proofs / ledger state | No | Unchanged; `network_id` serialization untouched |
| Third-party DApps | Low–medium | Continue working via legacy strings during transition period |

### Legacy ↔ CAIP-2 mappings

```
mainnet     ↔ midnight:mainnet     (registered)
preprod     ↔ midnight:preprod     (registered)
preview     ↔ midnight:preview     (registered)
testnet     ↔ midnight:testnet     (unregistered)
qanet       ↔ midnight:qanet       (unregistered)
devnet      ↔ midnight:devnet      (unregistered)
undeployed  ↔ midnight:undeployed  (unregistered)
```

Implementations MUST NOT treat `testnet` as an alias for `preprod` or `preview`.

### User impact

End users SHOULD NOT need manual migration if wallets auto-canonicalize on upgrade. Developers SHOULD update configuration docs and environment examples to show CAIP-2 as the preferred form.

## Security Considerations

### Chain spoofing

**Risk:** A malicious DApp or RPC endpoint supplies `midnight:mainnet` while the wallet is connected to `midnight:preprod`, or to a different chain that reports the name `mainnet`, inducing wrong-chain signing.

**Mitigations:**

- Wallets MUST perform *Chain-instance verification* before value-bearing operations. A DApp- or RPC-supplied identifier is never sufficient on its own.
- Wallets SHOULD display the human-readable network name prominently in confirmation UI.
- Production builds MUST reject `midnight:` references that are not in the canonical registry, except in explicit development or local modes.
- A node that reports a registered name with a different genesis identifier is treated as a different chain, whatever name it reports.

### Undetectable resets

**Risk:** A registered test network is relaunched from an unchanged chain specification. Its genesis identifier is unchanged, so clients that stored state against the previous instance cannot tell that state is gone.

**Mitigations:**

- *Reset and reuse rules*, rule 2: a reset of a registered network MUST produce a new genesis identifier.
- Applications store the genesis identifier with any state they persist (rule 5).
- For Preview's 7 November 2026 relaunch, which predates rule 2, verifiers also compare the hash of block 29, the first block after the relaunch (*Canonical network registry*, pre-MIP exception).

### Registry integrity

**Risk:** A tampered or stale copy of the registry leads clients to accept the wrong chain instance.

**Mitigations:**

- This MIP is the canonical registry. SDKs SHOULD ship the registry values as constants and update them only through releases.
- Genesis identifiers SHOULD be confirmed against more than one node operator before they are recorded.

### Identifier injection

**Risk:** Malformed identifiers bypass validation and corrupt storage.

**Mitigations:**

- Strict CAIP-2 regex validation at parse time.
- Split on first `:` only; validate reference separately (inputs like `midnight:mainnet:extra` yield reference `mainnet:extra`, which fails CAIP-2 character rules or registry lookup).

### Migration window

**Risk:** Extended dual support maintains two valid representations, increasing test surface and potential for inconsistent persistence.

**Mitigations:**

- Canonicalize to CAIP-2 on write in new SDK versions.
- Telemetry or debug logging when legacy form is received (opt-in) to measure migration progress.
- Published end date for legacy acceptance (duration set by core release planning).

### Privacy

Network identifier format changes MUST NOT alter zero-knowledge proof structures, shielded transaction semantics, or privacy guarantees. This MIP addresses identification metadata only.

## Implementation

### Components to modify (expected)

| Component | Change |
|---|---|
| `midnight-wallet` / `NetworkId.ts` | Add CAIP-2 parse, format, registry lookup including genesis identifiers; retain legacy constants |
| `midnight-wallet` / node client | Compare the genesis hash against the registry, in addition to the existing node/indexer comparison |
| `@midnight-ntwrk/midnight-js-network-id` | Accept CAIP-2 in `setNetworkId`; document canonical form |
| Wallet SDK facade | Dual-parse at initialization |
| Indexer | Expose CAIP-2 in API; validate incoming chain filters |
| DApp connector | Document and accept CAIP-2 in `connect()` |
| Documentation site | Update network and provider guides with CAIP-2 examples and the registry |

### Components explicitly deferred

| Component | Condition for inclusion |
|---|---|
| ChainAgnostic namespace registration | After MIP Accepted; separate downstream work |
| Third-party wallet registry updates | After MIP Accepted; separate downstream work |
| Bridge / cross-chain contracts | Out of scope |

### Dependencies

- [MPS-0003](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md) (problem statement)
- [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md)

### Resolved core decisions

| Item | Decision |
|---|---|
| Reference format | Network name (Option A). Genesis identifiers are published as verification metadata, not used as the reference |
| Genesis hash availability | Read from each network's node with `chain_getBlockHash(0)` and published in the canonical registry. This reuses existing plumbing, with no ledger, address or transaction-format changes: the wallet SDK already reads the genesis hash for its `WrongNetwork` check |
| Registry | `mainnet`, `preprod` and `preview` registered. `testnet`, `qanet`, `devnet`, `stagenet` and `undeployed` unregistered. `govnet` reserved |
| Resets | Name survives; a reset MUST produce a new genesis identifier; superseded identifiers are kept in the registry (*Reset and reuse rules*) |
| Ledger, addresses, internal IDs | Unchanged. CAIP-2 is an external format applied at API, SDK and connector boundaries |

## Testing

### Unit tests

- Parse valid CAIP-2 identifiers for every registered and unregistered name.
- Reject invalid namespace, empty reference, overlong reference, illegal characters.
- Round-trip: legacy → CAIP-2 → legacy for every name.
- Reject silent aliasing (e.g. `qanet` must not map to `preview`).
- Treat an all-zero genesis hash as a failed check.

### Integration tests

- A client connected to Preprod passes both verification checks against the registry.
- A local chain started with network ID `preprod` fails the genesis check and is reported as a different chain.
- After a registry update that records a new Preprod genesis identifier, a client that stored the old identifier detects the change.
- DApp connector accepts both `preprod` and `midnight:preprod` during transition.

### Regression tests

- Existing DApps using `setNetworkId('preprod')` continue to function unchanged during the transition period.
- Address generation and validation unchanged for all networks.
- ZK proof generation and verification unchanged.

### Test vectors

```
# Valid, registered
midnight:mainnet
midnight:preprod
midnight:preview

# Valid grammar, unregistered (development or local modes only)
midnight:undeployed
midnight:stagenet

# Invalid
Mainnet:mainnet          # wrong namespace case
midnight:                # empty reference
midnight:mainnet:extra   # reference becomes mainnet:extra — invalid or unknown
ethereum:1               # wrong namespace

# Genesis identifiers (chain_getBlockHash(0), verified 2026-11-20)
midnight:mainnet  0x1941ca8e2bb88146c14dea084d3be7eb6e96ca7135429c543848b628124f2854
midnight:preprod  0xdf831b09a8baa92badf47762ce5ac439b7e47e3ed3d39600cfdd44fad552361b
midnight:preview  0x3c096de209e06a1a8c52be7bda109ee8c891a42288c0a9db5495f9616dd13796
```

## References

- [CAIP-2: Blockchain ID Specification](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) (Final)
- [MPS-0003: CAIP-2 Compliant Network Identifiers for Wallet Ecosystem Integration](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md)
- [Discussion #195: MIP-0008 CAIP-2 Compliant Network IDs](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/195)
- [MIP-1: Midnight Improvement Proposal Process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-mip-process.md)
- [MIP template](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-template.md)
- [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces)
- [Cosmos namespace (CAIP-2)](https://github.com/ChainAgnostic/namespaces/blob/main/cosmos/caip2.md)
- [Solana namespace (CAIP-2)](https://github.com/ChainAgnostic/namespaces/blob/main/solana/caip2.md)
- [midnight-wallet NetworkId.ts](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts) (verified 2026-05-30)
- [midnight-wallet NodeClient.ts](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/node-client/src/effect/NodeClient.ts) (genesis-hash `WrongNetwork` check)
- [Midnight networks and environments](https://docs.midnight.network/guides/networks-and-environments)
- [Midnight configure providers / network ID](https://docs.midnight.network/guides/configure-providers)
- [Midnight Indexer v3.0.0 release notes (lowercase network ID)](https://docs.midnight.network/relnotes/midnight-indexer/midnight-indexer-3-0-0)
- [Midnight DApp connector InitialAPI](https://docs.midnight.network/api-reference/dapp-connector/type-aliases/InitialAPI)
- [OWS #222 — Chain support discussion (informational)](https://github.com/open-wallet-standard/core/issues/222)

## Acknowledgements

- Bob Blessing-Hartley and Midnight Foundation contributors for [MPS-0003](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md)
- kapke and NicolasDP for the review that settled the reference format and the chain-instance verification design
- ChainAgnostic community for CAIP-2 and namespace specifications

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission to the official Midnight Improvement Proposals repository requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for contributions to the Foundation.
