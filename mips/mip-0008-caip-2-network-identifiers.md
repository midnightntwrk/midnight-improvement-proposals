---
MIP: 0008
Title: CAIP-2 Network Identifiers for Midnight
Authors:
  - alba-press (alba-press)
Status: Draft
Category: Standards
Created: 2026-05-30
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

Midnight currently identifies networks using unqualified lowercase strings such as `mainnet`, `preprod`, and `preview`. These values appear in wallet SDKs, indexers, and DApp connector APIs. They are not globally unique: the string `mainnet` is shared by hundreds of blockchains and cannot be used safely in multi-chain wallet registries, CAIP chain-ID fields in wallet policy/audit surfaces, or cross-chain tooling without additional context.

This MIP proposes adopting [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) as the **external** network identification format for Midnight, using the proposed namespace `midnight` and a reference portion defined in this specification.

**Working recommendation (not canonical):** human-readable network names (Cosmos-style), for example `midnight:mainnet` and `midnight:preprod`, mapped one-to-one from existing Midnight `NetworkId` strings verified in [midnight-wallet `NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts). These examples are **illustrative** until Midnight core engineering ratifies the reference format and network registry.

**Open core decision:** whether human-readable network names are sufficient, or whether genesis-hash or hybrid references are required for consensus-verifiable uniqueness.

The specification covers CAIP-2 grammar constraints, parsing rules, an illustrative network registry (pending core ratification), resolution guidance, and a backwards-compatible transition strategy (dual acceptance of legacy and CAIP-2 forms, canonicalizing to CAIP-2 in new APIs). It documents genesis-hash, numeric, and hybrid reference formats as alternatives. Scope is limited to CAIP-2 chain identifiers; account IDs (CAIP-10), address encoding, bridge semantics, and ledger consensus are out of scope unless Midnight core engineering determines they are required.

This MIP responds to [MPS-0003: CAIP-2 Compliant Network Identifiers for Wallet Ecosystem Integration](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md).

**Submission scope:** This draft is intended for review in [midnight-improvement-proposals](https://github.com/midnightntwrk/midnight-improvement-proposals) only. ChainAgnostic namespace registration and third-party wallet registry updates (e.g. OWS) are **explicitly deferred** until this MIP is Accepted and core ratifies identifiers.

## Motivation

### Problem

Midnight network identifiers today are simple strings defined in wallet abstractions and consumed across the stack. Verified against the live wallet SDK source ([`NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts), 2026-05-30):

