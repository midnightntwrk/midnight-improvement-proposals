---
MIP: "xxxx"
Title: Private Credential Registry and Soulbound Profile
Authors:
  - Harley Hermanson (@HarleysCodes)
  - Dennis Zarelli (@DpacJones)
Status: Draft
Category: Standards
Created: 2026-06-25
Requires: none
Replaces: none
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

This MIP specifies a protocol for privately held, issuer-attested credentials, together with a **soulbound profile** under which a credential cannot be transferred. It defines, in terms of SHA-256 and explicit byte encodings rather than any implementation language: a hiding commitment suite binding a credential to its holder, its class, and a revocation handle; a revocation model separating membership evidence, which may be anchored to any accepted root, from non-revocation evidence, which MUST be anchored to the current root; a single presentation statement proving membership, non-revocation, and a profile predicate over one opening, returning a challenge-bound nullifier; and the domain-separation requirements that keep credential classes and deployments from colliding.

The commitment is built by the holder and handed to the issuer opaquely, so an issuer cannot impersonate a holder. Issuer authority is proof of knowledge of a secret, not a prover-supplied public key. Non-transferability is a property of that binding rather than of a token that could change custody, which is what distinguishes this from a transfer-disabled NFT.

The mechanism is not specific to soulbound credentials. The same registry serves any privately held, revocable, issuer-attested membership, and soulbound behaviour is one profile's binding predicate. This MIP defines a closed base cryptographic suite together with an extensible profile registry, so further profiles can be registered without re-specifying the registry.

Every base-suite byte derivation defined in §2 through §6 is a concatenation of 32-byte fields under SHA-256, published with conformance vectors, and all of them have been reproduced by an implementation written from this specification alone, using no Compact and no Midnight runtime. The authenticated-root and path derivations of §3 are deliberately outside that suite: this MIP specifies their behaviour and the identity rules by which incompatible implementations are detected, not their internals.

## Motivation

Several recurring use cases need a credential bound to one holder that cannot be traded: KYC and sanctions attestations, education credentials, named-attendee tickets, governance delegation, proof of personhood, DAO membership. Existing token standards are transfer-centric, and native shielded coins move at the protocol layer and cannot be soulbound.

A naive "ERC-721 with transfer disabled" port is inadequate on a privacy chain for three reasons. It stores a plaintext owner, which is a correlation handle. It provides non-transferability only as the absence of a function, which is a claim about an interface rather than about a binding. And it offers no normative commitment, so two implementations cannot verify each other's credentials. This primitive addresses all three.

The same three gaps recur for any privately held, revocable, issuer-attested membership, not only for credentials that are conceptually soulbound. The specification below is therefore written as a registry with a profile extension point, and the soulbound guarantee is stated as a property of one profile.

## Specification

This section is normative. A conforming implementation MUST satisfy §1 through §7 and MUST reproduce the conformance vectors of §10.

The Compact construction under **Implementation** is one conforming implementation and is **non-normative**. Where that construction makes a choice this specification does not require, such as contract topology, ledger data-structure selection, or circuit decomposition, the choice is identified as such. Toolchain versions are recorded with the construction rather than here: a change of compiler does not change this specification.

The key words MUST, MUST NOT, SHOULD, and MAY are to be interpreted as described in RFC 2119.

### 1. Terminology

**Issuer.** The party authorized to add credentials to a registry and to revoke them. Authority is defined in §4.

**Holder.** The party a credential is bound to. The holder constructs the commitment and retains the opening.

**Verifier.** The party that consumes a presentation and decides whether to act on it. Verifier obligations are defined in §7.

**Credential commitment (`cm`).** A hiding commitment to the credential's opening. The only credential artifact that reaches public state.

**Opening.** The values that reconstruct `cm`: profile tag, namespace, class identifier, holder public key, holder secret, revocation handle, and commitment nonce.

**Revocation handle.** A per-credential identifier, private until revocation, which the issuer publishes to revoke that credential. Defined in §2.4.

**Presentation.** A zero-knowledge proof satisfying the statement in §6, producing a nullifier.

**Registry.** One logical credential class: an issuer, a class identifier, a membership set, and a revocation set, sharing one namespace.

**Profile.** A binding predicate a presentation MUST satisfy in addition to membership and non-revocation. Defined in §5.

### 2. Cryptographic suite and object model

#### 2.1 Base cryptographic suite `mipa:cred:v1`

All derivations in this specification operate on **32-byte fields** and are defined in terms of SHA-256 (FIPS 180-4).

**Primitives.**

```
H(a_1, ..., a_n)              = SHA-256( a_1 || ... || a_n )
COMMIT(a_1, ..., a_n ; r)     = SHA-256( r || a_1 || ... || a_n )
```

where every `a_i` and `r` is exactly 32 bytes and `||` is byte concatenation. There is no length prefix, no type tag, and no padding beyond the 32-byte field width. In `COMMIT` the opening `r` is the **first** input.

**Encodings.**

```
TAG(s)   = the ASCII encoding of s, right-zero-padded to 32 bytes
LE32(x)  = the 32-byte little-endian encoding of the non-negative integer x (see §2.4)
```

**Tag canonicalization (normative).** `s` MUST be between 1 and 32 bytes, and every byte MUST be printable ASCII in the range `0x21` to `0x7E` inclusive. In particular `s` MUST NOT contain a NUL byte or a space. Without this restriction the right-zero padding is not injective, since `TAG("a")` and `TAG("a\0")` are the same 32 bytes; the tags defined in this MIP are unaffected, but the extensible registries of §10 make the ambiguity reachable.

**Relationship to the platform.** On Midnight, `H` corresponds to Compact's `persistentHash` and `COMMIT` to `persistentCommit`, both of which the platform documents as SHA-256 based and guaranteed stable across upgrades. The byte framing above is **not currently published by the platform**; it was established empirically against the reference construction and is pinned by the conformance vectors of §10. See **Rationale, platform constraints encountered**.

