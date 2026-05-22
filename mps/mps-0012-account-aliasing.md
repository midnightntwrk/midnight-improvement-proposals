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

---
MPS: 0012  
Title: Human-Readable Aliasing for Midnight Accounts  
Authors:
  - Karmel Abdeljawad <Karmoola>
  - Nick Stanford <nstanford5>  
Status: Proposed  
Category: Standards  
Created: 21-May-2026  
Requires: none  
Replaces: none  

---

## Abstract

A single Midnight wallet seed deterministically produces three distinct on-chain addresses — Shielded, Unshielded, and Dust — each with its own Bech32m encoding (`mn_shield-addr1…`, `mn_addr1…`, `mn_dust1…`). Future capabilities, such as Sig.Network-style external-chain address derivation, will add more. Asking users to manage, copy, and disambiguate these addresses raises onboarding friction and increases the risk of misdirected funds.

Several ecosystem partners are independently building human-readable naming layers. Without a network-level standard, these efforts risk namespace fragmentation: two providers could allow the same human-readable name to bind to different accounts, or the same name could resolve to different addresses across wallets, indexers, and dApps. There is no authoritative on-chain primitive guaranteeing that a name is bound to exactly one Midnight account globally.

This MPS identifies the absence of a single, authoritative standard for binding human-readable names to Midnight accounts and to the family of addresses derived from those accounts. It defines the problem space so that future MIPs can specify a registry, resolver interface, and rendering rules that wallets, dApps, indexers, and partner services can adopt uniformly.

## Vision

A user who completes Midnight onboarding receives one human-readable name — for example, `alice` — under the canonical Midnight namespace that resolves uniformly across every Midnight wallet, dApp, indexer, block explorer, and partner service. The name is the user's single public handle. It routes to the correct underlying address for the operation being performed: shielded transfers go to the Shielded address, unshielded operations to the Unshielded address, DUST sponsorship to the Dust address, and future cross-chain deposits to the appropriate externally-derived address. The user does not see, copy, or distinguish between these underlying addresses unless they explicitly opt into a power-user mode.

Two users cannot hold the same name simultaneously. When a user recovers their account on a new device from their mnemonic, their existing name re-points to the recovered account without becoming available for re-registration by anyone else.

## Problem

The current state of Midnight account addressing creates four compounding problems.

**1. Multiple addresses per account:** A single seed deterministically derives three user-facing addresses via BIP-32 along the path `m/44'/2400'/account'/role/index`: a Shielded address (Zswap role), an Unshielded address (Night role), and a Dust address (Dust role). Each uses its own Bech32m human-readable part (`mn_shield-addr`, `mn_addr`, `mn_dust`) and is intended for a distinct class of operation. Users must currently know which address to share for which interaction; copy-paste errors and "wrong address type" mistakes are a leading source of failed or stuck transactions during onboarding. Future capabilities — for example Sig.Network-derived external-chain addresses — will multiply this further.

**2. Uncoordinated naming providers:** Without a network-level standard, each ecosystem provider (Midnames, 1AM, and others) may publish a registry contract with its own uniqueness rules, resolver interface, and dispute semantics. Wallets and dApps must then choose which provider(s) to support, and a user named `alice` in one resolver may not be the same `alice` in another. This fragments the developer surface and the user-visible namespace simultaneously.

**3. No authoritative duplicate prevention:** Uniqueness of human-readable names must hold at any point in time. If two providers each maintain independent state, "uniqueness" only holds within a provider, not across the network. There is no current network-level primitive that guarantees a name is bound to exactly one account globally, on-chain and verifiable from chain state alone.

**4. No agreed rendering rules:** Even given a correct resolution, wallets and dApps have no shared guidance on how to render names safely — handling of homoglyphs, mixed scripts, case-folding, normalization, and confusable Unicode is left to each implementer. Inconsistent rendering creates phishing vectors that scale with adoption.

## Use Cases

**Use Case 1: First-time send:** Alice completes onboarding, receives the name `alice`, and shares it with Bob over a messaging app. Bob pastes `alice` into his Midnight wallet and sends her shielded tokens. He never sees a raw Bech32m address. The wallet resolves `alice` to Alice's Shielded address, regardless of whether Bob is using Lace, 1AM, or any other conforming wallet.

