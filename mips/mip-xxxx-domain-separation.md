---
MIP: X
Title: Domain-Separation Convention
Authors:
  - Jay Albert (@JAlbertCode)
Status: Proposed
Category: Standards
Created: 2026-09-15
Requires: none
Replaces: none
MPS: MPS-0027
License: Apache-2.0
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

Midnight separates its hash use sites by hand. `persistentHash` (SHA-256) and `persistentCommit` take no domain argument, so a caller keeps one hashed object distinct from another only by prepending a tag to the preimage. [MPS-0027](/mps/mps-0027-domain-separation.md) documents that this discipline is applied without coordination: roughly 28 use sites carry about 25 tags under three prefix schemes (`midnight:`, `mdn:`, and a spec-only `ni`), with inconsistent form and no shared definition, so a contract author has nothing to conform to.

This MIP specifies one convention for a domain separator: a `midnight:` prefix, colon-delimited segments naming the construction, a `[vN]` version suffix, and a length that fits Compact's hashing constraint. It requires that a hash needing per-contract uniqueness bind the contract address as a separate field of the preimage, following the pattern the ledger already uses to derive token colors, so two contracts implementing the same standard produce different hashes from the same separator string. It proposes `midnight:` as the standard prefix, because that is what the ledger and wallet already use across their subsystems, and it treats domain separators as frozen at network deployment.

## Motivation

Midnight's hash primitives take no domain argument. Separation between what they hash exists only because each caller prepends a tag. For the commitment and nullifier of a single secret this tag is the only thing keeping the two hashes distinct, so without it a coin's commitment would equal its nullifier. The primitives are fixed protocol functions and will not gain a domain parameter, so the separation has to be standardized at the tag the caller prepends. Nothing in the toolchain or any published document tells an author this discipline exists or how to apply it.

MPS-0027 inventoried the shipped ledger and the standard library and found the discipline applied inconsistently. Two prefix schemes appear in code, `midnight:` and `mdn:`, and a third, `ni`, appears in the wallet specification. Tag form varies: some carry a `[v1]` suffix, some a trailing colon, most are plain. Few tags are named constants; the rest are inline literals. No shared definition enumerates the tags in use, so a reviewer cannot confirm that every site is tagged, a new contract cannot check its separator against what exists, and the specification and the code already disagree on the separator for the coin public key.

## Specification

### The convention

A conforming domain separator is a byte string of the form:

```
midnight:<segment>[:<segment>...][vN]
```

1. **Prefix.** The separator begins with `midnight:`, the prefix the ledger and wallet already use across their subsystems (see Rationale).
2. **Segments.** After the prefix come one or more colon-delimited segments, ordered from the owning component to the specific construction. The first segment names the owner: a ledger subsystem (`zswap`, `dust`, `intent`), or, for a contract, the contract standard (`native-unshielded`). Later segments name the construction and role (`cc` for a coin commitment, `cn` for a nullifier, `minter` for a mint-authorization commitment). Segments use ASCII lowercase letters, digits, and hyphens. Examples: `midnight:zswap-cc[v1]`, `midnight:native-unshielded:minter[v1]`.
3. **Version.** The separator ends with a version tag `[vN]`, where `N` is a positive integer starting at `v1`. The version identifies the preimage format the separator commits to, so a change to the hashed structure increments the version and the two formats stay distinct. `N` has no upper bound; a multi-digit version such as `[v100]` is valid and simply spends more of the length budget below.
4. **Length.** The complete separator string is at most 32 UTF-8 bytes. A Compact circuit places it in one `Bytes<32>` cell of the `persistentHash<Vector<k, Bytes<32>>>` pattern using `pad(32, s)`, and a string longer than 32 bytes does not fit that cell and fails to compile. A separator over 32 bytes therefore cannot be used from a circuit.
5. **Uniqueness of construction.** A separator uniquely identifies the construction being hashed. Two different constructions do not share a separator string.

### Per-contract uniqueness

A domain separator names a construction, not an instance. The same separator string is expected to appear in every contract that implements a given standard, and that is safe, because per-contract uniqueness comes from a separate field of the preimage rather than from the separator string.

This follows the pattern the ledger already uses to derive token colors. `tokenType` commits to the pair `(domainSep, contractAddress)` under a fixed inner separator (`midnight:derive_token`), so the separator is a shared label and the contract address, hashed in alongside it, is what makes two contracts' results differ. A hash that must be unique per contract instance MUST bind the contract address (`kernel.self()`) as its own field in the preimage, the same way. A hash meant to produce the same value across instances does not. The address is never encoded into the separator string: it is a 32-byte value known only at execution time, and the separator is a compile-time label already at its 32-byte ceiling.