**The algorithms in this section are the conformance authority.** An implementation conforms by computing exactly what is written above, whether or not it uses Compact. The correspondence to the platform primitives is a **compatibility claim**: it asserts that a Compact implementation using `persistentHash` and `persistentCommit` produces the same values, and it is falsifiable against the vectors of §10. Should the platform publish a framing that differs, this MIP's algorithms remain the conformance authority for the current suite version and the divergence is resolved by a new suite version, not by silently tracking the platform.

**Scope limit.** This suite covers the derivations in §2 through §6. It does **not** cover the internal node hashing of the authenticated structures in §3, which the platform likewise does not publish. §3 therefore specifies those structures behaviourally, and §3.3 standardizes only the **logical layer** of a revocation set. The wire representation of the tree itself remains unspecified by this MIP and is declared by implementation identity under the two-part declaration of §3.3.

#### 2.2 Domain separation

Each credential class MUST be domain-separated. The `issuerNamespace` bound into a commitment MUST differ across classes, across issuers, and across distinct deployments of the same class by the same issuer. An implementation MUST NOT allow a commitment issued under one namespace to satisfy a presentation under another.

```
issuerNamespace = H( TAG("mipa:ns:v1"), initialIssuerPk, schemaId, deploymentSalt )
```

The namespace MUST be derived from the **initial** issuer identity rather than the current one, so that issuer rotation under §4 does not invalidate existing credentials.

`deploymentSalt` is a 32-byte value that MUST be unique per deployment. This specification does not mandate its source. The reference construction takes it as a deployer-supplied parameter, for the reason recorded under **Rationale, platform constraints encountered**: the contract's own address is not available at construction time on the current platform.

`issuerNamespace` and `credentialDomain` (the class identifier) MUST be fixed for the life of a registry. An implementation MUST NOT expose an interface that mutates either.

#### 2.3 Credential commitment

A conforming credential commitment is exactly:

```
cm = COMMIT( profileTag,          // 32 bytes, §5
             issuerNamespace,     // 32 bytes, §2.2
             credentialDomain,    // 32 bytes, class identifier
             holderPk,            // 32 bytes, §5
             holderSecret,        // 32 bytes, holder control secret
             LE32(handle)         // 32 bytes, §2.4
           ; credentialNonce )    // 32 bytes, the opening
```

that is, `SHA-256( credentialNonce || profileTag || issuerNamespace || credentialDomain || holderPk || holderSecret || LE32(handle) )`.

`holderSecret` MUST be generated by the holder from a CSPRNG and MUST NOT be shared with the issuer. `credentialNonce` MUST be generated from a CSPRNG and MUST be fresh per credential; it is the hiding opening, and reuse across credentials destroys hiding.

The holder builds `cm` and transmits only `cm` to the issuer. An issuer therefore cannot reconstruct an opening and cannot present as the holder.

Element order and encodings are normative. Any change is a new suite version under §10.

#### 2.4 Revocation handle representation

`handle` is an integer in the range `[1, 2^248)`. A total order over handles is required by the absence proof of §3, and 248 bits is the maximum comparable integer width available on the current platform. See **Rationale, platform constraints encountered**.

**Encoding.** `LE32(x)` is the 32-byte little-endian encoding of `x`: byte `i` of the output is `floor(x / 256^i) mod 256` for `i` in `0..31`.

**Decoding.** A 32-byte string `b` decodes to `sum(b[i] * 256^i)` for `i` in `0..31`.

**Validity.** A conforming encoding of a handle MUST have `b[31] = 0`, which is exactly the constraint `x < 2^248`. Bytes `b[0]` through `b[30]` are unconstrained; in particular the high bit of `b[30]` is part of the value and MUST NOT be masked. An implementation MUST reject `x = 0`, which is the reserved sentinel of §3.3, and MUST reject any `x >= 2^248`.

**Generation.** Handles MUST be drawn uniformly from `[1, 2^248)`. An implementation SHOULD use rejection sampling over 31 random bytes, rejecting the all-zero draw, rather than reducing a wider value modulo the range.

Little-endian is stated explicitly because it is the opposite of the convention most credential and hashing specifications assume. An implementation that encodes big-endian will produce different commitments and will silently fail to interoperate while appearing internally correct. The vectors in §10 are the authority.

### 3. Revocation and membership state

#### 3.1 Behavioural requirements

A registry maintains a **membership set** of issued commitments and a **revocation set** of revoked handles.

- The membership set MUST be an authenticated structure supporting succinct proofs that a given commitment is a member, against a published root, without revealing which member.
- The revocation set MUST be an authenticated structure supporting **sound proofs of absence** against a published root, without revealing the queried handle.
- **Membership evidence MAY be anchored to any accepted root** (§3.2). Without this, any issuance invalidates every in-flight presentation, because a presentation is prepared against the root observed when it was built.
- **Non-revocation evidence MUST be anchored to the current root** of the revocation set. An implementation MUST NOT accept a historic revocation root. This asymmetry is the load-bearing requirement of this section: accepting a historic revocation root would let a since-revoked holder anchor to a pre-revocation root and present successfully.
- Both statements MUST be bound to the same credential opening, per §6.

This section deliberately does **not** mandate a particular structure. A sparse Merkle tree, an authenticated dictionary, an indexed Merkle tree, or a suitable accumulator may each satisfy the above. A registry declares its choice through the two-part declaration of §3.3. Note that this MIP standardizes only the logical layer of that choice; proof-level interoperability additionally requires agreement on an authenticated-structure component, which this MIP does not specify.

**Revocation is public.** Any structure satisfying the above publishes the fact that some handle was revoked. §9 states what this does and does not reveal.

#### 3.2 Accepted membership roots

A membership root is **accepted** if it is authenticated as having been a root of this registry and has not been retired.

- A registry MUST maintain an authenticated record sufficient for a verifier to determine whether a claimed membership root was a root of that registry. A verifier MUST NOT accept a root on the prover's assertion alone.
- A registry MAY retire roots, for example to close a malformed issuance epoch or after a state migration. Retirement MUST be published, and a verifier MUST reject presentations anchored to a retired root.
- A registry MAY bound root retention rather than retaining every historic root indefinitely. If it does, it MUST publish the retention policy, and a verifier MUST reject roots outside it. Holders whose evidence has aged out re-prepare against a current root.

Unbounded retention is not required by this specification, and an implementation that assumes it is not conforming merely by being permissive.

