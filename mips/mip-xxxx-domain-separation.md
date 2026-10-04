---
MIP: "0021"
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

This MIP specifies one convention for a domain separator: a `midnight:` prefix, colon-delimited segments naming the construction, a `[vN]` version suffix, and a 64-byte maximum length. It requires that a hash needing per-contract uniqueness bind the contract address as a separate field of the preimage, following the pattern the ledger already uses to derive token colors, so two contracts implementing the same standard produce different hashes from the same separator string. It proposes `midnight:` as the standard prefix, because that is what the ledger and wallet already use across their subsystems, and it treats domain separators as frozen at network deployment.

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
3. **Version.** The separator ends with a version tag `[vN]`, where `N` is a positive integer starting at `v1`, bounded only by the length maximum below. The version identifies the preimage format the separator commits to, so a change to the hashed structure increments the version and the two formats stay distinct.
4. **Length.** The complete separator string is at most 64 UTF-8 bytes, and MUST NOT end in a zero byte. The 64-byte maximum is a standardization choice, not a hard technical limit (Compact permits far longer byte strings), chosen so every real tag fits with room to spare while bounding circuit cost and preventing abuse (see Rationale). The trailing-zero rule follows Compact's field-aligned-binary validity requirement, under which a value ending in a zero byte is invalid for a `bytes<n>` encoding.
5. **Uniqueness of construction.** A separator uniquely identifies the construction being hashed. Two different constructions do not share a separator string.

### Per-contract uniqueness

A domain separator names a construction, not an instance. The same separator string is expected to appear in every contract that implements a given standard, and that is safe, because per-contract uniqueness comes from a separate field of the preimage rather than from the separator string.

This follows the pattern the ledger already uses to derive token colors. `tokenType` commits to the pair `(domainSep, contractAddress)` under a fixed inner separator (`midnight:derive_token`), so the separator is a shared label and the contract address, hashed in alongside it, is what makes two contracts' results differ. A hash that must be unique per contract instance MUST bind the contract address (`kernel.self()`) as its own field in the preimage, the same way. A hash meant to produce the same value across instances does not. The address is never encoded into the separator string: it is a 32-byte value known only at execution time, and the separator is a compile-time label.

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

The `midnight:` prefix is longer than `mdn:`, and a domain separator is a hashed input, so it costs a few bytes per use. The Rationale below measures that cost and finds it negligible.

### Why the `[vN]` version suffix

The ledger already versions its tags this way (`midnight:zswap-cc[v1]`), so `[vN]` matches existing practice. It gives a construction a safe way to change its hashed format: increment the version, and the two versions are distinct objects.

### Why 64 bytes, and what length actually costs

Compact imposes no meaningful cap on a byte string (the language limit is 16,777,216 bytes), so any length rule is a standardization choice rather than a technical necessity. The 64-byte maximum was chosen against measured circuit cost and the real tag inventory.

The circuit cost of a domain separator was measured by compiling `persistentHash` circuits over tags of several widths (Compact 0.31, the reported circuit rows):

- 21-byte tag: 2293 rows
- 32-byte tag: 4168 rows
- 64-byte tag: 4200 rows
- 128-byte tag: 6128 rows

All four compile to the same circuit-size parameter (`k=13`). Two facts follow. First, the meaningful cost step is at 31 bytes: a tag of 31 bytes or fewer fits a single field element, while anything larger occupies two, which is the jump from 2293 to ~4200 rows. Between 32 and 64 the cost is almost flat (4168 versus 4200, under one percent), because both occupy two field elements. Second, cost scales with the number of hashes a circuit performs, not with tag width: a circuit chaining eight hashes measured 31370 rows at 32-byte tags and 31626 at 64-byte tags, the same one-percent gap multiplied through, with both landing at the same `k`.

So 64 costs essentially the same as 32 while fitting every tag in the shipped inventory (the longest non-composite tag is 21 bytes) and the longer composite tags a contract standard may build (for example a per-circuit authorization tag that appends a circuit name). A lower cap such as 32 would exclude those composites for no measured saving; a higher cap buys headroom no real tag needs and lets circuit cost climb (128 bytes already costs about 50 percent more than 64). 64 is the smallest round bound that clears every real tag while holding cost flat.

These measurements are on the `persistentHash` (SHA-256) path, which is where long composite tags occur; the `transientHash` path in the shipped code uses only short tags (for example the 19-byte `midnight:field_hash`), so the cap does not bind there.

### Why the contract address is a preimage field, not part of the separator

Per-contract uniqueness is bound into a separate preimage field so the separator string stays a shared, copyable label. Two authors implementing the same standard use the same separator and still produce different hashes, because their contract addresses differ. This matches how `tokenType` composes `(domainSep, contractAddress)`. Encoding a per-contract value into the separator string is not an option: the address is a 32-byte execution-time value, and the separator is a compile-time label.

### Alternatives considered