```typescript
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

Cross-chain wallet interoperability research—including work at **Ura Labs** that informed discussion in [OWS #222](https://github.com/open-wallet-standard/core/issues/222)—highlighted integrator friction with unqualified Midnight network strings. That research motivated follow-on specification work; it does not define canonical identifiers.

**Midnight Foundation, MIP editors, and Midnight core engineering retain sole authority** over reference format, registry contents, and acceptance of this MIP.

## Specification

### Scope

This MIP specifies:

- CAIP-2 grammar constraints applicable to Midnight
- The **proposed** CAIP-2 namespace `midnight`
- The format of the CAIP-2 `reference` portion (working recommendation and documented alternatives)
- An **intended** mapping between CAIP-2 identifiers and legacy Midnight network strings (pending core ratification)
- Parsing, validation, and canonical serialization rules
- Resolution method guidance for a future ChainAgnostic namespace registration
- Backwards-compatible transition requirements for APIs and SDKs

This MIP does **not** specify bridge protocols, CAIP-10 account IDs, address formats, or ledger-embedded `network_id` serialization except where Midnight core engineering confirms a consensus-level change is required.

### Non-goals

The following are explicitly out of scope:

- Bridge semantics, cross-chain deposit/claim/refund flows, or application-specific routing
- CAIP-10 account identifier adoption
- Changes to Bech32m address encoding (`mn_addr1…`, `mn_addr_preprod1…`, shielded address formats) unless Midnight core requires it for security
- Ledger consensus, block production, or ZK circuit changes unless Midnight core confirms they are required to adopt CAIP-2
- Renaming or consolidating Midnight test environments (`qanet`, `devnet`, etc.)—register as-is unless core decides otherwise
- WalletConnect / CAIP-25 session protocol details beyond registering chain IDs
- **ChainAgnostic namespace PRs** — deferred until this MIP is Accepted
- **Third-party wallet registry updates** (including OWS) — deferred until identifiers are ratified here

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
- CAIP-2 sets grammar only; it does **not** mandate genesis hash vs network name—the namespace spec decides.

**Midnight grammar check:** `midnight` is a legal namespace (8 characters, at the CAIP-2 maximum of the 3–8 limit). All reference candidates under consideration—`mainnet`, a truncated genesis hash, a numeric id, or a `mainnet-<hashprefix>` hybrid—fit the 1–32 character reference rule.

Implementations MUST reject malformed identifiers at API boundaries (invalid namespace, empty reference, illegal characters).

### Namespace

The **proposed** CAIP-2 namespace for Midnight blockchains is:

```
midnight
```

Properties:

- Length: 8 characters (maximum allowed by CAIP-2 `[-a-z0-9]{3,8}`)
- Lowercase only
- Refers to the Midnight blockchain platform and its deployed networks

Registration of this namespace at [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces) is a prerequisite for ecosystem-wide interoperability and SHOULD follow **Acceptance** of this MIP (or a successor with the same namespace and reference rules). **No ChainAgnostic PR is proposed as part of this submission.**

### Core Review Decision (reference format)

> **This is the primary open decision for Midnight core engineering.** Most other elements of this MIP can be specified from public standards and existing Midnight SDK strings. The reference format choice is not fully settled by MPS-0003 (a problem statement, not a normative decision).

**Core engineering review required:** Confirm whether **human-readable network names** (Option A) are canonical enough for Midnight's CAIP-2 references, or whether **genesis-hash** (Option B) or **hybrid name + hash prefix** (Option D) references are required for stronger consensus-verifiable uniqueness.

Dependent question: Can Midnight expose a **stable public genesis-hash constant** per network (and encoding rules) if genesis-hash or hybrid references are chosen?

Until core confirms, this MIP treats **Option A (network name)** as the **working recommendation only—not canonical**.

### Reference format

> **Review gate:** The choice of reference format below is a **working recommendation** for community review. **Final canonical reference format is subject to Midnight core engineering review** and may change before this MIP reaches Accepted status.

#### Working recommendation: network name (Option A)

The reference portion **may** be the existing Midnight network name string, lowercase, matching the well-known `NetworkId` values in [midnight-wallet `NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts).

Syntax:

```
reference = network-name
where network-name matches [-_a-zA-Z0-9]{1,32} per CAIP-2
```

In practice, all well-known Midnight network names observed today are lowercase ASCII and fit the direct reference pattern without hashing.

**Illustrative examples only—not canonical until ratified:**

```
midnight:mainnet
midnight:preprod
midnight:preview
```

Additional illustrative mappings from verified SDK strings (also **not canonical**):

```
midnight:testnet
midnight:qanet
midnight:devnet
midnight:undeployed
```

This follows the Cosmos namespace pattern (`cosmos:cosmoshub-4`) and aligns with MPS-0003 Appendix B. Other ecosystems use human-readable network-name references in a similar style; those precedents are informative only and not normative for Midnight.

#### Documented alternatives (not selected in this draft)

Reviewers MUST understand the following alternatives evaluated in MPS-0003 Open Question #1. Core engineering may adopt one instead of, or in combination with, Option A.

| Option | Example | Ecosystem precedent | Summary |
|--------|---------|---------------------|---------|
| **B. Genesis hash** | `midnight:91b171bb158e2d38…` | Solana, Bitcoin, Polkadot | Truncated genesis hash; cryptographically verifiable; poor UX. Polkadot uses first 16 bytes of genesis hash as lowercase hex, resolved via `chain_getBlockHash(0)`. Midnight does not currently expose a public genesis-hash constant (MPS-0003). |
| **C. Numeric ID** | `midnight:1`, `midnight:109` | EIP-155 (`eip155:1`) | Familiar to EVM developers; no established public Midnight numeric chain ID; internal values risk confusion with unrelated numeric namespaces. |
| **D. Hybrid name + hash prefix** | `midnight:mainnet-91b171bb` | — | Readability + verifiability; complex parsing; 32-char reference limit constrains hash prefix length. |
| **E. Hashed fallback** | `midnight:hashed-0204c92a0388779d` | Cosmos (long `chain_id`s) | Only needed if a reference exceeds 32 characters. |

Midnight core should confirm whether Midnight's CAIP-2 reference should optimize for **wallet readability and SDK alignment** (Option A) or **consensus-verifiable uniqueness** (Options B/D).

If core selects Option B, D, or E, this MIP MUST be amended before namespace registration. If Option C is selected, Midnight Foundation MUST publish an authoritative numeric ID table distinct from internal WASM/ledger constants unless explicitly aliased.