#### 3.3 Logical revocation-set component `mipa:revset:imt:v1`, and the two-part declaration

This component is normative **for registries that declare it**. It specifies the **logical layer** of a revocation set: the leaf preimage, the ordering discipline, and the absence condition. It is **not a complete proof-format declaration**, and a registry MUST NOT declare it alone; see *Two-part declaration* below.

The revocation set is an indexed Merkle tree whose leaves are `(value, next)` pairs forming a sorted linked list over the revoked handles, with `next = 0` denoting the end of the list. The pair `(0, 0)` is the sentinel planted at construction, which is why `0` is excluded as a handle in §2.4.

```
revLeaf(value, next) = H( TAG("mipa:revleaf:v1"), LE32(value), LE32(next) )
```

Absence of a handle `h` is proved by exhibiting the leaf `(value, next)` with `value < h` and `(next = 0 or h < next)`, together with a path from that leaf to the current root. Neither `h` nor the position of the gap is revealed.

**Scope limit (normative).** This component specifies the leaf preimage, the ordering discipline, and the absence condition. It does **not** specify tree depth or capacity, leaf placement or indexing, the empty-subtree convention, the internal node compression function, sibling ordering within a path, root derivation, or proof encoding. Those are supplied by the underlying authenticated-structure implementation, and on the current platform they are not published; see **Rationale, platform constraints encountered**. Two registries that agree on this component but differ in those details will produce mutually incompatible roots and paths. Cross-implementation verification of proofs therefore requires agreement on a specified authenticated-structure component **in addition to** this one, and this MIP does not define such a component.

**Two-part declaration (normative).** Because the items above determine whether two registries produce compatible roots and paths, declaring a logical component alone is operationally ambiguous: two mutually incompatible registries would present the same identifier, and a verifier's allowlist could not distinguish them. A registry MUST therefore declare **two** identifiers:

1. a **logical revocation-set component**, such as `mipa:revset:imt:v1`; and
2. an **authenticated-structure component**, fixing every item enumerated in the scope limit above.

A verifier MUST evaluate the pair, not either identifier alone, and MUST reject a presentation if it does not accept both (§7.6).

**Identity semantics for the authenticated-structure identifier (normative).** An identifier that does not distinguish differing behaviour provides no detection, so the following are required.

1. **Authenticated, not asserted.** The two-part declaration MUST be authenticated as metadata of the intended registry. A verifier MUST NOT accept either identifier on the prover's assertion alone.
2. **Immutable for the registry.** The declaration MUST be immutable for the registry, or for a state epoch the verifier can identify. A registry MUST NOT silently change it.
3. **Unambiguous.** The identifier MUST unambiguously designate an implementation that fixes **every** item enumerated in the scope limit above.
4. **Behaviour-changing means identifier-changing.** Any change affecting any scope-limit item MUST be published under a **different** identifier. An implementation version string that does not pin all such behaviour is insufficient for this purpose.
5. **Canonical and exactly compared.** The identifier's canonical byte representation and its comparison rule MUST be defined by whoever assigns it, and verifiers MUST compare canonically and exactly. Approximate, prefix, or version-range matching MUST NOT be used.

Two constructions satisfying these are a **digest over the artifacts that supply the structure**, and an **authority-controlled, immutable, globally unique version namespace**. This MIP does not mandate either.

A worked negative example: an identifier of the form `midnight:compact:merkletree<32>:lang-0.23` is **not** sufficient on its own, because a language version does not necessarily pin the compiler build, the runtime library, or the protocol version that together determine tree behaviour. Two mutually incompatible implementations could legitimately present it, which defeats requirement 4.

This MIP does not specify an authenticated-structure component, because the current platform does not publish the details one would have to fix. Until such a component exists, a registry declares this part by **implementation identity** under the rules above. Such an identifier names an implementation, not a specification. Its purpose is to make incompatibility **detectable**, which is the defect this two-part declaration repairs; it does not make independent reimplementation possible, and this MIP does not claim otherwise. Publishing the details listed in the scope limit would allow a properly specified component to take its place.

**No retroactive completion (normative).** A future revision MUST NOT extend `mipa:revset:imt:v1` to assign the tree details it currently leaves open. Implementations may already have made conflicting choices under this identifier, so a completed composition MUST use a new logical-component identifier, a newly specified authenticated-structure identifier, or both.

Other logical revocation components MAY be defined under their own identifiers, subject to the same two-part declaration requirement.

### 4. Issuer authority requirements

Authorization MUST be proof of knowledge of a secret. An implementation MUST NOT authorize on a prover-supplied identity value such as a caller public key obtained from a witness, because such a value is chosen by the prover and is therefore spoofable.

```
issuerPk = H( TAG("mipa:issuer:pk:v1"), issuerSecret )
```

`issuerPk` is recorded in public state. Issuance and revocation MUST require the caller to demonstrate knowledge of an `issuerSecret` deriving to the recorded `issuerPk`. `issuerSecret` MUST be generated from a CSPRNG.

**Rotation MUST be two-step.** The current issuer proposes a successor identity, and the successor accepts by proving knowledge of the corresponding secret. A single-step rotation would allow authority to be moved to an identity for which no secret is known, permanently bricking the registry.

Authority in this version is a single secret. A compromise of that secret compromises issuance and revocation until rotation. Multi-party issuer authority requires in-circuit signature verification and is out of scope for this version.

### 5. Profiles

A **profile** fixes the binding predicate a presentation MUST satisfy in addition to membership and non-revocation. A profile is identified by a 32-byte tag bound into the commitment, so a credential's profile is fixed at issuance and cannot be changed afterwards by the holder or the verifier.

This MIP defines two profiles.

| Profile | Tag | Additional predicate |
|---|---|---|
| Wallet (default) | `TAG("mipa:cred:wallet:v1")` | The presentation MUST prove knowledge of a `holderIdentitySecret` with `holderPk = H( TAG("mipa:holder:pk:v1"), holderIdentitySecret )`. |
| Bearer | `TAG("mipa:cred:bearer:v1")` | None. Possession is exactly knowledge of the opening. |

