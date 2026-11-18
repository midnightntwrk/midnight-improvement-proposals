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

This MIP specifies one convention for a domain separator, matching the form the ledger already uses: a namespace prefix (`midnight:` for the protocol, `mip-NNNN:` for a construction a MIP defines), colon-delimited segments naming the construction, a `[vN]` version suffix, and a recommended maximum length of 64 bytes, placed unpadded at the start of the hashed value. It requires that a hash needing per-contract uniqueness bind the contract address as a separate field of the preimage, following the pattern the ledger already uses to derive token colors, so two contracts implementing the same standard produce different hashes from the same separator string. A hash needing per-operation uniqueness binds the operation. Domain separators are frozen at network deployment, so the convention governs new separators and changes no existing hash.

## Motivation

Midnight's hash primitives take no domain argument. Separation between what they hash exists only because each caller prepends a tag. For the commitment and nullifier of a single secret this tag is the only thing keeping the two hashes distinct, so without it a coin's commitment would equal its nullifier. The primitives are fixed protocol functions and will not gain a domain parameter, so the separation has to be standardized at the tag the caller prepends. Nothing in the toolchain or any published document tells an author this discipline exists or how to apply it.

MPS-0027 inventoried the shipped ledger and the standard library and found the discipline applied inconsistently. Two prefix schemes appear in code, `midnight:` and `mdn:`, and a third, `ni`, appears in the wallet specification. Tag form varies: some carry a `[v1]` suffix, some a trailing colon, most are plain. Few tags are named constants; the rest are inline literals. No shared definition enumerates the tags in use, so a reviewer cannot confirm that every site is tagged, a new contract cannot check its separator against what exists, and the specification and the code already disagree on the separator for the coin public key.

## Specification

### The convention

A conforming domain separator is a byte string of the form:

```
<namespace>:<segment>[:<segment>...][vN]
```

1. **Namespace.** The separator begins with a namespace and a colon. The following namespaces are reserved:
   - `midnight` and `mdn`, for separators the protocol implements in the ledger, the wallet, and the compiler. How the protocol uses them is up to its maintainers (see Rationale).
   - `mip-NNNN`, for a construction defined by MIP number NNNN. A draft uses the literal placeholder `mip-xxxx`, which is replaced throughout the document when the editors assign its number.

   Every other namespace is open to the community. A contract SHOULD use a namespace that names its project, and MUST NOT use `midnight`, `mdn`, or a namespace beginning with `mip-`.
2. **Segments.** After the namespace come one or more colon-delimited segments, ordered from the owning component to the specific construction. The first segment names the owner: a ledger subsystem (`zswap`, `dust`, `intent`), or, for a contract, the contract standard (`native-unshielded`). Later segments name the construction and role (`cc` for a coin commitment, `cn` for a nullifier, `minter` for a mint-authorization commitment). Segments use ASCII lowercase letters, digits, hyphens, and underscores. Examples: `midnight:zswap-cc[v1]`, `mip-0014:native-unshielded:minter[v1]`.
3. **Version.** The separator ends with a version tag `[vN]`, where `N` is a positive integer starting at `v1`, bounded only by the length maximum below. The version identifies the preimage format the separator commits to, so a change to the hashed structure increments the version and the two formats stay distinct. The `]` of the version tag is the only `]` in the separator and is its final byte.
4. **Length.** The separator is ASCII and SHOULD be at most 64 bytes, including the version suffix. 64 bytes is the SHA-256 block size, so a separator within it occupies at most the first block of the preimage, and each further 64 bytes adds a block to every circuit that uses it. This is a cost recommendation, not a technical limit (Compact permits far longer byte strings; see Rationale).
5. **Uniqueness of construction.** A separator uniquely identifies the construction being hashed. Two different constructions do not share a separator string.

### Encoding

The separator's bytes are the first element of the hashed value (for `persistentCommit`, of the committed value), unpadded and without a length prefix, as the ledger does. In Compact the separator is a `Bytes<n>` element whose length is exactly the separator's length, for example `persistentHash<[Bytes<21>, Bytes<32>]>(["midnight:zswap-cc[v1]", x])`. A separator used as a key-derivation `info` string is used as its bytes.

Because the separator contains exactly one `]`, as its final byte, the boundary between the separator and the data that follows is unambiguous. No separator is a prefix of another, so two different separators never produce the same leading bytes, whatever data follows them.

### Per-contract uniqueness