### Illustrative network registry (pending core ratification)

> **No CAIP-2 identifier in this table is canonical.** Identifiers such as `midnight:mainnet` and `midnight:preprod` are **working recommendations for review** until Midnight core publishes a ratified registry as part of MIP acceptance or amendment.

The following rows are derived from verified [midnight-wallet `NetworkId.ts`](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts) (seven well-known values). Additional environments (e.g. `govnet` in operator tooling) are reserved for core amendment.

| CAIP-2 identifier (illustrative) | Legacy `NetworkId` | Notes |
|----------------------------------|-------------------|-------|
| `midnight:mainnet` | `mainnet` | Production Midnight network |
| `midnight:preprod` | `preprod` | Pre-production testnet |
| `midnight:preview` | `preview` | Preview testnet |
| `midnight:testnet` | `testnet` | Legacy/generic label; inclusion pending core decision on whether `testnet` remains distinct from `preprod`/`preview` |
| `midnight:qanet` | `qanet` | QA environment |
| `midnight:devnet` | `devnet` | Development environment |
| `midnight:undeployed` | `undeployed` | Local emulator / undeployed ledger parameters |
| *(reserved)* | `govnet`, others | To be added by core amendment |

**If Option A is adopted and this registry is ratified**, the following mapping rules apply:

1. CAIP-2 identifier = `midnight:` + legacy string (both lowercase).
2. No two legacy strings map to the same CAIP-2 identifier.
3. Silent aliasing (e.g. mapping `qanet` → `preview`) is forbidden; distinct environments receive distinct CAIP-2 IDs even when operator tooling maps multiple environments to shared off-chain infrastructure **(environment naming only; no bridge or settlement semantics)**.
4. **After Active:** the ratified registry is authoritative for translation between CAIP-2 and legacy forms. **Before Active:** the table is illustrative for integration planning only.

### Parsing and validation

Implementations MUST:

1. Split on the first `:` only: `namespace`, `reference`.
2. Verify `namespace === "midnight"`.
3. Verify `reference` matches `[-_a-zA-Z0-9]{1,32}` per CAIP-2.
4. **After Active:** verify `reference` against the **ratified** registry. **Before Active:** implementations SHOULD NOT treat unratified illustrative IDs as canonical.

Canonical serialization:

- Namespace MUST be lowercase `midnight`.
- Reference MUST use the ratified registry spelling (all observed Midnight network names today are lowercase).

Legacy parsing (transition period):

- A bare legacy string `mainnet` (no colon) MAY be accepted at API boundaries where historically supported.
- When accepted, implementations SHOULD canonicalize to the ratified CAIP-2 form (e.g. `midnight:mainnet` if Option A is adopted) for storage in new APIs and external-facing responses.

#### Reference implementation sketch (non-normative)

```typescript
// Illustrative API surface for wallet/SDK integrators
parseCaip2Midnight(id: string): { namespace: 'midnight'; reference: string } | Error
legacyToCaip2(legacy: string): string   // e.g. 'preprod' → 'midnight:preprod' when ratified
caip2ToLegacy(id: string): string        // inverse when reference is registry-known
```

### Resolution method

For a future ChainAgnostic namespace registration, the Midnight namespace specification MUST document how a client resolves a `reference` to a live blockchain.

**Working resolution method (Option A):**

1. Client connects to a Midnight node or indexer for the intended environment.
2. Client queries the network identifier returned by the service (indexer GraphQL network field, node configuration, or equivalent—**exact endpoint subject to core confirmation**).
3. Client maps the returned legacy string to CAIP-2 via the ratified registry.
4. Client verifies the constructed CAIP-2 identifier matches the expected value before signing or submitting transactions.

**Working resolution method (Option B, if adopted):** query genesis block hash from node (analogous to Polkadot `chain_getBlockHash(0)` or Solana `getGenesisHash`), truncate/encode per namespace rules, compare to expected reference.

Optional hardening (future amendment): compare against a published genesis hash even when Option A is canonical.

Clients MUST NOT rely solely on a DApp-supplied chain ID without corroborating against node/indexer state.

### API surfaces

The following Midnight components SHOULD accept CAIP-2 identifiers in new API versions while maintaining legacy support during transition:

| Surface | Current behavior | Target behavior |
|---------|------------------|-----------------|
| `@midnight-ntwrk/midnight-js-network-id` | `setNetworkId('preprod')` | Accept CAIP-2; canonicalize internally |
| Wallet SDK / facade | Legacy `networkId` string | Dual-parse; document canonical form |
| DApp connector `connect(networkId)` | Plain `'mainnet'` | Accept CAIP-2 and legacy during transition |
| Indexer GraphQL | Lowercase legacy string | Accept CAIP-2 in filters; return CAIP-2 in new schema fields |