The soulbound guarantee stated in §8 is a **wallet-profile** guarantee. The bearer profile is transferable by handing over the opening, which is the point of a bearer instrument. Integrators requiring non-transferability MUST issue wallet-profile credentials, and MUST accept only the wallet profile when consuming them.

**Profile hiding (normative).** A presentation MUST NOT reveal which profile the credential carries. Precisely: an adversary given the public inputs of §6, the proof, and the returned nullifier MUST learn nothing about `profileTag` beyond what is already implied by the set of profiles the verifier accepts. How an implementation achieves this is out of scope here; one technique is given under **Implementation**.

Profiles are extensible. A future MIP MAY register a new profile by specifying a new tag and its predicate, without re-specifying §2 through §4, §6, or §7, and **without changing the base suite version** (§10). A verifier MUST reject a presentation whose profile is not one it accepts.

### 6. Presentation statement

A presentation is a zero-knowledge proof of the following statement, over a **single** opening.

**Public inputs.** The registry's `issuerNamespace` and `credentialDomain`, the membership root the proof is anchored to, the current revocation root, and a verifier-supplied 32-byte challenge.

**Private inputs.** The credential opening of §2.3, the holder identity secret where the profile requires it, the membership proof, and the absence proof.

**Proven conjunction.** All three of the following, over the same opening:

1. **Membership.** The commitment reconstructed from the opening, using the registry's own `issuerNamespace` and `credentialDomain` rather than prover-supplied values, is a member of the membership set at the anchored root, without revealing which member.
2. **Non-revocation.** The **same** `handle` from that opening is absent from the revocation set at its current root, without revealing the handle.
3. **Profile predicate.** The predicate of the profile committed in the opening, per §5.

Binding membership and non-revocation to one opening is normative and is what makes revocation enforceable. If the two are proved independently, a holder can prove membership with a real handle and non-revocation with a fabricated one, and revocation becomes unenforceable under composition. An implementation MUST NOT expose membership and non-revocation as separately composable statements.

**Output.** The presentation discloses a **nullifier**:

```
nullifier = H( TAG("mipa:present:v1"), cm, holderSecret, verifierChallenge )
```

The nullifier binds the full credential context through `cm` and stays unlinkable across distinct challenges through the private `holderSecret`. It MUST NOT be derivable from public values alone, which would make it linkable, and it MUST include the verifier challenge, which is what makes it usable for replay detection under §7.

A presentation is **read-only**. It records nothing in public state. One-time-use semantics, where required, are a verifier obligation under §7 or a matter for a future extension that records nullifiers on-chain.

### 7. Verifier requirements

This section is normative. Without it, two implementations can agree on every commitment and still disagree on what accepting a presentation means. Because a presentation records nothing on-chain, **anti-replay rests entirely with the verifier.**

A verifier MUST:

1. **Generate the challenge itself,** as 32 bytes from a CSPRNG, and record it as outstanding together with an expiry and the operation or context it authorizes. A verifier MUST NOT accept a challenge it did not issue, including one supplied by the prover or by another verifier.
2. **Bind the challenge to its context,** so that a challenge issued for one verifier, audience, or operation cannot be used for another. The binding MUST be part of the verifier's own outstanding-challenge record; it is not a property the prover can assert.
3. **Require the exact issued challenge.** The verifier MUST compare the challenge in the presentation against a specific outstanding record, not merely check that it is well formed.
4. **Reject expired challenges.**
5. **Consume atomically on acceptance.** In one atomic transaction, the verifier MUST verify that the challenge exists as an outstanding record, is unexpired, is context-matched, and is unconsumed; verify that the returned nullifier has not previously been accepted; and only then mark the challenge consumed and record the nullifier. If any precondition fails, the verifier MUST reject without modifying either record. Non-atomic consumption permits concurrent submissions of the same proof to both succeed.
6. **Validate the anchoring state.** The membership root MUST be an accepted root of the intended registry per §3.2, and the revocation root MUST be the current one. The verifier MUST reject a presentation unless it accepts **both** identifiers of the registry's two-part revocation declaration, the logical component and the authenticated-structure component (§3.3). Accepting the logical identifier alone is insufficient, because incompatible registries can present the same one. The verifier MUST obtain the declaration as authenticated registry metadata rather than from the prover, and MUST compare both identifiers canonically and exactly, per the identity semantics of §3.3.
7. **Reject presentations whose profile is not accepted** for the decision being made, per §5.

**Correlation note.** The nullifier is deterministic in `(cm, holderSecret, verifierChallenge)`. Under a genuinely fresh challenge it therefore differs across presentations, and logging nullifiers does not by itself accumulate cross-session linkage. It correlates exactly the presentations made under the **same** challenge, which is the intended replay-detection property. A verifier that reuses a challenge across sessions or across holders converts that into broader linkage and MUST NOT do so.

### 8. Non-transferability: direct and indirect paths (normative scope)

**What soulbound binds to here.** On a privacy chain a credential is not a movable object; it is a hiding commitment stored in the issuer's tree, bound to the holder's key and control secret. "Soulbound" therefore means one precise thing in this primitive: the ability to present a credential is bound to knowledge of the holder's secrets, and for the wallet profile that binding is enforced at presentation time by the holder-key predicate of §5. Non-transferability is a property of that binding, not of a token that could change custody.

**Direct non-transferability (guaranteed).** The protocol exposes no interface that re-binds or moves a commitment. Issuance appends a new member, revocation adds a handle to the revoked set, presentation is read-only, and issuer rotation moves issuer authority only. None of them rewrites a holder binding from one `holderPk` to another, and there is no release, claim, or transfer. A conforming implementation MUST NOT add one. This closes the naive "ERC-721 with transfer disabled" gap: non-transferability is the structural absence of any re-binding path, verifiable by inspecting the interface surface, not a runtime flag.

**Indirect paths: what omission alone does not cover.** Omitting a transfer operation stops the credential from moving on-chain. It does not stop control of the credential from moving off-chain, and on this architecture all three indirect vectors reduce to a single root cause: custody of the presenting secrets.