### Conformance

A construction conforms when:

1. Every `persistentHash` and `persistentCommit` use site that needs domain separation uses a separator matching the convention. A site that needs none is documented as such, with the reason.
2. Every separator it uses matches the convention above.
3. A hash that must be unique per contract instance binds the contract address as a separate preimage field.

## Rationale

### Why `midnight:` is the standard prefix

MPS-0027 leaves the prefix open among `midnight:`, `mdn:`, and `ni`. Inspection of the shipped ledger and wallet settles it for `midnight:`.

`midnight:` is the prefix the core subsystems already use. In the ledger, `serialize/src/serializable.rs` defines the serialization tag constant as `midnight:`, and `coin-structure/src/coin.rs` hashes the Zswap coin commitment and nullifier under `midnight:zswap-cc[v1]` and `midnight:zswap-cn[v1]`. In the wallet, `key-derivation-reference.ts` derives every key under a `midnight:` separator: `midnight:esk`, `midnight:csk`, `midnight:zswap-pk[v1]`, and `midnight:dsk`.

`mdn:` appears only in Dust, and only for its coin commitment and nullifier and the Merkle leaf; Dust's own key derivation uses `midnight:dsk`. It is a localized inconsistency inside a subsystem that otherwise uses `midnight:`. Choosing `midnight:` reconciles those tags toward the prefix everything else already uses.

`midnight:` is longer than `mdn:` (nine bytes against four), and a domain separator is a hashed input, so the longer prefix costs a few bytes in constructions that do not pad. For the Compact path the separator is padded into a fixed 32-byte cell regardless, so within that path the length difference does not exist. The convention keeps segments terse so the readable prefix and the 32-byte ceiling coexist.

### Why the `[vN]` version suffix

The ledger already versions its tags this way (`midnight:zswap-cc[v1]`), so `[vN]` matches existing practice. It gives a construction a safe way to change its hashed format: increment the version, and the two versions are distinct objects.

### Why 32 bytes is a ceiling, not a fixed width

Compact's `persistentHash` consumes a `Vector` of `Bytes<32>` cells, so a separator used from a circuit occupies one 32-byte cell, filled with `pad(32, s)`. A shorter string is padded up to fill the cell, so authors may use any length at or under 32 bytes. The ledger's Rust constructions use exact-length byte arrays with no padding, for example a 12-byte `midnight:esk`, so variable length under the ceiling is already the norm; the fixed cell applies to the Compact path.

### Alternatives considered

- **Length or structural prefixes.** MPS-0027 raised, and the ledger team rejected, length-prefixing and self-describing structural prefixes, on the grounds that Midnight's serialization is prefixed with an identifier that names the format (ledger commit `2bce7b0`, addressing an audit suggestion). This MIP follows that decision: a separator names the format, with no length or structure prefix.
- **A central registry of every tag.** Rejected: the per-contract binding prevents contract-to-contract collisions with no registry, the shared ledger and standard-library tags already live in their source, and a hand-maintained list would be a standing maintenance burden that adds nothing conformance depends on.
- **Adding a domain argument to the hash primitives.** Rejected: the primitives are fixed protocol functions, and adding a parameter is a ledger crypto change outside the scope of a naming convention.
- **Standardizing on `mdn:` for its shorter length.** Rejected: the byte saving applies only in unpadded paths and is outweighed by consistency with the prefix the ledger and wallet already use.

## Path to Active

### Acceptance Criteria

- The convention is ratified through the MIP process.
- The prefix decision (`midnight:`) and the freeze-at-deployment position are accepted, or amended, by the working group.
- At least one downstream standard (for example the token standards) references the convention rather than an ad-hoc literal, and binds the contract address where per-instance uniqueness is required.

### Implementation Plan

1. Ratify the convention through the MIP process.
2. Reference the convention from downstream standards as they are written or revised, and apply the per-contract binding rule in each.
3. Bring existing separators into conformance where doing so is not consensus-affecting, and record the ones that are as candidates for the conditional migration MIP MPS-0027 anticipates.

## Backwards Compatibility Assessment

A domain separator that is an input to an on-chain hash cannot be changed without changing the hash, so reconciling an existing non-conforming tag is a consensus-affecting change. The ledger team's audit remediation that moved coin-structure separators to the versioned prefix form described that work as a far-reaching breaking change with knock-on effects for the wallet and Compact (commit `2bce7b0`). This MIP therefore treats domain separators as frozen at network deployment: the convention governs new constructions, and existing separators that already match (the `midnight:...[vN]` ledger tags) conform as they are.