Downstream wallet registries and standards such as OWS may use the ratified Midnight CAIP-2 identifiers after this MIP reaches consensus. Specific OWS metadata, signer, and wallet integration changes are out of scope for this MIP.

Deprecation:

- Legacy bare strings SHOULD continue to work during a **transition period (suggested minimum: six months, subject to core release planning and migration analysis per MPS-0003)** with deprecation warnings in logs/headers where applicable.
- After the deprecation window, bare strings MAY be rejected in a coordinated major SDK release, subject to core release planning.

### Versioning

The specification version is `0.1.0-draft`. Amendments that change the reference format or registry mappings require a MIP revision or superseding MIP.

## Rationale

### Why CAIP-2

[CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) is the Final chain-identification standard used by ChainAgnostic wallets, indexers, and multi-chain tooling. Adopting it removes bespoke translation at every integration boundary and prevents namespace collisions with Ethereum, Solana, and other ecosystems. The grammar accommodates Midnight without constraining the reference-format choice.

### Why namespace `midnight`

- Matches the platform name used in documentation, SDK packages (`@midnight-ntwrk/*`), and ecosystem discourse.
- Length 8 fits CAIP-2 namespace rules (`[-a-z0-9]{3,8}`) and is the maximum permitted length.
- No known collision with existing ChainAgnostic namespaces at time of drafting.

### How other ecosystems structure references

| Camp | Examples | Reference style | Resolution |
|------|----------|-----------------|------------|
| Network name | Cosmos (`cosmos:cosmoshub-4`), Tron, Sui, NEAR | Human-readable name | Node status / registry table |
| Numeric | EVM (`eip155:1`) | Chain ID integer | `eth_chainId` |
| Genesis hash | Solana, Bitcoin (BIP-122), Polkadot | Truncated genesis hash | `getGenesisHash`, block 0 hash, etc. |

Cosmos resolves via Tendermint `/status` → `chain_id`, hashing only if the name exceeds 32 characters. Polkadot deliberately uses genesis block hash (first 16 bytes, lowercase hex) because genesis hash is load-bearing in network/consensus and resolution is the consumer's job.

Midnight core should confirm whether Midnight's CAIP-2 reference should optimize for wallet readability or consensus-verifiable uniqueness.

### Why network-name references (Option A) as working recommendation

| Criterion | Option A (name) | Option B (genesis) | Option C (numeric) | Option D (hybrid) |
|-----------|-----------------|--------------------|--------------------|-------------------|
| Matches verified SDK strings | Yes | No | No | Partial |
| Human-readable | Yes | No | Moderate | Yes |
| Self-verifying from node | No (requires registry) | Yes | Yes (if IDs published) | Partial |
| MPS-0003 alignment | Appendix B soft recommendation | Open Question #1 | Open Question #1 | Open Question #1 |
| Ledger / ZK impact if metadata-only | Minimal | Minimal | Minimal | Minimal |

**Reasons for Option A as working recommendation (not final decision):**

1. **MPS-0003** leans network-name in Appendix B while keeping Open Question #1 formally open.
2. **Migration:** deterministic 1:1 mapping from verified `NetworkId.ts` strings minimizes SDK, indexer, and DApp breakage.
3. **Cosmos** and similar ecosystems provide readable precedents.

Option B remains appropriate if Midnight core prioritizes trustless verification and can publish stable genesis constants. Option D mitigates the readability/verification tradeoff. Option C should not be adopted without a published official numeric ID scheme.

### Downstream wallet standards (informational only)