1. **Proxying.** A holder who hands their opening and identity secret to another party lets that party present as them. The wallet profile resists casual proxying, because a bare proof is not replayable without the identity secret, but it cannot stop a holder who chooses to share the secret. This is intrinsic to secret-knowledge authentication and is not reachable by anything the protocol omits.

2. **Wrapping.** A wrapper contract cannot take custody of a credential the way it can an ERC-721, because there is no transfer to call and no object to escrow: the commitment lives in the registry, bound to `holderPk`. The only thing a wrapper can custody is the secret material, after which it presents on the holder's behalf. Wrapping therefore collapses into proxying and carries no additional on-chain surface.

3. **Contract-held issuance.** An issuer may issue to a `holderPk` controlled by a contract rather than a person. The credential is then only as non-transferable as control of that contract, so the contract's own governance becomes an effective transfer surface. This is an issuance-policy decision outside the primitive: the primitive cannot tell whether a `holderPk` is a human wallet or a contract-controlled key, and MUST NOT be relied on to enforce human holding by itself.

**Bearer profile is outside the soulbound guarantee by design.** The bearer profile omits the identity predicate, so possession is exactly knowledge of the opening. A bearer credential is transferable by handing over that opening; that is the point of a bearer instrument. The soulbound guarantee in this section is a wallet-profile claim. Integrators who need non-transferability MUST issue wallet-profile credentials.

**Path to closing the indirect gap (out of scope for v1).** Because the residual gap is secret custody, closing it requires binding presentation to something a holder cannot hand over as bytes:

- non-exportable or hardware-held holder keys, so the identity secret cannot be copied;
- an interactive challenge that proves live control of the holder key at presentation rather than mere knowledge of a secret (Schnorr-in-circuit);
- a cryptographically enforced caller identity, so presentation can require the transaction sender to be the holder key rather than any prover who knows the secret (see "Caller Identity Access in Compact Circuits", #213); and
- issuance policy that binds `holderPk` to an attested device.

None of these are in v1. Holder recovery and rotation, the legitimate counterpart to "control moved", are deferred to the companion recovery MIP; a holder with a lost or rotated key recovers through re-issuance, not through a transfer path.

### 9. Anonymity set

Revocation is a **public** event: revoking publishes the handle, and anyone can walk the public revocation set. This reveals **what** is revoked, not **who**. The credential commitment is hiding and never reveals which holder holds which handle. Under the component of §3.3 the bracket values are public regardless, since they are derivable by walking the structure, so revealing the predecessor in a proof reveals nothing further. A holder's non-revocation proof discloses only a public root, never the handle or its gap position.

The **number** of revocations is public, as is the number of issued credentials, since both are derivable from the public structures regardless of whether an implementation also maintains an explicit counter. Neither count is attributable to a holder.

The anonymity set is therefore all handles not already linked to a holder through out-of-band issuer records, issuance or presentation timing, or external information. It is **not** a guarantee against metadata correlation.

Tooling note: the bracket arguments of a revocation are public transaction inputs, because a revocation is a public event. Wallet and issuer interfaces should treat them as operational metadata and avoid surfacing them in a way that invites mapping handles to holders.

### 10. Versioning and conformance

**Base cryptographic suite `mipa:cred:v1` is closed.** It comprises exactly the primitives and encodings of §2.1 and the following five domain tags:

| Tag | Binds | Defined in |
|---|---|---|
| `mipa:ns:v1` | issuer namespace derivation | §2.2 |
| `mipa:issuer:pk:v1` | issuer identity derivation | §4 |
| `mipa:holder:pk:v1` | holder key derivation | §5 |
| `mipa:revleaf:v1` | revocation-set leaf encoding | §3.3 |
| `mipa:present:v1` | presentation nullifier derivation | §6 |

Any change to a primitive, an encoding, a preimage layout, an element order, or one of these five tags is a **new base suite version**.

**Profile tags are an extensible registry, separate from the base suite.** This MIP registers `mipa:cred:wallet:v1` and `mipa:cred:bearer:v1`. A future MIP MAY register additional profile tags under §5. Registering a profile tag does **not** change the base suite version, because it changes the *value* of the `profileTag` field rather than the construction that consumes it. Verifiers negotiate profiles by allowlist, per §5 and §7.7.

**Revocation declarations are two-part, and the two parts occupy separate identifier namespaces.** This MIP registers one **logical** revocation-set component, `mipa:revset:imt:v1` (§3.3). It registers **no** authenticated-structure component, because the platform does not publish the details one would have to fix; registries identify that part by implementation identity, under the rules of §3.3, until a specified component exists. A future MIP specifying an authenticated-structure component is the intended route to proof-level interoperability, and is the reason §3.3 forbids retroactively completing `mipa:revset:imt:v1` in place.

**All registered identifiers MUST satisfy the tag canonicalization rule of §2.1.** Because tags are right-zero-padded into a fixed 32-byte field, an unrestricted tag alphabet would let two distinct registry entries encode identically.

**Tag prefix is provisional.** The `mipa:` prefix derives from the working name of this proposal and is a placeholder. On assignment of a MIP number, every tag MUST be re-pinned to the assigned number, that is `mip-NNNN:cred:v1` and so on, and the conformance vectors regenerated. Implementers MUST NOT ship against the provisional prefix.

**Proposed general convention.** Domain tags in a Midnight standard SHOULD be prefixed with the number of the MIP that assigns them. Deriving tags from an authority that is unique by construction makes collisions across independently developed standards structurally impossible, and makes the owning specification of any observed tag self-evident. This convention is offered for adoption beyond this MIP and would fit naturally in a domain-tag registry.

**Conformance vectors.** A conforming implementation MUST reproduce the vectors under **Implementation**, covering both profile tags, the handle encoding of §2.4, the namespace derivation of §2.2, the issuer identity derivation of §4, the holder key derivation of §5, the revocation leaf encoding of §3.3 including the sentinel, the commitment of §2.3 for both profiles, and the nullifier derivation of §6. Negative vectors MUST include a big-endian handle encoding and a zero handle.

## Rationale

### Design decisions

- **Secret-derived issuer authority plus a holder-built commitment** close the two central risks together: no prover-supplied value is trusted for authorization, and the issuer never holds the holder's opening. Either alone is insufficient.
- **An opaque issuance interface plus a normative suite** keep issuance privacy-preserving while still guaranteeing cross-implementation verification. The cost, that the registry cannot check well-formedness of what it stores, is analysed under Security Considerations.
- **Accepted membership roots against a current-only revocation root** is the asymmetry that prevents stale-root revocation bypass while keeping in-flight presentations valid. Making both historic breaks revocation; making both current breaks usability.
- **One bound presentation statement rather than two composable ones** is what makes revocation enforceable. This was the single most consequential correction made during review of the reference construction: independent membership and non-revocation proofs are individually sound and jointly useless.
- **A hiding commitment rather than a hash** is required because several preimage elements are low-entropy or issuer-known.
- **Behavioural requirements for state, plus a standardized logical layer,** rather than mandating one tree shape. Earlier drafts wrote the reference implementation's indexed Merkle tree into the normative layer, which would have excluded sparse Merkle trees, authenticated dictionaries, and accumulators that satisfy the same statement. §3.1 states what must hold; §3.3 standardizes the leaf preimage, ordering, and absence condition, and requires the tree layer to be declared alongside it so that incompatibility is detectable rather than silent. Proof-level interoperability needs a specified authenticated-structure component, which does not yet exist on this platform.
- **A closed base suite with an extensible profile registry** rather than a single flat tag list. An earlier draft closed the entire tag set, which contradicted profile extensibility outright: no new profile could ever be registered without a suite version bump.

### Platform constraints encountered

Four properties of the current platform shaped this specification. They are recorded here because they generalize beyond this MIP, and because three of them are candidates for platform-level resolution.

**No published byte framing for the persistent hash and commitment primitives.** The platform documents `persistentHash` and `persistentCommit` as SHA-256 based and stable across upgrades, but does not publish how a value is serialized into the SHA-256 input. Without that, no standard built on these primitives can be implemented outside Compact, which makes every such standard un-interoperable by construction. The framing in §2.1 was therefore established empirically against the reference construction and is stated normatively here, pinned by conformance vectors. **Publishing a normative serialization for these primitives would benefit every Midnight standard, not only this one**, and would let specifications cite the platform instead of re-deriving it.

**Authenticated-structure internals are unpublished, and belong to a different primitive family.** The gap here is worse than for the persistent primitives, where the algorithm is documented and only the framing was missing. In the platform's Merkle structures, roots and path siblings are **field elements rather than byte strings**, the empty-subtree digest is the zero field element, and the node compression function is a circuit-optimised field-native hash that the platform does not document at all. It is therefore not reconstructible from a standard library the way §2.1 is, and this MIP does not attempt to freeze it by reverse engineering.

The practical consequence is a clean split: the **leaf** layer of a revocation set can be specified here, in SHA-256 terms, and is interoperable; the **tree** layer cannot be, and §3.3 says so explicitly. Publishing the node compression function, the field it operates over, the empty-subtree convention, the sibling ordering within a path, and the proof encoding would allow a specified authenticated-structure component to be defined and paired with §3.3's logical component, replacing the implementation-naming identifier that registries must use today. It would do the same for every other Midnight standard that needs to prove membership or absence across implementations.

**Contract self-address is unavailable during construction.** A constructor circuit is proven *before* the contract address exists, because the address is derived from the deploy transaction that embeds the constructor's initial state. A self-address query therefore resolves to a dummy value inside a constructor. This was confirmed empirically: two deployments with identical issuer and class identifier, deriving their namespace from the self-address, produced byte-identical namespaces. Consequently **any contract seeking per-instance domain separation sealed at deployment cannot currently derive it from its own address.** This specification's workaround is a deployer-supplied `deploymentSalt` that MUST be unique, which is load-bearing and unenforceable on-chain. A construction-time binding to the eventual address, or a protocol-supplied per-deployment nonce, would let this requirement be dropped, and would remove the same footgun from every other contract needing instance separation.

**Comparable integer width is capped at 248 bits.** The absence proof requires a total order over handles, and 248 bits is the maximum comparable integer width available. This fixes the handle domain at `[1, 2^248)`. 248 bits of randomness is cryptographically ample, so the ceiling constrains the design without weakening it, but it is a platform ceiling rather than a design choice. The little-endian byte order of §2.4 is likewise a consequence of the platform's field serialization rather than a choice made here.

## Path to Active

Reference implementation merged, deployed to public testnet, exercised by a disclosure-proof flow (companion MIP) and by a wallet, conformance vectors published, and reviewed for the authorization and correlation properties.

### Acceptance Criteria

- An independent implementation, written from this specification and not derived from the reference construction, reproduces every published conformance vector.
- A holder is issued a credential and proves possession under §6 without revealing the membership member, the handle, or any secret.
- Issuer authorization rejects a forged secret; no non-issuer interface mutates registry state; a revocation naming an inconsistent predecessor is rejected rather than corrupting the revocation set.
- A revoked holder cannot produce a passing presentation, including when anchoring the membership half to a pre-revocation root.
- Two registries sharing an issuer and class identifier but differing in `deploymentSalt` derive distinct namespaces, and a commitment from one does not satisfy a presentation against the other.
- A verifier implementation demonstrates the §7 obligations, including rejection of a challenge it did not issue, rejection after expiry, and atomic consumption under concurrent submission.
- A verifier rejects a registry whose authenticated-structure identifier it does not accept, and rejects a two-part declaration taken from the prover rather than from authenticated registry metadata.
- The reference construction demonstrates that its handle encoding equals `LE32` over the whole domain `[1, 2^248)`, not merely at sampled points.
- Security review covering issuer impersonation, commitment forgery, revocation bypass, replay, and correlation.

### Implementation Plan

Stage 1, this MIP: issuer authority, the credential registry, the revocation model, the base cryptographic suite, profiles, and the bound presentation statement. Stage 2, companion MIP: higher-level disclosure-proof composition and selective attribute disclosure. Stage 3, companion MIP: holder credential recovery and rotation.

## Backwards Compatibility Assessment

New primitive. No existing standard changes and no fork is required. Credentials under this MIP are non-fungible, and under the wallet profile they are intentionally non-transferable, so they are deliberately not interoperable with transfer surfaces. Bearer-profile credentials are transferable by opening, per §5 and §8. No migration.

## Security Considerations

- **Issuer authorization** is secret-derived; a prover-supplied identity value MUST NOT be used, per §4. Single-secret authority means a leak compromises issuance and revocation until rotation, and the two-step rotation of §4 prevents rotation to an unprovable identity. Multi-party authority requires in-circuit signature verification and is future work.
- **`deploymentSalt` is load-bearing.** Per-registry namespace separation depends entirely on it being fresh and unique. Reuse across registries sharing an issuer and class identifier collapses their namespaces and re-enables cross-registry commitment confusion. It cannot be replaced by the contract's own address, for the reason under Rationale.
- **Anti-replay rests entirely with the verifier,** per §7, because a presentation records nothing on-chain. The failure modes are specific: accepting a challenge the verifier did not issue lets a captured proof be replayed at a second verifier; non-atomic consumption lets concurrent submissions both succeed; and omitting expiry widens the window in which a captured proof remains valid.
- **Opaque issuance.** The issuance interface cannot validate the well-formedness or uniqueness of what it stores, so issuers MUST issue only commitments vetted out of band, with unique handles valid under §2.4. Malformed or duplicate members are unpresentable but consume registry capacity. Because presentation enforces the profile predicate and knowledge of the opening, the residual on-chain risk is limited to issuer-policy failure.
- **Holder control and delegation.** Specified normatively in §8: proxying, wrapping, and contract-held issuance all reduce to custody of the presenting secrets, and the residual gap, a willing holder sharing secrets, is documented there along with its path to closing.
- **Correlation.** Commitments carry no deterministic public holder identifier, preventing direct cryptographic linkage. Metadata, timing, and issuer-record correlation are not prevented, per §9. Nullifiers correlate presentations made under the same challenge, per §7.
- **Encoding conformance.** An implementation that encodes the handle big-endian produces commitments that are internally consistent and mutually incompatible with conforming implementations. This is a silent interoperability failure rather than a rejection, which is why §10 requires vectors.
- **Commitment framing.** Because `COMMIT` places the opening first and applies no length framing, all inputs are fixed at 32 bytes; an implementation that admits variable-length inputs into this construction reintroduces the concatenation ambiguity that fixed widths avoid.
- **Disclosure.** Only commitments, public roots, and the nullifier cross the boundary. The issuer secret, holder secret, holder identity secret, commitment nonce, and revocation handle never do.

## Implementation

### Independent implementability

The **base-suite byte derivations** defined in §2 through §6, that is every use of `H` and `COMMIT` written out in this document, were reproduced by an implementation written from this specification alone, using SHA-256 from a standard library, with no Compact toolchain and no Midnight runtime. All published commitment, key-derivation, revocation-leaf, and nullifier vectors matched. This is the evidence for the interoperability claim, and its scope is exact: the specification, not the reference construction, is sufficient to implement the **credential cryptographic layer**. It is not sufficient to implement the authenticated structures of §3, whose internals this MIP does not specify.

### Reference construction (Compact)

The following is **non-normative**: one conforming realization. It is compile-verified on compactc 0.31.0 and language 0.23.0 (`pragma language_version >= 0.23`). The toolchain pin applies to this construction only, not to the specification.

**Registry topology (a construction choice).** This construction seals the class identifier and namespace at deployment, so **one contract instance serves exactly one credential class**: KYC, proof of personhood, and DAO membership are three separate deployments. Integrators MUST NOT assume one instance serves multiple classes, and a wallet or verifier flow is scoped to a single deployed instance. Sealing the class at deploy binds a credential to its schema without per-operation overhead. Other constructions MAY satisfy §2.2 differently, for instance by binding a class identifier into the namespace derivation of a multi-class instance.

**Primitive mapping.** `H` is `persistentHash<Vector<n, Bytes<32>>>` and `COMMIT` is `persistentCommit<Vector<n, Bytes<32>>>`. `TAG(s)` is `pad(32, s)`. `LE32(x)` is `(x as Field) as Bytes<32>`, because a direct `Uint -> Bytes` cast is not available; the normative artifact is the encoding of §2.4, not this cast sequence, and the construction is required to demonstrate the two agree across the domain.

**State.** `issuedCredentials: HistoricMerkleTree<32, Bytes<32>>` provides §3.2's accepted-root behaviour; `revokedSet: MerkleTree<32, Bytes<32>>` is current-root-only and carries the `mipa:revset:imt:v1` logical component of §3.3.

**Two-part declaration in practice.** This construction declares the logical component `mipa:revset:imt:v1` together with an authenticated-structure identifier of the digest form permitted by §3.3, `midnight:merkletree32:sha256:<digest>`. A bare version string such as `lang-0.23` is deliberately **not** used, because it fails requirement 4 of §3.3: a language version does not pin the compiler build or runtime library, so two incompatible builds could present it.

**What the digest must cover.** Requirement 3 makes completeness a conformance obligation, not a matter of taste: the identifier must designate an implementation fixing *every* scope-limit item. Hashing a selected subset of inputs is therefore weaker evidence than hashing the artifacts that actually determine the behaviour. A conforming digest SHOULD cover the final compiled artifact together with the platform build and every external dependency that contributes to the structure's behaviour, rather than a chosen list of version strings. A digest over an incomplete artifact set is nonconforming under requirement 3 even though it is a digest.

The identifier is published as registry metadata and is not prover-supplied, per requirement 1. Note the honest limit: a digest is a **compatibility discriminator**. It proves that the covered artifacts match, not that the covered set is complete. That is a reason to prefer a digest over a version string and to make the covered set as complete as possible, not a reason to treat either as a specification.

Roots and path siblings in this construction are field elements rather than byte strings, and the empty-subtree digest is the zero field element, which is why the tree layer is not expressible in the SHA-256 terms of §2.1.

**Interface.**

```compact
export circuit issue(cm: Bytes<32>): [] { ... }              // issuer-only
export circuit revoke(handle: Uint<248>,
                      lowValue: Uint<248>,
                      lowNext: Uint<248>): [] { ... }        // issuer-only; splice index DERIVED
                                                             // from the proven path, not supplied
export circuit proposeIssuer(newIssuerPk: IssuerPublicKey): [] { ... }
export circuit acceptIssuer(): [] { ... }
export circuit provePresentation(verifierChallenge: Bytes<32>): Bytes<32> { ... }
export pure circuit presentationNullifier(cm: Bytes<32>,
                                          holderSecret: Bytes<32>,
                                          verifierChallenge: Bytes<32>): Bytes<32> { ... }
```

**Profile hiding technique (§5).** The construction expresses the profile predicate as an asserted implication rather than a branch, so the profile is not revealed by control flow. A consequence is that the identity-secret witness is evaluated unconditionally, so a bearer presentation must still supply a value for it; any value works and is ignored. This is one way to meet §5's requirement, not the only one.

`provePresentation` calls `presentationNullifier` rather than duplicating the derivation, so a published nullifier vector cannot drift from what a presentation emits; the test suite asserts that equality directly. Exposing the derivation adds nothing to the provable surface, since a pure circuit requires no proving key.

### Reference implementation

Compact contract, TypeScript witnesses, and conformance vectors, Apache-2.0: https://github.com/DpacJones/midnight-nft, directory `contracts/credential-zk/`.

### Conformance vectors

All values hex, 32 bytes. Pinned to the **provisional** `mipa:` tag prefix; regenerate when tags are re-pinned to the assigned MIP number, per §10.

| Vector | Value |
|---|---|
| `TAG("mipa:cred:wallet:v1")` | `6d6970613a637265643a77616c6c65743a763100000000000000000000000000` |
| `TAG("mipa:cred:bearer:v1")` | `6d6970613a637265643a6265617265723a763100000000000000000000000000` |
| `issuerPk`, `issuerSecret = 0x01 * 32` | `ed3afc3b9e884bf4065fa07da037457344228734e96f63d48d81214a7e10d9df` |
| `issuerNamespace` [0] | `82f9b391e9bcce3bcc7867cf6c8834cbf1d4524ee7d7be79fb61fbd548fe42db` |
| `holderPk`, `identitySecret = 0x02 * 32` | `054210f0406c3e998ae51205c5d401540e6591f8ed95984ec7420b1230391604` |
| `revLeaf(0, 0)` sentinel | `e804ca81fbefef1416b03a36098a74e23481540254292a5180fbf4e064f922ea` |
| `revLeaf(1, 0)` | `0f2b1cee67e474361ba437df6f2e1f5ed5b67ce2e4e46cb953b37514d2c47368` |
| `revLeaf(5, 9)` | `1eee30c8afeaca7c646b0c15270eaf020c209a51b6ae1096acdaf1c540f24763` |
| `cm`, wallet profile [1] | `432d543fbf904b2cf94d79f5e64d7161cccf3088c04328f4c56a52debf0f6cea` |
| `cm`, bearer profile [1] | `82b7a477e580e0ec15a99169be538a12a79be94965e29ccf39948d058dc1bd90` |
| `nullifier`, wallet profile [2] | `2d0482744548dccb8e1613b8c7753e83326de41ca5d55104477d5f8faecdf741` |
| `nullifier`, bearer profile [2] | `d03c30340ec4fbf41ac3ec3ed78c2f1037528328f20ced00bd9ae6295966fbd2` |

[0] `initialIssuerPk` as in the row above (`issuerSecret = 0x01 * 32`), `schemaId = 0x77 * 32`, `deploymentSalt = 0x55 * 32`. This vector was taken from the sealed ledger state of an actual deployment and independently reproduced from §2.2, so it exercises the constructor path, not only the derivation.

[1] `issuerNamespace = 0xaa * 32`, `credentialDomain = 0xbb * 32`, `holderPk = 0xcc * 32`, `holderSecret = 0xdd * 32`, `handle = 0x0102030405`, `credentialNonce = 0xee * 32`, profile tag as listed.

[2] The `cm` of the corresponding row, `holderSecret = 0xdd * 32`, `verifierChallenge = 0x0f * 32`.

Handle encoding vectors, per §2.4:

| handle | `LE32(handle)` |
|---|---|
| `1` | `0100000000000000000000000000000000000000000000000000000000000000` |
| `256` | `0001000000000000000000000000000000000000000000000000000000000000` |
| `0x0102030405` | `0504030201000000000000000000000000000000000000000000000000000000` |
| `2^247` | `0000000000000000000000000000000000000000000000000000000000008000` |
| `2^248 - 1` | `ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff00` |

## Testing

Unit: issuer-only enforcement, forged-secret rejection, two-step rotation, handle validity per §2.4, rejection of a revocation with an inconsistent predecessor. Invariant: no non-issuer interface mutates registry state; a revoked holder cannot present; two credentials for the same holder are distinct members. Privacy: no secret reaches public state; a presentation reveals only public roots and the nullifier; profile hiding per §5. Separation: distinct `deploymentSalt` values yield distinct namespaces, and cross-registry commitments do not verify. Verifier: rejection of an unissued challenge, rejection after expiry, and atomic consumption under concurrent submission. Conformance: positive and negative vectors including a big-endian handle encoding and a zero handle, an assertion that the separately exposed nullifier derivation reproduces what a real presentation emits, and a domain-wide check that the construction's handle cast equals `LE32`.

## References

- Soulbound and Non-Transferable Shielded Attestations (problem statement), midnight-improvement-proposals PR #211.
- Caller Identity Access in Compact Circuits, midnight-improvement-proposals PR #213.
- MIP-0004, MIP-0011, token standards this composes alongside.
- FIPS 180-4, Secure Hash Standard.
- Reference implementation: https://github.com/DpacJones/midnight-nft (Apache-2.0).

## Acknowledgements

Problem framing and the binding-as-correlation-handle and anonymity-set analysis: the PR #211 author group. The §8 non-transferability analysis, in particular the reduction of proxying, wrapping, and contract-held custody to a single root cause, and the wallet-profile scoping of the soulbound claim: CjDabrow.

## Copyright Waiver

Licensed under Apache-2.0; contribution subject to the repository Contributor License Agreement. All authorship is human, with no AI authors or co-authors, per repository policy.
