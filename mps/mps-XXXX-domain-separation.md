---
MPS: <Number> # assigned by editors
Title: Domain Separation for Midnight Hash Constructions
Authors: Hector Bulgarini (hbulgarini), Nicolas Di Prima (NicolasDP)
Status: Proposed
Category: Libraries and Tooling | Standards
Created: 23-Jun-2026
Requires: none
Replaces: none

---

## Abstract

Midnight hashes many distinct objects (coin commitments, nullifiers, public
keys, addresses, intent hashes, Merkle nodes, and key derivations) with two
shared primitives: `persistentHash` (SHA-256) and `transientHash` (Poseidon).
Neither primitive takes a domain argument, so separation between use sites is
achieved entirely by callers hand-prepending a tag to the preimage. This
discipline is load-bearing (for a contract-owned coin the commitment and the
nullifier hash an identical preimage and are kept distinct only by their tag),
yet it is uncoordinated. An inventory of the shipped ledger and the Compact
standard library found roughly 28 such sites carrying about 25 distinct tags,
applied without a shared convention: two prefix schemes coexist in the code
(`midnight:` and `mdn:`) and a third (`ni`) in the canonical wallet
specification; only a handful of tags are even named constants; their form
varies (version suffixes, fixed-width padding, or neither); at least two sites
carry no tag at all; and not one tag is declared in any specification or MIP.
The same conceptual objects carry different separators across the
implementation, the specification, and the specification's own diagram. There is
no canonical convention, no authoritative registry, and no enforcement, so a
third-party Compact author has nothing to conform to and no way to avoid
colliding with a ledger or standard-library tag. The shipping implementation is
internally consistent and these constructions work today; the gap is the absence
of a shared standard, not a known defect. This MPS frames that absence so that
future MIPs can specify a convention, a source of truth, and the means to keep
the implementation and the specifications aligned to it.

## Vision

Every hashed object on Midnight draws its domain separator from a single,
published source of truth under one naming convention. A contract author
reserves or looks up a domain tag the way they look up a token type, without
reading ledger source, and is structurally unable to collide with an existing
one. The ledger, the Compact standard library, the wallet, the specifications,
and their diagrams all reference the same canonical tags, and any divergence
between them is a mechanically detectable error rather than a latent surprise
discovered during an audit. Domain separation stops being tribal knowledge held
in scattered string literals and becomes a first-class, reviewable part of the
protocol surface.

## Problem

**Separation is load-bearing, yet the primitive is raw.** `persistent_hash`
(SHA-256) and `transient_hash` (Poseidon) take no domain argument; separation
exists only because each caller prepends a tag, whether into the SHA-256
preimage, as a Poseidon opening, or as a struct field. This is not optional
hygiene. For a contract-owned coin, the commitment and the nullifier hash an
*identical* preimage (same coin, same `data_type`, same contract address) and
are kept distinct only by their domain tag (`mdn:cc` versus `mdn:cn`;
`midnight:zswap-cc[v1]` versus `…-cn[v1]`). A user coin additionally binds spend
authority, so the tag is defence-in-depth there; a contract has no secret, so
the tag is the *sole* separator: without it a coin's commitment would equal its
nullifier. Yet nothing in the primitive, the toolchain, or any document tells an
author that this discipline exists, or how to apply it.

**No canonical convention.** Across shipped code, two prefix schemes coexist:
`midnight:` (zswap, intents, keys, kernel) and `mdn:` (dust, the Merkle leaf).
Dust mixes both. The canonical wallet specification uses a third scheme (`ni`).
Tag form is unsystematic: some carry a `[v1]` version suffix, some a trailing
colon, some are NUL-padded to a fixed 32-byte width, most are plain. Of roughly
25 distinct tags, only four are hoisted even to a named constant; the rest are
inline string literals.

**No registry or source of truth.** No artefact enumerates the domain tags in
use. There is no list a reviewer can consult to confirm completeness, no place a
new contract can check to avoid a collision, and no authority a tag is reserved
against. The inventory that backs this MPS had to be produced by scanning source
by hand.