- **Length or structural prefixes.** MPS-0027 raised, and the ledger team rejected, length-prefixing and self-describing structural prefixes, on the grounds that Midnight's serialization is prefixed with an identifier that names the format (ledger commit `2bce7b0`, addressing an audit suggestion). This MIP follows that decision: a separator names the format, with no length or structure prefix.
- **A central registry of every tag.** Rejected: the per-contract binding prevents contract-to-contract collisions with no registry, the shared ledger and standard-library tags already live in their source, and a hand-maintained list would be a standing maintenance burden that adds nothing conformance depends on.
- **Adding a domain argument to the hash primitives.** Rejected: the primitives are fixed protocol functions, and adding a parameter is a ledger crypto change outside the scope of a naming convention.
- **A 32-byte cap.** Rejected: it would exclude the longer composite tags a contract standard legitimately builds, and the measurements show no cost saving over 64 (a 32-byte and a 64-byte tag cost within one percent of each other and share the same circuit-size parameter).
- **No length cap.** Rejected: Compact permits multi-kilobyte byte strings, which would bloat circuits and invite abuse, and an unbounded separator serves no real tag.
- **Standardizing on `mdn:` for its shorter length.** Rejected: the byte saving is negligible against the measurements and is outweighed by consistency with the prefix the ledger and wallet already use.

## Path to Active

### Acceptance Criteria

- The convention is ratified through the MIP process.
- The prefix decision (`midnight:`), the 64-byte maximum, and the freeze-at-deployment position are accepted, or amended, by the working group.
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

Three further points:

- **A separator is public and adds no secrecy.** Its inputs are knowable: the separator string is published in the standard, and the contract address, where bound, is on-chain. A hash whose preimage holds only public values can be reproduced by anyone, which is correct for a public identifier such as a token color but wrong for anything meant to hide a value or authenticate a caller. Those constructions derive that property from a secret in the preimage, or from a `persistentCommit` opening (a random value), not from the separator. Where the set of secret inputs is small, `persistentCommit`'s opening is what prevents an observer who knows the other inputs from guessing and checking.
- **Versioning.** Because a version tag distinguishes preimage formats, an implementation MUST treat `midnight:x[v1]` and `midnight:x[v2]` as unrelated separators and never accept one where the other is expected. A construction that migrates versions retires the old separator rather than reinterpreting it in place.
- **Contract address as the binding field.** The per-contract binding relies on `kernel.self()` returning the real deployed address. A constructor runs before the contract has a deployed address, so a hash that binds the address MUST be computed in a circuit that runs after deployment, not in the constructor, where `kernel.self()` is a placeholder.

## Implementation

The convention is a specification the ledger, the standard library, the wallet, and third-party contract standards follow when forming domain separators. It introduces no new artefact and no new primitive: it standardizes the tag that existing primitives already consume, and the per-contract binding uses `kernel.self()`, which contracts already have. Adoption is coordination with the ledger and `compactc` maintainers, who own the constructions and could enforce the convention in the toolchain, and reference from downstream standards as they are written.

## Testing

- **Convention parser.** A check that a separator string matches the convention: `midnight:` prefix, segment form, `[vN]` suffix, 64-byte maximum, and no trailing zero byte. The known non-conforming tags are flagged as migration candidates.
- **Length bound.** A compile test confirming a 64-byte separator compiles in a `persistentHash` circuit and costs no more than a small margin above a 32-byte separator, pinning the length choice to toolchain behaviour.
- **Per-contract uniqueness.** A test deploying two instances of the same contract standard and confirming that a hash which binds the contract address produces different outputs under the same separator string.
- **Round-trip agreement.** For an object computed in more than one implementation (for example a coin commitment in the ledger and in a Compact contract), a test that both use the same separator and produce the same hash.

## References

- [MPS-0027: Domain Separation for Midnight Hash Constructions](/mps/mps-0027-domain-separation.md), the problem statement this MIP answers, including the tag inventory.
- [MIP-0001: Midnight Improvement Proposal Process](/mips/mip-0001-mip-process.md).
- [MIP-0003: ECDSA support](/mips/mip-0003-ecdsa-support.md), precedent for separators frozen at network deployment.
- Ledger source (`midnightntwrk/midnight-ledger`): `serialize/src/serializable.rs` (the `midnight:` serialization tag); `coin-structure/src/coin.rs` (`midnight:zswap-cc[v1]`, `midnight:zswap-cn[v1]`, and `tokenType` committing `(domainSep, contractAddress)` under `midnight:derive_token`); commit `2bce7b0`, "[PM-20171] Address audit Suggestion 4," which moved coin-structure separators to the versioned prefix form.
- Wallet source (`midnightntwrk/midnight-wallet`): `packages/spec-reference/src/key-derivation-reference.ts` (`midnight:esk`, `midnight:csk`, `midnight:dsk`, `midnight:zswap-pk[v1]`).
- Compact documentation: the `pad` construct, the `persistentHash` hashing pattern, the field-aligned-binary field representation (the 31-byte field element and the trailing-zero validity rule), and the implementation-specific byte-vector limit; the security guide's domain-separation guidance for commitments and nullifiers.
- Prior art: BIP-340 tagged hashes; RFC 9380 hash-to-curve domain separation tags; NIST SP 800-185 cSHAKE customization strings.

## Acknowledgements

This MIP builds on the inventory and framing in MPS-0027 by Hector Bulgarini and Nicolas Di Prima, and on the ledger and cryptography reviewers whose audit remediation established the versioned-prefix separator form.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