A domain separator names a construction, not an instance. The same separator string is expected to appear in every contract that implements a given standard, and that is safe, because per-contract uniqueness comes from a separate field of the preimage rather than from the separator string.

This follows the pattern the ledger already uses to derive token colors. `tokenType` commits to the pair `(domainSep, contractAddress)` under a fixed inner separator (`midnight:derive_token`), so the separator is a shared label and the contract address, hashed in alongside it, is what makes two contracts' results differ. A hash that must be unique per contract instance MUST bind the contract address (`kernel.self()`) as its own field in the preimage, the same way. A hash meant to produce the same value across instances does not. The address is never encoded into the separator string: it is a 32-byte value known only at execution time, and the separator is a compile-time label.

### Per-operation uniqueness

A separator and an address separate constructions and instances, not operations within one contract. When one construction serves several operations whose results must not be interchangeable, such as signature challenges for different circuits, the operation MUST be bound, either as the final segment of the separator or as a separate field of the preimage.

### Conformance

A construction conforms when:

1. Every `persistentHash` and `persistentCommit` use site that needs domain separation uses a separator matching the convention. A site that needs none is documented as such, with the reason.
2. Every separator it uses matches the convention above.
3. A hash that must be unique per contract instance binds the contract address as a separate preimage field.
4. A hash that must be unique per operation binds the operation.

### Versioning

Until this MIP is Accepted, it is versioned by its revision history. After acceptance, a change to the convention or the binding rules requires a new MIP that lists this one in `Replaces`.

## Rationale

### Why these namespaces

MPS-0027 leaves the prefix open among `midnight:`, `mdn:`, and `ni`. The shipped ledger and wallet use `midnight:` for most separators and `mdn:` for a few. Both belong to the protocol, so this MIP reserves them for it and leaves their use to its maintainers. A construction that a MIP defines uses `mip-NNNN:` instead, as [MIP-0018](/mips/mip-0018-on-chain-token-metadata.md) already does for its event name. Editors assign MIP numbers, so the namespace is unique without a registry; the separator names the specification that defines it; and it cannot collide with data the ledger hashes. Contracts outside MIPs use their own namespace, and per-contract binding keeps them apart from each other.

`midnight:` is the prefix the core subsystems already use. In the ledger, `serialize/src/serializable.rs` defines the serialization tag constant as `midnight:`, and `coin-structure/src/coin.rs` hashes the Zswap coin commitment and nullifier under `midnight:zswap-cc[v1]` and `midnight:zswap-cn[v1]`. In the wallet, `key-derivation-reference.ts` derives every key under a `midnight:` separator: `midnight:esk`, `midnight:csk`, `midnight:zswap-pk[v1]`, and `midnight:dsk`.

`mdn:` appears only in Dust, and only for its coin commitment and nullifier and the Merkle leaf; Dust's own key derivation uses `midnight:dsk`.

### Why the `[vN]` version suffix

The ledger already versions its tags this way (`midnight:zswap-cc[v1]`), so `[vN]` matches existing practice. It gives a construction a safe way to change its hashed format: increment the version, and the two versions are distinct objects. Because the suffix closes the separator, it also marks where the separator ends, so separators of any length can be placed unpadded without ambiguity.

### Why 64 bytes, and what length actually costs

Compact imposes no meaningful cap on a byte string (the language limit is 16,777,216 bytes), and an unpadded separator is unambiguous at any length, so this MIP recommends a maximum rather than requiring one. The 64-byte recommendation follows from measured circuit cost and the real tag inventory.

`persistentHash` is SHA-256, whose circuit cost grows with the number of 64-byte blocks in the whole preimage. A separator therefore adds cost only when it pushes the preimage into another block. Measured with compactc 0.36.1 (`zkir-v3 mock-compile`), for a separator followed by 32 bytes of data:

| Separator | Circuit rows |
|---|---|
| 21 bytes | 2018 |
| 31 bytes | 3870 |
| 32 bytes | 3872 |
| 64 bytes | 3888 |
| 128 bytes | 5768 |

All compile to the same circuit-size parameter (`k=13`). A 32-byte and a 64-byte separator cost within one percent of each other, while 128 bytes adds a block. 64 fits every real tag, including per-operation tags that append an operation name.

Placing the separator unpadded is never more expensive than zero-padding it to a fixed width, and is sometimes cheaper. Measured on existing constructions:

| Construction | Padded to 32 bytes | Unpadded |
|---|---|---|
| MIP-0013 device entry | 6109 rows | 6105 rows |
| Three 32-byte fields | 3978 rows | 3975 rows |
| MIP-0032 namespace (10-byte tag) | 5932 rows | 4072 rows |