**The specification, the code, and the diagram disagree.** Because there is
nothing to check against, the same object carries different separators across
artefacts. The coin public key is derived under `midnight:zswap-pk[v1]` in the
ledger, under `ni-pk[v1]`/`ni` in the wallet specification and its diagram
source, and under `mdn:pk` in that diagram's rendered image. The coin secret,
encryption, and Dust key derivations use `midnight:`-prefixed tags in code but
`ni` in the specification. These divergences went uncaught precisely because no
registry exists to reconcile them. Within the shipping ledger the separators are
internally consistent and the constructions work; the hazard is *between*
artefacts (the specification a wallet implementor follows does not match the
code), and whether any such pair denotes the same on-chain object (a latent
mismatch) or merely distinct ones is itself unanswered (see the open questions
below).

**Untagged and ambiguous sites.** At least two hash use sites carry no domain
tag where one would be expected: most clearly the user address, a bare
`persistent_hash` of a verifying key. Several commitments separate by a numeric
or random opening rather than a domain tag, which a reviewer cannot distinguish
from "tag forgotten" without reading the construction.

## Use Cases

- **Third-party contract author** commits to user state with `persistentHash`
  but has no convention to follow and no registry to consult, so they invent an
  ad-hoc prefix, or forget one. If it collides with a ledger or standard-library
  tag, two unrelated objects hash into the same space, and nothing warns them,
  at author time or ever.
- **Cross-implementation agreement:** a wallet (TypeScript), the ledger (Rust),
  and a Compact contract must compute the same commitment to recognise an
  object, yet each must reverse-engineer the correct tag from another's source.
  The coin-public-key divergence is this failure already realised between the
  specification and the implementation.
- **Security review:** an auditor cannot enumerate the domain-separation
  surface, confirm every use site is tagged, or verify that no two objects share
  a tag, without manually scanning the codebase, which is brittle, unrepeatable,
  and exactly what this MPS had to do.
- **Specification maintenance:** a maintainer updating the wallet specification
  or its diagram has no canonical reference to validate separators against, so
  the text, the diagram source, and the rendered image drift apart and ship
  inconsistent.
- **Evolving a tag:** a construction is revised and its separator should change
  (some tags already carry a `[v1]` suffix), but with no versioning convention
  and no registry to record it there is no safe, observable way to do so,
  particularly if separators are frozen at network deployment.

## Goals

1. **One convention** for forming a domain separator (its scheme, length
   discipline, and versioning) that every hash use site follows.
2. **One source of truth:** a single authoritative record of all domain tags in
   use, citable and verifiable, against which the implementation and the
   specifications are checked.
3. **Completeness.** Every `persistentHash` / `transientHash` use site either
   carries a separator drawn from that source of truth, or is explicitly and
   visibly justified as not needing one.
4. **Collision-freedom.** No two distinct objects can share a separator, and an
   author can obtain one guaranteed not to collide with an existing tag.
5. **Cross-implementation agreement.** The ledger, the Compact standard library,
   the wallet, the specifications, and third-party contracts that compute the
   same object use the same separator for it.
6. **Discoverability.** A contract author can discover or reserve a separator
   without reading ledger or standard-library source.
7. **Privacy-preserving lookup.** Whatever interface exposes the source of truth
   does not itself become an oracle for participation patterns, for example by
   admitting enumeration of identifiers per domain, or scoped existence probes,
   beyond what participants strictly need.

## Expected Outcomes

Domain collisions become structurally impossible rather than merely unlikely,
and the divergences between the specification, the code, and the diagrams are
resolved and kept aligned by construction. Third-party Compact authors gain a
single integration target (look up or reserve a tag, conform to one convention)
instead of reverse-engineering separators from source. Security reviewers gain
an enumerable, auditable surface they can check for completeness and
collision-freedom mechanically. Maintainers can evolve separators safely under a
versioning discipline. The domain-separation surface that today exists only as
scattered string literals becomes a reviewable, adoptable part of the Midnight
protocol that wallets, contracts, and the ledger share.

## Open Questions

- **Same object, or two families?** Are the Compact standard library's `mdn:cc`
  / `mdn:cn` (coin commitment / nullifier) and the ledger's
  `midnight:zswap-cc[v1]` / `midnight:zswap-cn[v1]` the same on-chain object (in
  which case the differing separators are a latent mismatch), or distinct
  commitment families that the naming simply fails to distinguish?
