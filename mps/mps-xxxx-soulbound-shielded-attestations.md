---
MPS: <Number> # assigned by editors
Title: Soulbound and Non-Transferable Shielded Attestations on Midnight
Authors: Harley Hermanson (HarleysCodes)
Status: Proposed
Category: Standards
Created: 25-Jun-2026
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
 See the License for the License for the specific language governing permissions and
 limitations under the License.
-->


## Abstract

The Midnight Passport program (P9) defines the credential substrate for the
network: an on-chain attestation tree (C18), an off-chain issuance flow (C19),
and a selective-disclosure proof primitive (C20). Together these cover
*transferable* credentials — credentials the holder can move to a new wallet
or alias under the issuer's policy. What Passport does not currently specify
is the *non-transferable* specialization: a credential that is bound to one
holder and provably refuses to move, regardless of issuer policy. This MPS
frames that gap and the design space the resulting MIPs will need to cover.
It does not select a solution.

The use cases this gap excludes are concrete and recurring: KYC and
sanctions attestations that a regulated venue must revoke when a holder's
status changes; education and professional credentials whose value is the
identity of the recipient; event tickets with named attendees; governance
delegations to a specific delegate; proof-of-personhood stamps that must
resist Sybil attacks by being non-tradeable; and non-transferable membership
in a DAO or federation. None of these can be expressed on Midnight today
without each project rebuilding its own ad hoc, contract-bound, non-portable
solution that opts out of the Passport transfer semantics.

The recommended holder-binding primitive is a Midnight Passport alias or,
where Passport is out of scope, a per-credential spending key held by the
recipient — both are shielded-address primitives over which
non-transferability can be enforced. Cross-credential correlation (the
same holder identifiable across distinct credentials) is mitigated by
composing with the proposed Domain Separation for Midnight Hash
Constructions MPS, which provides a registry of canonical domain tags
that this MPS recommends for any binding construction.


## Vision

A Midnight where any issuer can mint a credential that is provably bound to
one holder and provably refuses to move, where the holder can prove possession
of the credential at any of Midnight's three disclosure tiers (public,
auditor, private) without revealing unrelated identity, and where the issuer
can revoke the credential on a defined policy without seizing the holder's
other assets. A soulbound attestation on Midnight should be the same kind of
basic primitive that ERC-721 is on Ethereum — boring, composable, and assumed
to exist by every higher-level application — rather than a custom integration
per project.


## Problem

**The Midnight Passport credential substrate covers transferable credentials;
this MPS scopes the non-transferable specialization.** The Passport program's
P9 deliverable — an on-chain attestation tree (C18) plus issuance (C19) and
selective-disclosure proof (C20) — provides the substrate for issuer-issued
credentials with proof-of-membership semantics. P9 does not specify that a
credential *refuses* to transfer; the design assumes the issuer's policy
governs movement, and a Passport credential can in principle be re-issued to
a new alias under that policy. The non-transferable specialization is a
*gap*: there is no standard for a credential whose transfer logic is
explicitly refused at the protocol level, regardless of issuer policy. The
token standards in scope today (MIP-0004 fungible, MIP-0011 native shielded,
the closed MRC-721 / MRC-1155 family) all express transferable tokens; no
standard specifies how to issue a token that *refuses* to transfer, and the
Passport substrate inherits that assumption.

**Soulbound credentials are not a niche use case; they are recurring.** KYC
and sanctions compliance require an attestation that follows a specific
natural person and is revocable on status change; today every regulated venue
on Midnight would have to either trust off-chain processes or design its own
soulbound contract. Education and professional credentials (degrees, medical
licences, bar admissions) carry their meaning from the recipient's identity.
Tickets for events, conferences, and travel require named attendance to
support insurance, security, and refund policy. Governance delegations in
DAOs are only meaningful if the delegate is named and the delegation cannot
be silently re-routed. Proof-of-personhood systems (BrightID-style, Idena,
Worldcoin) are inherently soulbound: their entire value proposition is that
the credential cannot be sold or transferred, because transfer would
re-introduce Sybil vulnerability. DAO and federation membership tokens that
carry rights on-chain (vote, propose, claim allocation) must not be tradeable
without the issuer forfeiting governance safety.