**Use Case 2: Cross-context routing:** A dApp asks the user to register their name as the recipient of a recurring DUST sponsorship. The dApp uses `alice` and the resolver returns Alice's Dust address. The same `alice` used in a shielded-token gift flow routes to Alice's Shielded address. The user does not select an address type; the operation context determines routing.

**Use Case 3: Recovery:** Alice loses her device. She restores her wallet from her BIP-39 mnemonic on a new device. Because the addresses derive deterministically from the mnemonic, the same Shielded/Unshielded/Dust addresses reappear, and the name `alice` continues to resolve to the recovered account. No one was able to re-register `alice` while Alice was offline.

**Use Case 4: Block-explorer display:** An explorer indexes a shielded transaction and needs to display the sender or recipient (where permitted). With a single canonical resolver, the explorer shows `alice` consistently across views. Without one, the explorer must choose a provider (potentially showing different names to different users) or fall back to raw addresses.

**Use Case 5: Future external-chain deposit:** Once Midnight integrates a cross-chain derivation capability (e.g., Sig.Network-style chain signatures), Bob deposits assets from another chain to `alice`. The resolver returns the externally-derived address on the source chain. The user experience is unchanged: one name, correct routing.

## Goals

1. **One name per user, resolving everywhere:** A single human-readable name MUST be sufficient for any send-receive-identify operation a user performs on Midnight, abstracting over Shielded, Unshielded, and Dust addresses, and over future external-chain addresses derived from the same account.

2. **Global uniqueness, enforced on-chain:** At any point in time, a given name MUST resolve to at most one account, and this property MUST be verifiable from chain state alone — not from a particular provider's off-chain index.

3. **Recovery-safe:** Account-recovery flows MUST preserve name ownership across device loss. A name MUST NOT become available for re-registration simply because the owner is temporarily offline.

4. **Phishing-resistant rendering:** The standard MUST give wallets and dApps enough information to safely render names in UI without each implementer reinventing the rules. This includes guidance on normalization, confusable scripts, and the display of unverified or expired names.

5. **Resolver-uniformity across clients:** The same name MUST resolve to the same address across Lace, 1AM, and any conforming wallet, dApp, indexer, or explorer. Behaviour MUST NOT depend on which provider the client happens to consult first.

6. **Privacy-preserving by default:** The aliasing system MUST NOT weaken Midnight's existing privacy guarantees. In particular, resolving `alice → Shielded address` MUST NOT, by itself, reveal more information than directly sharing the Shielded address would.

## Expected Outcomes

If this problem is addressed, Midnight users will see one canonical name during onboarding and use it as their single public handle for the lifetime of their account. Onboarding friction from "which address do I copy?" disappears, as does the misdirected-funds risk it creates. Wallets, dApps, indexers, and explorers converge on a single resolver, so a user's identity is consistent across the ecosystem. Partner naming providers can continue to build value-added services (vanity names, premium tiers, off-chain metadata) on top of a shared on-chain uniqueness primitive, rather than competing on whose name is "real."

Developers gain a single integration target instead of N provider-specific SDKs. Phishing-resistance guidance reduces the implementation burden of safe rendering. Adoption of Midnight wallets and dApps improves as the address-management UX approaches that of established naming systems on other chains.

## Open Questions

1. **On-chain primitive:** Should global uniqueness be enforced by a registry contract written in Compact, by a protocol-level primitive in the ledger, or by a hybrid (e.g., a sealed-ledger contract endorsed by the Foundation)? What are the trust trade-offs of each?

2. **Account binding:** What is the canonical "account" a name binds to, given that the seed produces three independent addresses? Candidates include the HD root public key, a derived account identifier, or a Shielded-address commitment. The choice affects recovery, multi-device, and external-chain extensibility.

3. **Reverse resolution:** Should a Midnight address resolve back to a name (e.g., for explorer display)? If so, which of the three addresses anchors the reverse mapping, and how are conflicts resolved if a user re-binds their name to a different account?

4. **Renewal and expiry:** Are names held in perpetuity, or do they require renewal? Expiry creates squatting protection but risks losing names due to inactivity — including during long account-recovery windows.

