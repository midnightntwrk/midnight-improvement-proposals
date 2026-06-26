---
MIP: "xxxx"
Title: Soulbound Credential Primitive
Authors: Dennis Zarelli @DpacJones, Harley Hermanson @HarleysCodes
Status: Draft
Category: Standards
Created: 2026-06-25
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

This MIP specifies a minimal on-chain primitive for a **soulbound credential**: an attestation bound to a holder with no non-issuer interface to move it. An issuer deploys one contract instance **per credential class**; the contract stores **holder-built hiding commitments** in a Merkle tree and revoked handles in an Indexed Merkle Tree, gated by a **secret-derived issuer authority**. The holder builds the commitment off-chain from their key, a control secret, a per-credential nonce, and a revocation handle, and hands the issuer only the opaque commitment — so the issuer cannot impersonate the holder. A single presentation circuit proves, over **one** opening, that the credential is issued and not revoked (and, for the wallet profile, that the holder controls the committed key), returning a challenge-bound nullifier. The commitment suite is **normative** so independent implementations interoperate. The presentation/disclosure proof composition and recovery are scoped to companion MIPs.

## Motivation

Several recurring use cases need a credential bound to one holder that cannot be traded: KYC/sanctions attestations, education credentials, named-attendee tickets, governance delegation, proof-of-personhood, DAO membership. Existing token standards are transfer-centric; native shielded coins move at the protocol layer and cannot be soulbound. A naïve "ERC-721 with transfer disabled" port is inadequate on a privacy chain: it stores a plaintext owner (a correlation handle), provides non-transferability only as the absence of a function, and offers no normative commitment for cross-implementation verification. This primitive addresses all three.

## Specification

Compact source below is compile-verified on compactc 0.31.0 / language 0.23.0 (`pragma language_version >= 0.23`).

### One instance = one credential class (normative)

`credentialDomain` and the issuer namespace are **sealed at deployment**. Consequently **each credential class (e.g. KYC vs proof-of-personhood vs DAO membership) is a separate contract deployment.** Integrators MUST NOT assume one instance serves multiple classes; a wallet/verifier flow is scoped to a single deployed instance. Sealing the class at deploy is deliberate — it binds a credential to its schema without per-operation overhead.

### Issuer authority (secret-derived)

Authorization MUST NOT use `ownPublicKey()` (a spoofable prover-supplied witness). Authority is proof of knowledge of a deploy-time secret:

```compact
struct IssuerSecretKey { bytes: Bytes<32>; }
struct IssuerPublicKey { bytes: Bytes<32>; }

export ledger issuer: IssuerPublicKey;
witness issuerSecretKey(): IssuerSecretKey;

export circuit deriveIssuerPk(sk: IssuerSecretKey): IssuerPublicKey {
  return IssuerPublicKey {
    bytes: persistentHash<Vector<2, Bytes<32>>>([pad(32, "mipa:issuer:pk:v1"), sk.bytes])
  };
}
```

### Deployment, namespace, and `deploymentSalt`

```compact
export sealed ledger issuerNamespace: Bytes<32>;
export sealed ledger credentialDomain: Bytes<32>;

constructor(domain: Bytes<32>, schemaId: Bytes<32>, deploymentSalt: Bytes<32>) {
  const initialIssuer = deriveIssuerPk(issuerSecretKey());
  issuer = disclose(initialIssuer);
  issuerNamespace = disclose(persistentHash<Vector<4, Bytes<32>>>(
    [pad(32, "mipa:ns:v1"), initialIssuer.bytes, schemaId, deploymentSalt]));
  credentialDomain = disclose(domain);
  // ... plant the IMT (0,0) sentinel
}
```

The namespace is derived from the **initial** issuer (so issuer rotation does not invalidate existing credentials) plus a **`deploymentSalt`**. `kernel.self()` is deliberately **not** used here: a constructor circuit is proven *before* the contract address exists (the address is derived from the deploy transaction, which embeds this initial state), so `kernel.self()` resolves to a dummy/zero address during construction — empirically two deployments with identical issuer+schema produced byte-identical namespaces under `kernel.self()`. **The deployer MUST supply a unique `deploymentSalt` per deployment**; reusing a salt across deployments with the same issuer+schema collapses their namespaces (see Security Considerations).