**Why the Passport substrate (C18) is the right starting point for
non-transferability.** On transparent chains, a soulbound token can be
implemented by simply omitting a `transfer` function from the contract;
the lack of a function is enforced by the contract's code. On Midnight,
a *native* shielded coin (MIP-0011) transfers at the protocol level via
`sendImmediateShielded` and `sendShielded` — the contract is not in the
path — so a soulbound credential cannot be a native shielded coin.
Midnight Passport's P9 solution is to anchor credentials not as
shielded coins but as Merkle-tree leaves in an on-chain attestation
tree (C18), where the issuer's policy governs movement under a
credential-issuance flow (C19). The non-transferable specialization is
therefore a *constraint on the C18 attestation tree*: it specifies that
a credential leaf cannot be re-issued to a new alias and cannot be
re-presented by a different holder, regardless of issuer policy. This is
a different shape of constraint than C18 specifies, and the standard
does not currently cover it.

**The non-transferable disclosure proof refinement is unsolved.** Passport's
C20 selective-disclosure proof primitive covers the general case: a
holder proves membership in an attestation-tree leaf (C18) without
revealing the underlying attribute. The non-transferable specialization
introduces a proof refinement that C20 does not require: the proof must
demonstrate that the credential *cannot be re-presented by a different
holder*. In the transferable case this is automatic — the holder has
the credential — but in the non-transferable case it is load-bearing,
because the proof must not itself be a transfer vector. The proof must
also continue to not leak the holder's other Midnight identity, must not
leak which credential they hold to anyone other than the verifier, and
must compose with Midnight's three-tier disclosure model (public /
auditor / private). This is the refinement MIP-B addresses, on top of
C20 rather than from scratch.

**Revocation interacts with credential-side and wallet-side recovery.**
When the issuer revokes a credential (e.g. expired licence, sanctions
hit, lost diploma, terminated employment), the revocation must propagate
to verifiers without the holder being able to present a stale proof.
When the holder loses the wallet that controls the credential, recovery
must be possible without breaking the non-transferability property — the
credential cannot move to the new wallet as a transfer, but it may need
to be re-bound to the new wallet under a defined recovery policy. The
Passport program provides wallet-side recovery (sharded Google / Apple /
social recovery) for the holder's wallet itself; this MPS scopes the
*credential-side* re-binding that must not be expressible as a transfer.
MPS-0018 (multi-key account custody) addresses the asset-recovery side
of this for fungible assets; the soulbound case adds a constraint that
recovery must not be expressible as a transfer, and the credential-side
re-binding is not currently covered.


## Use Cases

**KYC and sanctions compliance at a regulated venue.** A regulated exchange
on Midnight needs to verify that a depositing user holds a valid KYC
attestation from an approved issuer. The attestation must (a) belong to the
depositing user and not be transferable, because the exchange's regulator
requires per-user limits; (b) be revocable by the issuer when the user's
status changes, with the revocation visible to the exchange within a defined
window; (c) be provable by the user to the exchange without the user
revealing other Midnight identity. Today each venue would have to design
its own contract; there is no shared primitive.

**Education credential presented to an employer.** A university issues a
degree as a soulbound token bound to the graduate's Midnight address. The
graduate proves possession of the degree to a prospective employer via a
zero-knowledge proof that discloses only "this address holds a valid degree
from university X dated within window Y." The graduate cannot sell the
degree. The university can revoke (e.g. for fraud discovered later). A
verifier checking the proof does not learn the graduate's address or other
Midnight activity.