5. **Namespace governance:** Who controls the root Midnight namespace? Is registration permissionless, gated by proof-of-uniqueness checks, or governed by an authority? What is the dispute process for trademarked or reserved names?

6. **Privacy of registrations:** Should the registration of `alice → account-X` itself be public, shielded, or selectively disclosable? A fully public registry exposes the social graph; a fully shielded registry complicates uniqueness enforcement.

7. **Sig.Network and external-chain derivation:** If/when Midnight adopts cross-chain address derivation, how does a name extend to cover those addresses? Is the binding implicit (any future derivation is automatically reachable via the name) or explicit (the user opts each external chain in)?

8. **Migration of existing partner registries:** Midnames, 1AM, and others have already issued names off-chain or on partner registries. What is the migration path, and how are conflicts between pre-existing partner registrations resolved when consolidated onto the canonical namespace?

9. **Internationalization and confusables:** What normalization form (e.g., UTS #46, ENSIP-15) applies? Which scripts are permitted, and how are mixed-script attacks (e.g., Cyrillic `а` vs Latin `a`) prevented at the registry layer rather than the rendering layer?

10. **Wallet integration surface:** Does the standard mandate a specific resolver API (e.g., a JSON-RPC method, a GraphQL query, a Compact circuit), or is it transport-agnostic so long as on-chain truth is the same?

## Recommended MIPs

The following MIP areas are recommended to address the problems above. Each is a candidate for a dedicated MIP authored under MIP-0001.

- **MIP: Canonical Midnight Name Registry** Specify an on-chain registry contract (or protocol primitive) that enforces global name-to-account uniqueness, supports registration, transfer, renewal (if applicable), and binding to a recovery-safe account identifier. Must specify the account-binding choice raised in Open Question 2.

- **MIP: Resolver Interface and Routing Rules** Define the resolver API and the operation-to-address routing table — i.e., when a wallet/dApp resolves `alice`, which of {Shielded, Unshielded, Dust, external-chain} is returned for a given operation context. Must be transport-agnostic and reference on-chain state as the source of truth.

- **MIP: Name Rendering and Phishing-Resistance Guidance** Specify normalization (e.g., UTS #46 profile), permitted scripts, confusable handling, and required UI affordances (e.g., showing unverified, expired, or homoglyph-flagged names distinctly). Targets wallets, dApps, explorers, and indexers.

- **MIP: Recovery and Multi-Device Semantics** Specify how name ownership survives device loss, mnemonic restore, and account migration; in particular, how a registry prevents re-registration during an owner's offline window without granting permanent un-revocable ownership.

- **MIP: Partner-Registry Migration** Specify a one-time migration path from existing off-chain or partner-specific registrations (Midnames, 1AM, etc.) to the canonical namespace, including conflict resolution where multiple parties hold the same name across providers.

- **MIP: External-Chain Alias Extension** Once cross-chain derivation (e.g., Sig.Network-style chain signatures) is available on Midnight, specify how a single Midnight name extends to cover externally-derived addresses without further user action and without weakening the resolver's privacy posture.

## References

- Midnight wallet HD derivation: `m/44'/2400'/account'/role/index`, BIP-32/BIP-44, defined in `packages/hd/src/HDWallet.ts` of `midnightntwrk/midnight-wallet`.
- Midnight address formats: `packages/address-format/src/index.ts` — `ShieldedAddress` (`mn_shield-addr1…`), `UnshieldedAddress` (`mn_addr1…`), `DustAddress` (`mn_dust1…`).
- MPS-0003: CAIP-2 Compliant Network Identifiers for Wallet Ecosystem Integration — related work on identifier standards.
- MIP-0001: Midnight Improvement Proposal Process.
- ENSIP-15: ENS Name Normalization — prior art for name normalization and confusable handling.
- UTS #46: Unicode IDNA Compatibility Processing — prior art for internationalized name handling.

## Acknowledgements

Thanks to ecosystem partners building naming layers (Midnames, 1AM, and others) for surfacing the underlying coordination problem, and to reviewers and community members who provided feedback on the initial draft.

## Copyright

This MPS is licensed under CC-BY-4.0.