Whether the non-conforming tags (`mdn:` in the standard library, the `ni` separators in the wallet specification) can be reconciled, and whether doing so is a documentation fix or a hard fork, depends on a question this MIP does not resolve: whether a given `mdn:` tag and its `midnight:` counterpart denote the same on-chain object or two distinct ones. That reconciliation is deferred to the conditional migration MIP MPS-0027 names. This MIP is prescriptive for new constructions and does not alter any shipped on-chain hash.

## Security Considerations

Domain separation prevents hash collision. Two distinct objects that hash under the same separator can be substituted for one another, and for a commitment and nullifier derived from the same secret the separator is the only thing keeping the two hashes distinct. The convention makes the separator explicit and versioned, and the per-contract binding keeps two contracts that use the same separator string from producing the same hash.

Two further points:

- **A separator is public and adds no secrecy.** Its inputs are knowable: the separator string is published in the standard, and the contract address, where bound, is on-chain. A hash whose preimage holds only public values can be reproduced by anyone, which is correct for a public identifier such as a token color but wrong for anything meant to hide a value or authenticate a caller. Those constructions derive that property from a secret in the preimage, or from a `persistentCommit` opening (a random value), not from the separator. Where the set of secret inputs is small, `persistentCommit`'s opening is what prevents an observer who knows the other inputs from guessing and checking.
- **Versioning.** Because a version tag distinguishes preimage formats, an implementation MUST treat `midnight:x[v1]` and `midnight:x[v2]` as unrelated separators and never accept one where the other is expected. A construction that migrates versions retires the old separator rather than reinterpreting it in place.
- **Contract address as the binding field.** The per-contract binding relies on `kernel.self()` returning the real deployed address. A constructor runs before the contract has a deployed address, so a hash that binds the address MUST be computed in a circuit that runs after deployment, not in the constructor, where `kernel.self()` is a placeholder.

## Implementation

The convention is a specification the ledger, the standard library, the wallet, and third-party contract standards follow when forming domain separators. It introduces no new artefact and no new primitive: it standardizes the tag that existing primitives already consume, and the per-contract binding uses `kernel.self()`, which contracts already have. Adoption is coordination with the ledger and `compactc` maintainers, who own the constructions and could enforce the convention in the toolchain, and reference from downstream standards as they are written.

## Testing

- **Convention parser.** A check that a separator string matches the convention: `midnight:` prefix, segment form, `[vN]` suffix, and 32-byte ceiling. The known non-conforming tags are flagged as migration candidates.
- **Compact ceiling.** A compile test confirming a 32-byte separator compiles in the `persistentHash<Vector<_, Bytes<32>>>` pattern and a 33-byte separator does not, pinning the length rule to toolchain behaviour.
- **Per-contract uniqueness.** A test deploying two instances of the same contract standard and confirming that a hash which binds the contract address produces different outputs under the same separator string.
- **Round-trip agreement.** For an object computed in more than one implementation (for example a coin commitment in the ledger and in a Compact contract), a test that both use the same separator and produce the same hash.

## References

- [MPS-0027: Domain Separation for Midnight Hash Constructions](/mps/mps-0027-domain-separation.md), the problem statement this MIP answers, including the tag inventory.
- [MIP-0001: Midnight Improvement Proposal Process](/mips/mip-0001-mip-process.md).
- [MIP-0003: ECDSA support](/mips/mip-0003-ecdsa-support.md), precedent for separators frozen at network deployment.
- Ledger source (`midnightntwrk/midnight-ledger`): `serialize/src/serializable.rs` (the `midnight:` serialization tag); `coin-structure/src/coin.rs` (`midnight:zswap-cc[v1]`, `midnight:zswap-cn[v1]`, and `tokenType` committing `(domainSep, contractAddress)` under `midnight:derive_token`); commit `2bce7b0`, "[PM-20171] Address audit Suggestion 4," which moved coin-structure separators to the versioned prefix form.
- Wallet source (`midnightntwrk/midnight-wallet`): `packages/spec-reference/src/key-derivation-reference.ts` (`midnight:esk`, `midnight:csk`, `midnight:dsk`, `midnight:zswap-pk[v1]`).
- Compact documentation: the `pad(32, s)` construct and the `persistentHash<Vector<k, Bytes<32>>>` pattern that fixes the 32-byte cell; the security guide's domain-separation guidance for commitments and nullifiers.
- Prior art: BIP-340 tagged hashes; RFC 9380 hash-to-curve domain separation tags; NIST SP 800-185 cSHAKE customization strings.

## Acknowledgements

This MIP builds on the inventory and framing in MPS-0027 by Hector Bulgarini and Nicolas Di Prima, and on the ledger and cryptography reviewers whose audit remediation established the versioned-prefix separator form.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