### Why the contract address is a preimage field, not part of the separator

Per-contract uniqueness is bound into a separate preimage field so the separator string stays a shared, copyable label. Two authors implementing the same standard use the same separator and still produce different hashes, because their contract addresses differ. This matches how `tokenType` composes `(domainSep, contractAddress)`. Encoding a per-contract value into the separator string is not an option: the address is a 32-byte execution-time value, and the separator is a compile-time label.

### Alternatives considered

- **Length or structural prefixes.** MPS-0027 raised, and the ledger team rejected, length-prefixing and self-describing structural prefixes, on the grounds that Midnight's serialization is prefixed with an identifier that names the format (ledger commit `2bce7b0`, addressing an audit suggestion). This MIP follows that decision: a separator names the format, with no length or structure prefix.
- **A central registry of every tag.** Rejected: the per-contract binding prevents contract-to-contract collisions with no registry, the shared ledger and standard-library tags already live in their source, and a hand-maintained list would be a standing maintenance burden that adds nothing conformance depends on. The `mip-NNNN:` namespace makes each separator a MIP defines unique and traceable to its specification without a list.
- **Adding a domain argument to the hash primitives.** Rejected: the primitives are fixed protocol functions, and adding a parameter is a ledger crypto change outside the scope of a naming convention.
- **A 32-byte cap.** Rejected: it would exclude the longer composite tags a contract standard legitimately builds, and the measurements show no cost saving over 64 (a 32-byte and a 64-byte tag cost within one percent of each other and share the same circuit-size parameter).
- **Zero-padded separators with a `:vN` segment**, the form several draft MIPs use. Rejected: padding needs its own rules to keep a separator's boundary unambiguous (no zero bytes, one width per separator), it differs from the ledger's form, and it is never cheaper than placing the separator unpadded.
- **A shared readable namespace for standards, or a name per standard.** Rejected: the names would have to be reserved and checked across MIPs, and a separator would not identify the specification that defines it.
- **A required maximum length.** Not adopted: no functional limit requires one, and a longer separator costs only the constructions that use it. The recommendation states that cost instead.

## Path to Active

### Acceptance Criteria

- The convention is ratified through the MIP process.
- The prefix decision (`midnight:`), the 64-byte recommended maximum, and the freeze-at-deployment position are accepted, or amended, by the working group.
- At least one downstream standard (for example the token standards) references the convention rather than an ad-hoc literal, and binds the contract address where per-instance uniqueness is required.

### Implementation Plan

1. Ratify the convention through the MIP process.
2. Reference the convention from downstream standards as they are written or revised, and apply the per-contract binding rule in each.
3. Align separators that MIPs define but that are not yet deployed on mainnet before those constructions deploy.

## Backwards Compatibility Assessment

A domain separator that is an input to an on-chain hash cannot be changed without changing the hash, so reconciling an existing non-conforming tag is a consensus-affecting change. The ledger team's audit remediation that moved coin-structure separators to the versioned prefix form described that work as a far-reaching breaking change with knock-on effects for the wallet and Compact (commit `2bce7b0`). This MIP therefore treats domain separators as frozen at network deployment: the convention governs new constructions, and existing separators that already match (the `midnight:...[vN]` ledger tags) conform as they are.

Existing protocol separators, including those in the `mdn:` namespace, stay as deployed. The coin commitment and nullifier that MPS-0027 asked about now use the ledger's `midnight:zswap-cc[v1]` and `midnight:zswap-cn[v1]` in the Compact standard library as well. MIP-0012, MIP-0013, MIP-0022, MIP-0030, MIP-0032 and MIP-0033 define `:vN` separators, most of them zero-padded, for constructions not yet deployed on mainnet; aligning them before deployment changes no deployed hash. Separators already deployed by contracts stay valid. This MIP is prescriptive for new constructions and does not alter any shipped on-chain hash.

## Security Considerations

Domain separation prevents hash collision. Two distinct objects that hash under the same separator can be substituted for one another, and for a commitment and nullifier derived from the same secret the separator is the only thing keeping the two hashes distinct. The convention makes the separator explicit and versioned, and the per-contract binding keeps two contracts that use the same separator string from producing the same hash.

Three further points:

- **A separator is public and adds no secrecy.** Its inputs are knowable: the separator string is published in the standard, and the contract address, where bound, is on-chain. A hash whose preimage holds only public values can be reproduced by anyone, which is correct for a public identifier such as a token color but wrong for anything meant to hide a value or authenticate a caller. Those constructions derive that property from a secret in the preimage, or from a `persistentCommit` opening (a random value), not from the separator. Where the set of secret inputs is small, `persistentCommit`'s opening is what prevents an observer who knows the other inputs from guessing and checking.
- **Versioning.** Because a version tag distinguishes preimage formats, an implementation MUST treat `midnight:x[v1]` and `midnight:x[v2]` as unrelated separators and never accept one where the other is expected. A construction that migrates versions retires the old separator rather than reinterpreting it in place.
- **Contract address as the binding field.** The per-contract binding relies on `kernel.self()` returning the real deployed address. A contract's address is the hash of its deploy data, which includes the constructor's initial state (`ContractDeploy::address` in the ledger; MIP-0013 section 3; MIP-0032 Rationale), so inside a constructor `kernel.self()` does not return the deployed address. A hash that binds the address MUST be computed in a circuit that runs after deployment. A construction that needs instance separation at construction time MAY take a deployer-supplied salt instead, and MUST state that the salt's uniqueness is the deployer's responsibility.

## Implementation

The convention is a specification the ledger, the standard library, the wallet, and third-party contract standards follow when forming domain separators. It introduces no new artefact and no new primitive: it standardizes the tag that existing primitives already consume, and the per-contract binding uses `kernel.self()`, which contracts already have. Adoption is coordination with the ledger and `compactc` maintainers, who own the constructions and could enforce the convention in the toolchain, and reference from downstream standards as they are written.

## Testing

- **Convention parser.** A check that a separator string matches the convention: namespace, segment form, and a single `[vN]` suffix as the final bytes, and that reports separators longer than 64 bytes. Existing separators that predate the convention are reported, not rejected.
- **Length bound.** A compile test confirming a 64-byte separator compiles in a `persistentHash` circuit and costs no more than a small margin above a 32-byte separator, pinning the length choice to toolchain behaviour.
- **Per-contract uniqueness.** A test deploying two instances of the same contract standard and confirming that a hash which binds the contract address produces different outputs under the same separator string.
- **Round-trip agreement.** For an object computed in more than one implementation (for example a coin commitment in the ledger and in a Compact contract), a test that both use the same separator and produce the same hash.

## References

- [MPS-0027: Domain Separation for Midnight Hash Constructions](/mps/mps-0027-domain-separation.md), the problem statement this MIP answers, including the tag inventory.
- [MIP-0001: Midnight Improvement Proposal Process](/mips/mip-0001-mip-process.md).
- [MIP-0003: ECDSA support](/mips/mip-0003-ecdsa-support.md), precedent for separators frozen at network deployment.
- [MIP-0013: Account Authorisation](/mips/mip-0013-account-authorisation.md) and [MIP-0032: Private Credential Registry](/mips/mip-0032-private-credential-registry.md), on the contract address being unavailable in a constructor.
- [MIP-0018: On-Chain Token Metadata Emission](/mips/mip-0018-on-chain-token-metadata.md), precedent for the `mip-NNNN:` namespace.
- Ledger source (`midnightntwrk/midnight-ledger`): `serialize/src/serializable.rs` (the `midnight:` serialization tag); `coin-structure/src/coin.rs` (`midnight:zswap-cc[v1]`, `midnight:zswap-cn[v1]`, and `tokenType` committing `(domainSep, contractAddress)` under `midnight:derive_token`); commit `2bce7b0`, "[PM-20171] Address audit Suggestion 4," which moved coin-structure separators to the versioned prefix form; `ledger/src/structure.rs` (`ContractDeploy::address`).
- Wallet source (`midnightntwrk/midnight-wallet`): `packages/spec-reference/src/key-derivation-reference.ts` (`midnight:esk`, `midnight:csk`, `midnight:dsk`, `midnight:zswap-pk[v1]`).
- Compact documentation: the `persistentHash` hashing pattern and the implementation-specific byte-vector limit; the security guide's domain-separation guidance for commitments and nullifiers.
- Prior art: BIP-340 tagged hashes; RFC 9380 hash-to-curve domain separation tags; NIST SP 800-185 cSHAKE customization strings.

## Acknowledgements

This MIP builds on the inventory and framing in MPS-0027 by Hector Bulgarini and Nicolas Di Prima, on review comments from Nicolas Di Prima and Nick Stanford, and on the ledger and cryptography reviewers whose audit remediation established the versioned-prefix separator form.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