Multi-chain wallet tooling expects `namespace:reference` chain IDs per [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md). Community discussion in [OWS #222](https://github.com/open-wallet-standard/core/issues/222) surfaced integrator interest in Midnight identifiers; that discussion is **context only** and does not define canonical identifiers.

Downstream wallet registries and standards such as OWS may use the ratified Midnight CAIP-2 identifiers after this MIP reaches consensus. Specific OWS metadata, signer, and wallet integration changes are out of scope for this MIP. **No OWS or ChainAgnostic PR is proposed as part of this submission.**

### Why dual support during transition

Hard cutover risks breaking shipped wallets, DApps, and operator tooling that persist legacy strings. Dual acceptance with canonicalization balances safety and a clear end state, consistent with optional migration targets discussed in MPS-0003.

### Why decouple from address format

Bech32m address HRPs (`mn_addr_preprod1…`) already encode network context for addresses. CAIP-2 identifies the **chain** for wallet routing and policy, not the address serialization format. Merging the two would break existing address validation and test vectors.

## Path to Active

### Acceptance Criteria

This MIP reaches **Accepted** when:

1. MIP editors assign a number and merge the draft after community commentary (≥2 weeks per MIP-1).
2. Midnight core engineering confirms the reference format (Option A or amended alternative)—see **Core Review Decision**.
3. The illustrative network registry is ratified or amended by core.
4. No unresolved security objections from reviewers regarding chain-spoofing or migration windows.

This MIP reaches **Active** when:

1. Reference format and ratified registry are implemented in released wallet SDK and indexer versions designated by Midnight Foundation.
2. At least one first-party wallet connector documents CAIP-2 acceptance.
3. Migration guide published for wallet and DApp developers.

Downstream ecosystem work may include ChainAgnostic namespace registration, wallet registry updates, and OWS chain metadata once Midnight core ratifies the identifier format. That work is **not** a blocking condition for this MIP reaching Active.

### Implementation Plan

1. **Phase 0 — Specification (this MIP):** Community review in midnight-improvement-proposals; finalize reference format and registry with core review.
2. **Phase 1 — Adapter layer:** Add parse/canonicalize helpers in `midnight-js-network-id` and wallet abstractions; no ledger changes.
3. **Phase 2 — API exposure:** Indexer and connector return/accept CAIP-2; deprecation warnings for legacy-only clients.
4. **Phase 3 — Deprecation:** Remove legacy bare-string acceptance in a coordinated major SDK release after the transition period.

Downstream ecosystem work (ChainAgnostic namespace registration, third-party wallet registries, OWS chain metadata) follows **after** Midnight core ratifies identifiers and is out of scope for this MIP's implementation plan.

Ledger-embedded `network_id` changes, if required, MUST be scheduled as a separate network upgrade with explicit fork coordination—not silently bundled into Phase 1.

## Backwards Compatibility Assessment

### Impact summary

| Area | Hard fork required? | Compatibility approach |
|------|---------------------|------------------------|
| SDK `setNetworkId` | No | Dual-parse legacy and CAIP-2 |
| Persisted wallet config | No | Migrate on read; canonicalize on write |
| Indexer queries | No | Additive schema fields; legacy filters supported during transition period |
| Bech32m addresses | No | Unchanged |
| ZK proofs / ledger state | Unlikely if metadata-only | Prefer metadata-layer adoption; avoid changing serialized `network_id` unless core requires |
| Third-party DApps | Low–medium | Continue working via legacy strings during transition period |

### Proposed legacy ↔ CAIP-2 mappings (illustrative)

Derived from verified `NetworkId.ts`; **not canonical** until registry ratification:

```
mainnet     ↔ midnight:mainnet     (illustrative)
preprod     ↔ midnight:preprod     (illustrative)
preview     ↔ midnight:preview     (illustrative)
testnet     ↔ midnight:testnet     (illustrative)
qanet       ↔ midnight:qanet       (illustrative)
devnet      ↔ midnight:devnet       (illustrative)
undeployed  ↔ midnight:undeployed   (illustrative)
```

Implementations MUST NOT treat `testnet` as an alias for `preprod` or `preview` unless Midnight core explicitly deprecates `testnet` and documents the alias in an amendment.

### User impact

End users SHOULD NOT need manual migration if wallets auto-canonicalize on upgrade. Developers SHOULD update configuration docs and environment examples to show CAIP-2 as preferred form once the registry is ratified.

## Security Considerations

### Chain spoofing

**Risk:** A malicious DApp or RPC endpoint supplies `midnight:mainnet` while the wallet is connected to `midnight:preprod`, inducing wrong-chain signing.

**Mitigations:**

- Wallets MUST corroborate CAIP-2 identifiers against node/indexer-reported network strings before high-value operations.
- Wallets SHOULD display human-readable network names prominently in confirmation UI.
- **After Active:** registry lookups MUST reject unknown `midnight:{reference}` values in production builds unless explicitly in dev/local mode.

### Identifier injection

**Risk:** Malformed identifiers bypass validation and corrupt storage.

**Mitigations:**

- Strict CAIP-2 regex validation at parse time.
- Split on first `:` only; validate reference separately (inputs like `midnight:mainnet:extra` yield reference `mainnet:extra`, which fails CAIP-2 length/character rules or registry lookup).

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
|-----------|--------|
| `midnight-wallet` / `NetworkId.ts` | Add CAIP-2 parse, format, registry lookup; retain legacy constants |
| `@midnight-ntwrk/midnight-js-network-id` | Accept CAIP-2 in `setNetworkId`; document canonical form |
| Wallet SDK facade | Dual-parse at initialization |
| Indexer | Expose CAIP-2 in API; validate incoming chain filters |
| DApp connector | Document and accept CAIP-2 in `connect()` |
| Documentation site | Update configure-providers guide with CAIP-2 examples |

### Components explicitly deferred

| Component | Condition for inclusion |
|-----------|-------------------------|
| ChainAgnostic namespace registration | After MIP Accepted; separate downstream work |
| Third-party wallet registry updates | After registry ratified; separate downstream work |
| Ledger `StandardTransaction.network_id` | Only if core confirms embedding CAIP-2 in consensus-critical state |
| Bech32m encoder | Only if core mandates address/chain ID unification |
| Bridge / cross-chain contracts | Out of scope |

### Dependencies

- [MPS-0003](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md) (problem statement)
- [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md)
- Midnight core ratification of reference format and registry (blocking for downstream work)

### Open items for Midnight core engineering

1. **Reference format (primary):** Confirm Option A (network name), Option B (genesis hash), Option D (hybrid), or another MPS-0003 alternative—and whether genesis-hash verification is required even if Option A is canonical.

2. **Genesis hash availability (dependent):** If Option B or D is chosen, publish stable genesis-hash constants per network and encoding/truncation rules for the namespace spec.

3. **Registry ratification:** Confirm the authoritative network list (including whether `testnet` remains distinct and whether local-only `undeployed` appears in public registries).

## Testing

### Unit tests

- Parse valid CAIP-2 identifiers for all illustrative registry rows.
- Reject invalid namespace, empty reference, overlong reference, illegal characters.
- Round-trip: legacy → CAIP-2 → legacy for all illustrative registry rows (once ratified).
- Reject silent aliasing (e.g. `qanet` must not map to `preview`).

### Integration tests

- Wallet connects to Preprod node; corroborates illustrative `midnight:preprod` against indexer-reported network string (after ratification).
- DApp connector accepts both `preprod` and ratified CAIP-2 form during transition.

### Regression tests

- Existing DApps using `setNetworkId('preprod')` continue to function unchanged during transition period.
- Address generation and validation unchanged for all networks.
- ZK proof generation and verification unchanged (metadata-layer adoption only).

### Test vectors (illustrative)

```
# Valid grammar (illustrative references — not canonical until ratified)
midnight:mainnet
midnight:preprod
midnight:undeployed

# Invalid
Mainnet:mainnet          # wrong namespace case
midnight:                # empty reference
midnight:mainnet:extra   # reference becomes mainnet:extra — invalid or unknown
ethereum:1               # wrong namespace
```

## References

- [CAIP-2: Blockchain ID Specification](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) (Final)
- [MPS-0003: CAIP-2 Compliant Network Identifiers for Wallet Ecosystem Integration](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md)
- [MIP-1: Midnight Improvement Proposal Process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-mip-process.md)
- [MIP template](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-template.md)
- [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces)
- [Cosmos namespace (CAIP-2)](https://github.com/ChainAgnostic/namespaces/blob/main/cosmos/caip2.md)
- [Solana namespace (CAIP-2)](https://github.com/ChainAgnostic/namespaces/blob/main/solana/caip2.md)
- [midnight-wallet NetworkId.ts](https://github.com/midnightntwrk/midnight-wallet/blob/main/packages/abstractions/src/NetworkId.ts) (verified 2026-05-30)
- [Midnight configure providers / network ID](https://docs.midnight.network/guides/configure-providers)
- [Midnight Indexer v3.0.0 release notes (lowercase network ID)](https://docs.midnight.network/relnotes/midnight-indexer/midnight-indexer-3-0-0)
- [Midnight DApp connector InitialAPI](https://docs.midnight.network/api-reference/dapp-connector/type-aliases/InitialAPI)
- [OWS #222 — Chain support discussion (informational)](https://github.com/open-wallet-standard/core/issues/222)

## Acknowledgements

- Bob Blessing-Hartley and Midnight Foundation contributors for [MPS-0003](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md)
- ChainAgnostic community for CAIP-2 and namespace specifications

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission to the official Midnight Improvement Proposals repository requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for contributions to the Foundation.