**Event ticket with named attendee.** A conference issues tickets bound to
specific attendees. Insurance and refund policy depend on the ticket not
being transferable on the secondary market without the issuer's consent.
The attendee proves ticket possession at the door via a short-range
disclosure proof; the venue verifies without learning the attendee's broader
Midnight identity.

**Governance delegation.** A token holder delegates voting power to a named
delegate. The delegation is a soulbound attestation issued by the holder to
the delegate. The delegate's contract can verify the attestation exists and
vote on behalf of the holder; the holder cannot sell or re-route the
delegation without an explicit re-issue. Revocation is the holder's
prerogative.

**Proof-of-personhood stamp.** A proof-of-personhood provider (BrightID,
Idena, Worldcoin-style) issues a non-transferable attestation that a
specific Midnight address is operated by a unique human. The attestation is
the unit of Sybil resistance: it must not be tradeable, or the entire
proof-of-personhood guarantee collapses. Smart contracts gating access on
personhood (e.g. quadratic-voting rounds, airdrop fairness checks) verify
the attestation without learning the holder's address.

**DAO and federation membership with on-chain rights.** A DAO or
federation issues membership tokens that carry vote, propose, or claim
rights. The membership must be non-transferable so that rights cannot be
concentrated by purchase. Issuance, revocation, recovery, and proof all
require standard primitives that today do not exist.


## Goals

The following objectives should be achievable by solutions specified in
MIPs downstream of this MPS.

1. **Non-transferability as a primitive property.** A soulbound attestation
   must be non-transferable by construction, not by the absence of a
   `transfer` function in an ad hoc contract. The non-transferability
   guarantee must hold against both direct transfer attempts and indirect
   paths (e.g. wrapping, proxying, or holding the credential through a
   contract whose code can move it).

2. **Holder-side privacy.** A holder must be able to prove possession of a
   soulbound attestation to a specific verifier without revealing their
   identity, their other Midnight activity, or which credentials they hold to
   anyone other than the verifier. The proof must compose with Midnight's
   three-tier disclosure model.

3. **Issuer-side revocation.** An issuer must be able to revoke a specific
   attestation on a defined policy (expiry, sanctions hit, status change,
   fraud), and verifiers must observe the revocation within a defined
   window. Revocation must not give the issuer the ability to seize the
   holder's other assets.

4. **Recovery without transferability.** When the holder loses the wallet
   that controls a soulbound attestation, recovery must be possible without
   breaking the non-transferability property. The recovery path must be
   defined by the issuer at issuance and must not be expressible as a
   transfer to a different address.

5. **Composability with the Passport credential substrate and existing
   token standards.** Soulbound attestations must compose cleanly with
   Midnight Passport's P9 deliverable (C18 attestation tree, C19
   issuance, C20 selective-disclosure proof), with MIP-0004 (fungible),
   with MIP-0011 (native shielded), and with any ratified successor. A
   soulbound attestation must not require a holder to lock up or expose
   unrelated assets to use it, and the non-transferability constraint
   must not introduce additional verifier integration beyond what
   transferable Passport credentials already require.