### Credential storage

```compact
export ledger issuedCredentials: HistoricMerkleTree<32, Bytes<32>>;  // membership (historic roots)
export ledger revokedSet: MerkleTree<32, Bytes<32>>;                 // Indexed Merkle Tree (current root only)
```

`issuedCredentials` is **Historic** so in-flight membership proofs survive later issuance. `revokedSet` is a **plain** `MerkleTree` (current-root-only) by design: a since-revoked holder cannot anchor a non-revocation proof to a stale pre-revocation root.

### Normative commitment suite `mipa:cred:v1`

A conforming credential commitment is exactly:

```
cm = persistentCommit<Vector<6, Bytes<32>>>([
  profileTag,                              // "mipa:cred:wallet:v1" (default) | "mipa:cred:bearer:v1"
  issuerNamespace,                         // sealed; binds class + issuer + deploymentSalt
  credentialDomain,                        // sealed class id
  holderPk,                                // Bytes<32>
  holderSecret,                            // holder control secret; CSPRNG, NEVER shared with the issuer
  (revocationHandle as Field) as Bytes<32> // see handle representation
], credentialNonce)                        // opening / randomness
```

`persistentCommit` (a **hiding** commitment) is required, not `persistentHash` (a hash is not a hiding commitment). The holder builds `cm` and gives the issuer only `cm`. Element order, tags, and representations are normative; a different layout is a different suite version. Conformance vectors (positive + negative) MUST accompany a reference implementation.

**Handle representation (normative).** `revocationHandle` is a `Uint<248>` — 248 is the maximum comparable `Uint` width in Compact, and ordering is required for the IMT bracketing below; 248 bits of randomness is cryptographically ample. It is embedded into the `Bytes<32>` commitment slot via **`(revocationHandle as Field) as Bytes<32>`**: a direct `Uint -> Bytes` cast is not permitted in Compact, so the value is routed through `Field` (`Uint -> Field -> Bytes`). **`revocationHandle` MUST be non-zero** — `0` is the reserved IMT sentinel (the `(0,0)` leaf) and is structurally unrevokable and unpresentable; issuers/holders MUST generate handles uniformly at random from `[1, 2^248)`.

### Circuits

```compact
// issuer-only: insert the holder's opaque commitment.
export circuit issue(cm: Bytes<32>): [] {
  assert(issuer == deriveIssuerPk(issuerSecretKey()), "Not authorized to issue.");
  issuedCredentials.insertHash(disclose(cm));
}

// issuer-only: revoke a handle by splicing it into the IMT. handle/bracket are public
// (a revocation is a public event); the predecessor path is a witness, and the splice index
// is DERIVED from the proven path (not a free parameter) so a wrong index cannot corrupt the tree.
export circuit revoke(handle: Uint<248>, lowValue: Uint<248>, lowNext: Uint<248>): [] { ... }

// hardened two-step rotation: current issuer proposes; the NEW issuer accepts by proving
// knowledge of the new secret (authority can never rotate to an unprovable key).
export circuit proposeIssuer(newIssuerPk: IssuerPublicKey): [] { ... }
export circuit acceptIssuer(): [] { ... }

// ONE bound presentation over a SINGLE opening (see Possession).
export circuit provePresentation(verifierChallenge: Bytes<32>): Bytes<32> { ... }
```

### Possession — one bound presentation (normative)

`provePresentation` takes a single credential opening and proves, in zero knowledge, all of:
1. **Membership** — the rebuilt `cm` (from the opening + the *sealed* `issuerNamespace`/`credentialDomain`) is a leaf of `issuedCredentials` (`merkleTreePathRootNoLeafHash<32>` + `checkRoot`), without revealing the leaf;
2. **Non-revocation** — the **same** `revocationHandle` is absent from `revokedSet`, proved by bracketing the sorted IMT (`low.value < handle < low.next`, with `0` = +∞) against the **current** root, without revealing the handle or which gap it occupies;
3. **Wallet profile only** — the holder controls the key behind `holderPk` (`holderPk == deriveUserPk(holderIdentitySecret)`); the bearer profile skips this. Expressed as an asserted implication so the profile bit is not disclosed. Note: the circuit evaluates the `holderIdentitySecret()` witness unconditionally (no ZK short-circuit), so a **bearer** presentation MUST still supply a witness value for it — any value (e.g. zero) works and is ignored, because the bearer disjunct (`profileTag != walletProfileTag()`) already satisfies the assertion.