- **Which scheme is canonical?** `midnight:`, `mdn:`, or `ni`? Reconciling onto
  one is either a documentation fix or a consensus-affecting change, depending on
  whether separators are frozen at deployment.
- **Are separators frozen at deployment?** Like the address separators in
  MIP-0003, a domain tag that is an input to an on-chain hash may be frozen at
  network deployment. If so, reconciling the divergences is a migration or hard
  fork, not a refactor, and the convention must be settled before the relevant
  constructions are finalised.
- **Where does the source of truth live?** A specification document, an on-chain
  artefact, the Compact standard library, or several kept in sync? This shapes
  how it is enforced and queried.
- **Convention by discipline, or enforced?** Is conformance left to authors, or
  checked at compile time / build time by the Compact toolchain? What is
  feasible, and who owns the tooling?
- **Scope.** Does the convention cover only `persistentHash` / `transientHash`,
  or also the opening-based commitments and the key-derivation sampling that
  today carry their own separators?
- **Versioning semantics.** How does a tag version (e.g. `[v1]`) interact with
  circuit upgrades and any freeze-at-deployment constraint?
- **Registry as oracle.** What read interface exposes the source of truth
  without leaking participation patterns?

## Recommended MIPs

- **Midnight Domain-Separation Convention and Registry.** Specify the canonical
  form of a domain separator (scheme, length, versioning) and an authoritative
  registry of the tags in use: the source of truth that the ledger, the Compact
  standard library, the wallet, and the specifications are checked against. The
  registry can be seeded directly from the inventory in the References. Must
  resolve the canonical-scheme and freeze-at-deployment choices. This is the
  keystone the others hang from.
- **Domain-Separation Conformance and Tooling** *(optional / follow-on).*
  Specify how use sites are checked against the registry: a compile-time or
  build-time check that every `persistentHash` / `transientHash` site carries a
  registered separator, and/or a standard-library helper that applies one.
- **Separator Reconciliation and Migration** *(conditional).* If the
  specification-versus-code divergences describe identical on-chain objects and
  separators are frozen at deployment, specify the reconciliation path and its
  compatibility implications.

## References

- **Domain-separation inventory:** a complete, reproducible scan of the hash use
  sites and domain tags across the shipped ledger and the Compact standard
  library, produced by Midnight Passport (ARC), 2026/06/02. The scan covers
  `midnightntwrk/midnight-ledger` (commit `dfb450d`; crates `base-crypto`,
  `transient-crypto`, `coin-structure`, `zswap`, `ledger`, `onchain-state`) and
  `midnightntwrk/compactc` (`compiler/standard-library.compact`, commit
  `0e76dafa`).
- **Hash primitives:** `base-crypto/src/hash.rs` (`persistent_hash`, SHA-256);
  `transient-crypto/src/hash.rs` (`transient_hash`, Poseidon); the snark-upgrade
  proposal that introduced the persistent/transient split.
- **Divergence sources:** `coin-structure/src/coin.rs` (`midnight:zswap-pk[v1]`
  / `-cc[v1]` / `-cn[v1]`); the WalletEngine specification and
  `zswap-keys.puml` / `zswap-keys.svg` (`ni-pk[v1]` / `ni` / `mdn:pk`).
- **Related MPSs:** MPS-0008 (Keccak-256 hashing) and MPS-0011 (native
  cryptographic primitives), which add hash and signature primitives to Compact
  but do not address how they are domain-separated; MPS-0012 (human-readable
  aliasing), which similarly identifies an absent network-level standard.
- **MIP-0001:** Midnight Improvement Proposal Process.
- **MIP-0003:** ECDSA support; precedent for separators (here, address
  separators) that are frozen at network deployment.
- **Prior art:** BIP-340 tagged hashes; RFC 9380 hash-to-curve domain separation
  tags; NIST SP 800-185 cSHAKE customization strings; Zcash / ZIP domain
  separation. Each establishes per-use-site domain separation as standard
  cryptographic practice.

## Acknowledgements

The IOG ARC department and the Midnight Foundation ledger and cryptography
reviewers.

## Copyright

This MPS is licensed under CC-BY-4.0.