6. **Standard disclosure proof.** The proof format a holder uses to show
   possession of a soulbound attestation to a verifier must be standardised
   so verifiers do not need per-issuer integration. The format must support
   selective disclosure at multiple granularity levels (e.g. "I hold *some*
   attestation from issuer X" vs "I hold attestation Y from issuer X with
   attribute Z in range W").

7. **Auditability when required.** When the holder chooses the *auditor*
   disclosure tier, a designated auditor must be able to verify the
   attestation and any associated metadata without learning the holder's
   other activity. The auditor disclosure path must be explicit, not
   emergent.


## Expected Outcomes

If the recommended MIPs are specified and adopted, the Midnight ecosystem
should expect:

- A drop-in primitive that any project can use to issue a non-transferable
  credential without re-implementing the proof, recovery, and revocation
  surface each time. This reduces the cost of building regulated,
  credential-gated, or personhood-gated applications on Midnight.

- Compatibility with the Midnight Passport credential substrate (P9) and
  with existing token standards (MIP-0004, MIP-0011) and identity work
  (MPS-0015 / proposed MAIS) so that a soulbound credential can be issued
  by, gated on, or composed with the rest of the Midnight stack rather
  than sitting beside it. Specifically, the non-transferability
  constraint should not require a verifier to integrate with two
  credential substrates; a verifier integrating with Passport should be
  able to verify a non-transferable credential with no additional
  primitive beyond the Passport substrate plus the
  non-transferability marker this MPS specifies.

- A clear separation of concerns between *issuing* a soulbound credential,
  *holding* it, *proving* possession, and *revoking* it. Each of these
  should be expressible independently so that applications can adopt the
  subset they need.

- Reduced regulatory friction for projects building on Midnight that
  require KYC, sanctions screening, or membership gating. Today each
  project designs its own primitive; a standard makes the regulatory
  conversation reusable.

- A foundation for proof-of-personhood and Sybil-resistance primitives
  that depend on the non-tradeability of their credentials.


## Open Questions

The following questions require further discussion or research before a
solution can be specified.

1. **How does the non-transferability constraint compose with the C18
   attestation tree?** Three candidate shapes are plausible: (a) a flag
   bit on the attestation leaf that the issuance transaction sets and
   that verifiers check; (b) a separate registry of non-transferable
   credentials keyed by Passport alias, anchored to C18 by a secondary
   Merkle root; (c) a sub-tree of C18 whose root is published under a
   known non-transferable domain tag (composing with the Domain
   Separation MPS). Each option has different verifier-cost, revocation
   propagation latency, and on-chain footprint trade-offs. The choice
   also determines whether non-transferability is opt-in per credential
   or a property of the issuer's policy envelope.

2. **How is the holder bound to the credential, and what does the binding
   itself correlate?** The recommended binding primitive is a Midnight
   Passport alias (where Passport is the user-facing identity and wallet
   layer) or, where Passport is out of scope, a per-credential spending
   key held by the recipient. However, any persistent holder binding is
   itself a correlation handle across verifiers and contexts, regardless
   of how it is cryptographically realized: non-transferable and
   non-correlating are in tension at the system level, not the primitive
   level. Solutions should explicitly address which correlations the
   binding does and does not introduce — for example, same-issuer
   cross-credential correlation, cross-issuer correlation via a shared
   binding key, on-chain linkability of presentation proofs across
   sessions, and verifier-side policy that may itself de-anonymise the
   holder. The recommended mitigation is to compose with the proposed
   Domain Separation for Midnight Hash Constructions MPS, which provides
   a registry of canonical domain tags that this MPS recommends a
   solution use for any binding construction: distinct credentials issued
   to the same holder should hash under distinct domain tags, making
   cross-credential correlation structurally impossible even if the
   underlying holder primitive is shared. A credential that is provably
   bound but correlatable across the issuer's full credential population
   is closer to a public SBT than to a Midnight-native primitive.

   **This question is coupled to OQ#6:** the binding-format choice
   constrains the disclosure-proof design space. A plaintext on-chain
   owner field commits the binding to a value any verifier can read,
   which structurally limits what the disclosure proof can guarantee
   regardless of its own design — see OQ#6 for the corresponding
   constraint on the proof format.

3. **What is the revocation latency model?** When an issuer revokes a
   credential, verifiers must observe the revocation within a defined
   window. Should the window be a function of block time, a function of
   proof freshness, or a hybrid (proofs carry an expiry, plus an on-chain
   revocation list)? The trade-off is between revocation latency, verifier
   overhead, and holder-side state.

4. **How does recovery compose with non-transferability?** When the holder
   loses the wallet that controls a soulbound credential, the credential
   must be re-bound to a new wallet. The recovery path must not be
   expressible as a transfer. Is the right model an issuer-signed
   re-binding, a multi-key custody scheme (MPS-0018), or a verifier-side
   policy?

5. **How does this compose with the proposed MAIS (MPS-0015)?** MAIS
   proposes an identity, reputation, and validation framework for AI
   agents. Soulbound attestations for *human* holders and for *agent*
   holders are similar in shape but differ in issuer policy, recovery
   semantics, and the legal weight of revocation. Are these the same
   primitive with different policies, or two distinct primitives that share
   a proof format?

6. **Is the disclosure proof a SNARK, an issuer-signed message, or a
   viewing-key-style capability?** Midnight's privacy architecture allows
   all three. Each has different trade-offs in proof generation cost,
   verifier trust assumptions, and the ability to support selective
   disclosure at multiple granularity levels.

   **This question is coupled to OQ#2:** the disclosure-proof design
   space is constrained by the binding-format choice. The proof must be
   specifiable against a binding that is provable-but-not-plaintext; a
   plaintext on-chain owner field (an answer rejected by MIP-A's
   plaintext-binding prohibition) would make Goal #2 (holder-side
   privacy) structurally unreachable regardless of what this question
   resolves to. MIP-B's design must therefore assume MIP-A has not
   committed to a public owner field, and must not rely on one to
   achieve its privacy guarantees.