It returns a disclosed **presentation nullifier** `persistentHash([pad(32,"mipa:present:v1"), cm, holderSecret, verifierChallenge])`. Binding both checks to one opening makes the "prove issued with the real handle, prove not-revoked with a fabricated handle" bypass structurally impossible. The nullifier binds `cm` (the full context) and stays unlinkable via the private `holderSecret`.

### Verifier integration (non-normative)

`provePresentation` **returns** the nullifier; it does not record it on-chain (the primitive is a presentation, not a spend). A verifier MUST therefore (1) issue a fresh `verifierChallenge` per presentation, and (2) persist accepted nullifiers in a per-session/per-audience store and reject repeats — otherwise a captured proof is replayable within the challenge window. One-time-use presentation semantics, if required, are a verifier-side store (or a future on-chain nullifier `Set`), not a property this read-only circuit enforces.

### Anonymity set (the privacy argument)

Revocation is a **public** event: `revoke` publishes the handle and its bracket, and anyone can walk the public IMT. This reveals **what** is revoked, not **who**. The credential commitment is hiding and never reveals which holder holds which handle; the bracket values are public regardless (derivable by walking the tree), so revealing the predecessor in a proof reveals nothing further. A holder's non-revocation proof discloses only a public Merkle root, never the handle or its gap position. The anonymity set is therefore all handles not already linked to a holder through out-of-band issuer records, issuance/presentation timing, or external information — *not* a guarantee against metadata correlation. (Framing per the problem statement's anonymity-set analysis.) Tooling note: `revoke`'s `lowValue`/`lowNext` bracket arguments are public transaction inputs (a revocation is a public event); wallet/issuer UIs should treat them as operational metadata and avoid surfacing them in a way that invites mapping handles to holders.

### Versioning (normative)

This MIP defines commitment suite `mipa:cred:v1`, issuer tag `mipa:issuer:pk:v1`, holder-key tag `mipa:holder:pk:v1`, namespace tag `mipa:ns:v1`, IMT leaf tag `mipa:revleaf:v1`, and nullifier tag `mipa:present:v1`. Any change to a preimage layout, element order, hash function, or domain tag is a new suite version.

### Non-transferability (scope)

Guaranteed: no non-issuer interface mutates a credential's binding. Out of scope: a holder sharing `holderSecret`/`holderIdentitySecret`, transferring their whole wallet, issuer-holder collusion, or duplicate issuance. The wallet profile resists *casual* sharing (presentation requires the identity secret) but cannot prevent a willing holder from delegating; full non-delegation needs hardware-bound keys / Schnorr-in-circuit and is out of scope for v1.

## Rationale

- Secret-derived issuer auth + holder-built commitment close the two central risks: no witnessed key is trusted for authorization, and the issuer never holds the holder's opening.
- Opaque `issue(cm)` + a normative suite keep issuance privacy-preserving while guaranteeing cross-implementation verification.
- `HistoricMerkleTree` for membership vs a plain `MerkleTree` for the revocation IMT is the asymmetry that prevents stale-root non-revocation bypass.
- One bound presentation circuit (not two) is what makes revocation enforceable under composition.

## Path to Active

Reference implementation merged, deployed to public testnet, exercised by a disclosure-proof flow (companion MIP) and a wallet, conformance vectors published, and reviewed for the authorization and correlation properties.

## Acceptance Criteria