7. **What is the right scoping for an MIP vs continued problem-framing in
   the MPS?** If the design space is too broad for a single MIP, the
   recommendation may be two or more MIPs (e.g. one for the credential
   primitive, one for the disclosure proof, one for the revocation
   surface).


## Recommended MIPs

The following MIPs are recommended to specify the non-transferable
specialization on top of the Midnight Passport credential substrate (P9:
C18 attestation tree, C19 issuance, C20 selective-disclosure proof). Each
MIP is described in terms of what it should specify, not how. The MIPs are
interdependent but separable: a single project can adopt one alone and
migrate to the others later.

**MIP-A: Non-Transferability Constraint for Passport Credentials.** This
MIP should specify the additional constraints a Passport credential must
satisfy to be provably non-transferable: the holder-binding interface (with
the constraint that the binding must be provable but not a plaintext
on-chain owner field — a public owner field would commit the binding to a
value any verifier can read, structurally precluding the holder-side
privacy guarantees required by Goal #2 regardless of what MIP-B
specifies), the on-chain marker or predicate by which verifiers distinguish
a non-transferable credential from a transferable Passport credential,
the issuer's role in binding and revocation, and the events verifiers and
indexers observe. It should specify whether the non-transferability
constraint lives in the contract that holds the credential leaf, in a
separate soulbound credential registry keyed by Passport alias, or in a
protocol-level predicate on the attestation tree (C18) itself. It should
explicitly compose with C18 / C19 / C20 and not duplicate the issuance or
disclosure-proof machinery they already specify. It should not specify the
recovery mechanism (covered by MIP-C) or the disclosure proof refinements
specific to non-transferable credentials (covered by MIP-B).

**MIP-B: Selective Disclosure Proof for Non-Transferable Credentials.**
This MIP should specify the proof refinements needed when the credential
being proved is non-transferable, building on the C20 selective-disclosure
proof primitive. It should specify the supported disclosure granularities
(existence only, attribute range, equality), the role of Midnight's three
disclosure tiers (public / auditor / private), the verifier interface, and
the freshness / revocation-check semantics. The non-transferable
specialization introduces a requirement that C20 does not have: the proof
must demonstrate that the credential cannot be re-presented by a different
holder, which is automatic in the transferable case (the holder has it) but
load-bearing here (the proof must not be a transfer vector). It should
support composition with existing proof systems (MIP-0004 view keys,
MIP-0011 shielded ownership proofs, MAIS Disclosure-Tier Registry) where
applicable.