- Reference contract compiles on the pinned toolchain and deploys to public testnet.
- A holder is issued a credential and proves possession via `provePresentation` without revealing the leaf, handle, or secrets.
- Secret-derived auth rejects a forged secret; no non-issuer interface mutates state; a wrong revoke index reverts.
- A revoked holder cannot produce a passing presentation.
- Two deployments with the same issuer+schema but distinct `deploymentSalt` derive distinct namespaces.
- Security review covering issuer impersonation, commitment forgery, revocation bypass, replay, and correlation.

## Implementation Plan

Stage 1 (this MIP): issuer auth, Merkle credential storage, the IMT revocation set, the normative suite, and the bound presentation. Stage 2 (companion): higher-level disclosure-proof composition / selective attribute disclosure. Stage 3 (companion): holder credential recovery/rotation.

## Backwards Compatibility Assessment

New primitive; no existing standard changes. Intentionally non-transferable/non-fungible — not interoperable with transfer surfaces, by design. No migration.

## Security Considerations

- **Issuer authorization** is secret-derived; `ownPublicKey()` is forbidden for authorization. Single-secret authority: leak ⇒ compromise until `rotateIssuer`; the two-step rotation prevents bricking. A multi-party authority needs Schnorr-in-circuit (future).
- **`deploymentSalt` is load-bearing.** Per-instance namespace separation depends entirely on the deployer supplying a fresh, unique salt; reuse across same-issuer+schema deployments collapses namespaces and re-enables cross-instance commitment confusion. `kernel.self()` cannot substitute (unavailable in-constructor).
- **`verifierChallenge`** MUST be a verifier-chosen, fresh, audience-bound 32-byte challenge; the contract cannot enforce freshness. Intentional reuse yields the same nullifier (in-session reuse detection); reuse across contexts degrades unlinkability. Anti-replay is the verifier's responsibility (fresh challenge per presentation).
- **Opaque issuance.** `issue(cm)` cannot validate well-formedness/uniqueness; issuers MUST only issue out-of-band-vetted `mipa:cred:*:v1` commitments with unique non-zero handles. Junk/duplicate leaves are unpresentable but consume tree slots. Because presentation now enforces holder identity (wallet) and opening knowledge, on-chain risk is limited to issuer-policy failures.
- **Holder control / delegation.** The wallet profile binds `holderPk = deriveUserPk(holderIdentitySecret)`; possession requires that secret. A willing holder can still delegate by sharing secrets — documented limitation.
- **Correlation.** Commitments carry no deterministic public holder id, preventing direct cryptographic linkage; metadata/timing/issuer-record correlation is not prevented (see Anonymity set).
- **Disclosure.** Only commitments, public Merkle roots, and the nullifier cross the boundary; `issuerSecretKey`, `holderSecret`, `holderIdentitySecret`, `credentialNonce`, and the handle never do.

## Implementation

Compact contract + TypeScript witnesses + conformance vectors. Reference implementation (Apache-2.0): https://github.com/DpacJones/midnight-nft — `contracts/credential-zk/`. Five provable circuits compile with ZK keys (compactc 0.31.0 / language 0.23.0); 28 tests including adversarial cases (forged secret, wrong revoke index, revoked-holder bypass attempt, wallet identity, namespace separation, deep IMT indices).

## Testing

Unit: issuer-only enforcement, forged-secret rejection, two-step rotation, `handle != 0`, wrong-index revoke reverts. Invariant: no non-issuer mutation; a revoked holder cannot present; same-holder credentials are distinct leaves. Privacy: secrets never reach the ledger; presentations reveal only public roots + nullifier. Namespace: distinct `deploymentSalt` ⇒ distinct namespaces. Conformance: positive/negative `mipa:cred:v1` vectors.

## References

- Soulbound and Non-Transferable Shielded Attestations (problem statement), midnight-improvement-proposals PR #211.
- MIP-0004, MIP-0011 (token standards this composes alongside).
- Reference implementation: https://github.com/DpacJones/midnight-nft (Apache-2.0).

## Acknowledgements

Problem framing and the binding-as-correlation-handle / anonymity-set analysis: the PR #211 author group.

## Copyright Waiver

Licensed under Apache-2.0; contribution subject to the repository Contributor License Agreement. All authorship is human (no AI authors/co-authors, per repo policy).