**MIP-C: Recovery and Rotation for Non-Transferable Credentials.** This
MIP should specify how a holder recovers a non-transferable Passport
credential after losing the wallet that controls it, without breaking the
non-transferability property. It should specify the issuer-defined recovery
policy interface, the multi-key custody composition (cross-reference
MPS-0018), the revocation interaction, and the upper bound on the recovery
window. It should address the Sybil-resistance implication: a recovery path
that is too easy effectively becomes a transfer path. The Passport program's
own recovery model (sharded Google / Apple / social recovery, in scope of
the broader Passport design but outside P9) provides the wallet-side
recovery; this MIP specifies the *credential*-side re-binding that must
not be expressible as a transfer.

These three MIPs are interdependent but separable. A single project could
adopt MIP-A alone (using Passport's off-chain disclosure proof and an
issuer-controlled recovery policy) and migrate to MIP-B and MIP-C later.
This staged adoption is a goal, not a constraint; the MPS recommends the
full set but does not require it for partial adoption.


## References

- MIP-0004: Fungible Token Standard with UTXO Conversion Extensions.
  https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0004-fungible-token-standard-with-utxo.md

- MIP-0011: Native Shielded Token Standard.
  https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0011-native-shielded-token.md

- MPS-0015: Agent Identity Gap on Midnight (companion to proposed MAIS).
  https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md

- MIP-xxx: Midnight Agent Identity Standard (MAIS), issue #110 in
  midnightntwrk/midnight-improvement-proposals.
  https://github.com/midnightntwrk/midnight-improvement-proposals/issues/110

- MPS-0018: Multi-key Account Custody for Midnight-Native Assets.
  https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md

- Closed proposal MRC-721 / MRC-1155 family for non-fungible tokens on
  Midnight. Issue #25 in midnightntwrk/midnight-improvement-proposals.
  https://github.com/midnightntwrk/midnight-improvement-proposals/issues/25

- ERC-721 (Ethereum) for the historical reference shape of a
  non-fungible-token standard on a transparent chain.
  https://eips.ethereum.org/EIPS/eip-721

- Soulbound Tokens (Weyl, Ohlhaver, Buterin, 2022) for the original
  framing of non-transferable credentials in a blockchain context.
  https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4105763

- Midnight Passport (the user-facing identity and wallet layer for the
  Midnight network; this MPS recommends a Midnight Passport alias as the
  primary holder-binding primitive where Passport is in scope).
  https://github.com/midnightntwrk/passport

- Passport P9 credential substrate components:
  - C18 — Attestation tree (Merkle-anchored on-chain substrate for
    credentials). This MPS specifies the non-transferability constraint
    on top of C18's tree.
  - C19 — Credential issuance (off-chain issuer flow). This MPS does not
    duplicate C19; it composes with it for issuance semantics.
  - C20 — Selective-disclosure proof. MIP-B builds on C20 with the
    non-transferability refinements.
  https://github.com/midnightntwrk/passport/tree/main/docs/plans/components

- MPS-xxxx: Domain Separation for Midnight Hash Constructions
  (proposed in the Midnight Passport repository; provides the canonical
  domain-tag registry this MPS recommends for binding construction).
  https://github.com/midnightntwrk/passport/blob/main/docs/mps-mip/mps/mps-domain-separation.md


## Acknowledgements

The author thanks the Midnight community for the prior work that motivates
this proposal — particularly the MPS-0018 authors (Hector Bulgarini, Nicolas
Di Prima) for the custody framing, the MPS-0015 / MAIS authors for the
adjacent identity work, the OpenZeppelin authors of MIP-0011 (Iskander
Andrews, Andrew Fleming) for the shielded-coin model that makes the design
space non-trivial, and Karmel Elshinnawi for the Midnight Passport
workshop (June 2026) and the Domain Separation MPS that anchors the
binding-construction recommendation. Any errors or omissions are the author's.

Tooling disclosure: this document was drafted with AI assistance (Hermes,
running on the MiniMax-M3 model) for structure and prose, then reviewed and
edited by the named author prior to submission. Per the repository's
authorship policy, the named author is accountable for the content.


## Copyright

This MPS is licensed under CC-BY-4.0.
