---
MIP: X
Title: Signature-Authorized Shielded Spends with VRF Nullifiers
Authors:
  - Ricardo Rius (riusricardo)
Status: Draft
Category: Core
Created: 2026-08-28
Requires: none
Replaces: none
MPS: MPS-0035
Related-MPS: MPS-0024, MPS-0016
Related-MIP: MIP-0005, MIP-0006
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

Midnight's current shielded spending circuit receives the coin spending key as a private
witness. When a wallet delegates proving, that key crosses the proving-service boundary. A
compromised service or an observer who captures the request can obtain reusable spending
authority, not just the information needed for one payment.

This MIP introduces shielded note version 2, using a verifiable random function (VRF) to derive
each coin's nullifier without sending the spending key to the prover. The key holder generates
a payment-bound VRF proof locally. The shielded circuit verifies that the same key owns the
coin, generated its VRF output, and authorized the protected payment effects. The proving
service builds the final zero-knowledge proof from that evidence and the required coin data.

The design uses a Chaum-Pedersen equality-of-discrete-logarithms proof (DLEQ) over Jubjub to
combine VRF correctness and payment authorization. Its linear relations support MPC custody
without reconstructing the spending key. Existing notes remain spendable, and independently
proven offers remain mergeable.

**The objective is to separate custody from computation.** VRF-based nullifier derivation is
required. This document defines the expected behavior and its rationale; it leaves byte layouts,
circuit organization, and software interfaces to the implementation design.

**Supporting specification:** The [Shielded Note V2 specification](mip-xxxx/specification.md)
contains the detailed reference construction: cryptographic formulas, encodings, payment scopes,
circuit constraints, and validator rules. It distinguishes verified existing behavior from v2
requirements and unresolved protocol choices. This MIP defines the required properties and design
rationale; the attachment develops one realization without overriding the implementation latitude
described below.

## Motivation

A zero-knowledge proof hides its witness from the verifier, not from the machine generating
the proof. Encrypting the connection to a proof service protects transport but does not prevent
the service from reading a spending key included in the request.

This is also a custody problem. A secure device should perform a bounded local operation
without exporting its key. An MPC group should produce the evidence needed for a spend from
shares, without assembling the key at a coordinator or jointly computing the entire ZK proof.

V2 replaces the key-bearing witness with evidence limited to the intended spend. Merely hiding
the key is insufficient: the prover must also be unable to redirect the coin it is proving.
Conversely, signing a payment is insufficient unless the circuit establishes the correct,
repeatable nullifier that prevents double spending.

The change has three distinct parts:

- **VRF nullifiers** make per-coin nullifier derivation verifiable without revealing the key.
- **Payment binding** limits the use of that evidence to the protected effects of the spend.
- **Compatibility proofs** keep existing notes usable without publicly identifying the private
  note-version branch of a spend.

These solve different problems. None is a new on-chain permission system or a replacement for
Midnight's balance and execution rules.

## Place in Midnight's Architecture

Midnight separates private transaction preparation, provable computation, and public ledger
execution. Kachina explains this through transcripts of state operations: the proof establishes
that private inputs justify the published operations, while the ledger determines whether those
operations can be applied to its current state.

V2 retains this model. It changes the ownership and nullifier relation, not the commitment tree,
the nullifier database, or the requirement to conserve value. Transactions should not become
dependent on unrelated ledger state merely because the owner uses a VRF.

The final prover demonstrates possession of valid owner-issued evidence and matching coin
openings, rather than knowledge of the owner's spending key. That is the intended division of
responsibility. Kachina supplies the architectural foundation, not a security proof for this
specific replacement relation or its MPC implementation.

## Expected Solution

The terms MUST and MUST NOT identify security or compatibility properties that an implementation
cannot relax. The remaining design discussion explains the chosen approach without prescribing
an internal API or circuit layout.

### A VRF-derived nullifier

Each v2 coin MUST have one deterministic nullifier derived from a VRF under its owner's
spending key. The VRF input identifies the coin and its owner. It does not include payment
details, proof randomness, the Merkle path, or the set of MPC participants.

Consequently, retrying a payment, changing its recipients, refreshing a path, or choosing another
valid signing subset cannot give the same coin a different nullifier. Fresh coin nonces remain
necessary to distinguish otherwise identical outputs.

The selected construction has the following conceptual relations:

```text
public spending key = spending secret * G
per-coin base      = hash_to_curve(coin identity, owner)
VRF output         = spending secret * per-coin base
nullifier          = domain-separated derivation from the coin and VRF output
```

These describe the relationship, not a serialization or hash-suite definition. The coin
commitment must bind ownership to the public key used in the VRF relation, directly or through
its recipient-key representation. The circuit must link that commitment to the spent leaf.

The hash-to-curve relation MUST admit exactly one valid non-identity subgroup point for each
input. A deterministic host implementation is not enough if the circuit accepts alternative
points: each accepted alternative is another base on which the key holder can honestly evaluate
the VRF, and therefore another nullifier for the same coin. The key holder derives or
independently checks this base before using its secret; arbitrary host-selected points are not
acceptable.

A simple hash-to-scalar multiplied by the generator is not a substitute for a secure
hash-to-curve map: its known discrete logarithm would let anyone with the public spending key
compute the VRF output. Uniqueness and resistance to public evaluation are separate properties.

### A payment-bound VRF proof

The key holder produces one local DLEQ proof that both certifies the VRF output and binds its use
to a digest of the protected payment effects. A VRF output obtained for viewing is not spending
permission. The circuit MUST reject a spend lacking the corresponding payment-bound evidence.

The spending key, wallet seed, key shares, and signing nonces MUST NOT enter a v2 proving
request. The prover may receive coin openings, a Merkle path, commitment randomness, and the
local proof. It receives enough to prove the intended relation, not enough to issue another
authorization. This restriction applies to v2 shielded proving, not to the unchanged v1 and
Dust paths.

The local proof is part of the normal Send operation. It does not introduce a separate approval
transaction, a second spending signature, or a mandatory extra user confirmation. Hardware
confirmation and MPC coordination remain matters of custody policy.

### Payment binding that preserves offer merging

The protected payment effects include the inputs being spent, outputs and change being created,
and any contract intents on which the payment depends. The binding also covers the network and
execution segments. A prover MUST NOT be able to remove, substitute, or move protected records
without invalidating the owner's evidence.

Existing balance and binding-commitment checks do not replace this authorization. A prover
given the necessary openings can construct different outputs and recompute the corresponding
balance binding. The missing property is the owner's approval of those effects.

Binding the whole final transaction would prevent independently proven offers from being merged
unchanged. Instead, the design binds each participant's protected contribution. A successful
merge may add compatible contributions but must preserve the meaning of every existing binding.
Changing proof bytes must not disguise a duplicated input or nullifier.

This document uses **payment scope** to mean that protected contribution, not a required wire
type. Developers can choose its representation and the way references are resolved, provided
the validator can establish coverage unambiguously. Payment binding excludes final proof bytes
to avoid circular dependencies. One shared scope for a wallet's contribution in each execution
segment is the normal case; there is no need to make every input a separate approval workflow.

Scope completeness remains the wallet's responsibility. The ledger can check referenced records
and require every v2 input to be authorized, but it cannot infer which outputs the user intended
to create. The wallet therefore finalizes the protected records before generating the local
proof, and regenerates that proof when they change.

Protected inclusion is not guaranteed execution. A guaranteed input and an output in a fallible
segment do not become atomic merely because one digest names both. Applications needing atomic
payment conditions must respect the existing segment semantics.

### Verification remains a ledger responsibility

A valid v2 spend must establish all of the following, regardless of circuit organization:

- The spent commitment belongs to the declared commitment tree and binds the claimed owner.
- The VRF output was derived for that coin by the same key that owns it.
- The owner's evidence binds the protected payment effects checked by the validator.
- The public nullifier matches that VRF output, and the existing uniqueness rule prevents reuse.
- The value commitment matches the coin's value, asset type, execution segment, and blinding.

Outputs must likewise establish their recipient commitment and retain existing value and
ciphertext-binding rules. Coins created and consumed within one transaction must keep the
existing transient relationship between their input and output proofs.

All required checks must be enforced by the final proof and ledger validation, not merely by an
honest constructor. A correctly formed DLEQ proof alone does not establish Merkle membership,
balance, scope completeness, or transaction validity.

## Why These Cryptographic Choices

### DLEQ rather than a single Schnorr proof

Ordinary Schnorr already supports threshold signing. The reason to use Chaum-Pedersen DLEQ is
the additional need to prove that the ownership key and VRF output use the same secret.

A single Schnorr signature can approve a proposed VRF output without proving it was derived
correctly. A dishonest key holder could then approve unrelated outputs and obtain multiple
nullifiers for one coin. Two independent proofs of knowledge do not establish equality of the
secrets either.

With public key `P`, per-coin base `H`, VRF output `Y`, and nonce commitments `R_G` and `R_H`,
the DLEQ verifier checks one response `s` against one challenge `c`:

```text
s * G = R_G + c * P
s * H = R_H + c * Y
```

The challenge binds the complete DLEQ statement and the payment digest. The first equation
establishes key ownership; the second establishes that the same key produced the VRF output.
Together they provide VRF correctness and payment binding in one local proof, without a
separate VRF proof followed by an independent spending signature.

Hashing an ordinary signature into a nullifier would not achieve the required uniqueness.
Different nonces or payment digests yield different signatures, and signature verification
does not enforce a signer's private deterministic-nonce procedure.

### Jubjub and circuit-friendly hashing

Jubjub is the selected curve because its base field matches Midnight's circuit field and the
ledger already uses compatible point operations. This avoids introducing non-native field
arithmetic solely to use a more familiar curve. It is an engineering choice for circuit cost
and reuse, not a claim that Jubjub is intrinsically more secure or universally supported by
secure elements.

Circuit-friendly hashing is appropriate for the internal VRF and key-binding relations. The
design retains the persistent SHA-256 outer shape of commitments and nullifiers so their byte
representation does not trivially distinguish v2 notes. A bare field encoding can carry fixed
bits that reveal a different format.

The exact hash parameters, domain labels, encodings, and curve-map construction belong in a
common cryptographic profile. Consensus-visible encodings and the per-note derivations MUST be
specified consistently before deployment and remain stable for that note version. They cannot
silently change with a generic transient-hash upgrade. Any truncation or conversion used inside
the commitment must be included in the binding-security analysis.

### MPC without reconstructing the key

Both public-key derivation and VRF evaluation are linear in the secret. Participants can compute
public-key shares and per-coin VRF shares and combine them with the appropriate reconstruction
coefficients. A coordinated threshold protocol can similarly produce an aggregate DLEQ proof
that the circuit verifies in the same way as a single-key proof.

This makes MPC custody a supported use case without putting participant lists or threshold
logic into the shielded spending circuit. A single local key holder remains a valid deployment;
users do not need MPC to benefit from non-custodial proving.

Independent signatures cannot simply be added. Participants must agree on the key, coin base,
payment digest, and common challenge, while handling malicious shares and nonce safety.
One-base FROST is not a drop-in protocol for this two-base relation. The threshold construction
requires its own security analysis and must reconstruct neither the full key nor a signing
nonce at the coordinator.

## Payment Flow and Custody Boundaries

1. **Set up.** A local signer or MPC group holds the spending key and exports its public key.
   The wallet combines the recipient-key representation with the existing encryption public key
   to form a versioned receiving address.
2. **Receive.** A sender creates a v2 output without needing the recipient's spending key. The
   receiving wallet decrypts and matches the actual commitment before crediting the coin. It
   obtains the coin's VRF output from the key holder when spend detection is needed.
3. **Prepare.** The wallet selects inputs and fixes the protected recipients, amounts, change,
   conditions, and segments. It computes the payment digest from those records.
4. **Generate the payment-bound VRF proof locally.** The key holder derives the coin's base and
   generates the DLEQ evidence for that digest, subject to custody policy. No spending key is
   exported, and no on-chain approval is created.
5. **Prove.** The service receives the local evidence and required private data, builds the final
   ZK proof, and returns it. The wallet verifies the returned proof before broadcast.
6. **Settle and reconcile.** The network validates the complete transaction and applies its
   effects. The wallet follows actual segment outcomes and chain finality, not proof completion
   alone.

A failed prover can be replaced using the same cached local evidence while the payment and
ledger prerequisites remain valid. A changed protected record requires new evidence. A pending
submission is not a confirmed spend, and a failed conditional output is not a completed payment.

The key holder is trusted to make the authorization decision. The prover is not trusted with
spending authority, but does see the supplied transaction details. Validators receive the public
records and final proofs, not the private VRF/DLEQ witness.

## Viewing and Recovery

Receiving and spending remain separate capabilities. The existing encryption path is retained;
the wallet identifies ownership and note version by matching the decrypted coin's commitment
against its held receiving keys. Successful decryption alone is not evidence of ownership.

A v2 incoming viewing key can identify receipts without a spending key, but cannot independently
derive nullifiers. The key holder can provide per-coin VRF outputs for the wallet to build a
nullifier list. Combined with incoming viewing information, a complete list supports spend
detection and auditing without spending authority. Missing entries mean incomplete visibility,
not that every received coin remains unspent.

The derivation should reuse the existing Zswap seed role with separation from v1 and encryption
keys, avoiding unnecessary new backup material. A key created by distributed key generation
(DKG), however, is not automatically recoverable from that wallet seed and needs its own share
recovery arrangement.

Restoring or refreshing shares must preserve the public spending key to retain authority over
existing notes. Changing the key requires a transfer while the old key can still authorize.
Viewing records and provers cannot replace a lost spending key. Recovery must not restore nonce
state in a way that permits reuse.

## Backwards Compatibility and Network Impact

V2 changes consensus verification and therefore requires a coordinated network upgrade. A
wallet-only change cannot make a VRF witness satisfy the existing v1 circuit or verifier key.

Existing v1 notes MUST remain spendable under their original ownership and nullifier relation.
Migration is a transfer to a v2 address, not reinterpretation of an existing leaf. A wallet that
must keep its old key private produces the v1 input proof locally; later spends of the v2 output
can be delegated without exposing either spending key.

The design keeps one commitment tree and one nullifier set. Compatibility circuits with a
private note-version selector allow v1 and v2 user-owned notes to share the same proof format
and anonymity set. The selected branch must not be revealed through public fields, proof size,
or different responses to a modified public payment digest. This does not guarantee identical
real-world anonymity: publicly known note history and application behavior can still distinguish
subsets of coins.

Validators need an unambiguous way to select legacy or compatibility verification and reject
unknown formats. The representation of that choice is not fixed here. The transition should
move user-owned proofs to the common compatibility path while leaving v1 notes spendable;
activation timing and any retirement schedule for legacy user proof formats belong to the
network upgrade, not to a new ownership rule for old notes.

The design preserves value commitments, including their asset and segment dependence,
balancing, execution, replay protection, and fee rules. New proof and scope-processing costs
still need accounting; unchanged fee rules do not promise identical fees. Compatibility
circuits are intended to use the published, provenance-checked parameters rather than require
a new ceremony. If they cannot fit those resources, that is a protocol design decision to
revisit, not permission to weaken validation or silently change the trust assumptions.

Offer-file and atomic-swap flows described by MIP-0005 and MIP-0006 remain supported. Contract
execution, including fallible segments and transient coins, retains its current meaning.
Indexers may keep the existing ciphertext relevance filter, but must understand the activated
ledger formats and outcomes. A new receiving address must be distinguished from v1 so senders
choose the correct output relation.

Contracts cannot yet pay v2 recipients. The Compact standard library and its runtime express a
user recipient only as a v1 coin public key, so a coin that a contract mints or sends to a user
is a v1 note, and a v2 recipient encoded into that field would make the coin unspendable.
Extending that toolchain is a separate change against which deployed contracts must be
redeployed; this MIP fixes only the v2 output relation it must produce. Until then a user who
receives from contracts keeps a v1 receiving address, and those coins reach v2 only by a later
transfer.

## Security Considerations

**Captured proving requests.** Someone obtaining the complete v2 payload may reproduce or submit
the authorized scope, link requests, or withhold service. They MUST NOT gain the ability to
recover the key, authorize another coin, or alter protected payment effects. Authenticated,
encrypted transport remains necessary for privacy and integrity; the design does not rely on
transport confidentiality alone to protect spending authority.

**A compromised host.** Key isolation is not safe authorization by itself. A device that blindly
signs a host-supplied digest can approve a malicious payment without losing its key. Deployments
claiming protection against host compromise need trusted user approval or independent policy
over the actual recipients, values, network, and conditions. Display text supplied by the same
untrusted host is insufficient.

**Cryptographic validity.** Valid encodings, subgroup membership, and rejection of prohibited
identity points are security requirements, not parsing conveniences. Scalars and challenges
must have agreed interpretations in the signer and circuit. The implementation must establish
the canonical VRF relation in the proof, not rely on honest host calculations. Signing nonces
must stay secret and avoid reuse; leakage or reuse can reveal the spending key. The exact nonce
derivation and circuit gadgets are implementation choices subject to these properties.

**Authorization lifetime.** A local proof does not reserve a coin or create a revocable
permission. Switching provers or refreshing shares under the same key does not cancel issued
evidence. Expiry, where required, must be enforced by existing ledger validity mechanisms and
bound to the payment, not an unbound wallet timer. Competing spends still share one nullifier;
normal ledger rules determine which, if any, succeeds.

**Privacy.** Delegated proving is non-custodial, not private from the prover. Public payment
bindings may also reveal how participants' contributions were grouped in a merged transaction.
The representation should disclose no more than needed for validation, but this tradeoff must
be explicit. Kachina's ideal NIZK interface hides witnesses from the adversary; external witness
disclosure does not inherit that guarantee.

**Security evidence.** The modified payment relation, its binding mechanism, and any threshold
protocol need independent cryptographic analysis. Kachina's results do not automatically extend
to this construction; its treatment of general cross-contract composition and multiparty
ownership is expressly limited. Correct equations and a plausible architecture are not evidence
that a compiled circuit or complete transaction is valid.

## Implementation Latitude

The protocol properties above are fixed; the low-level recipe is not. Developers can choose the
simplest construction that preserves them. The
[supporting specification](mip-xxxx/specification.md) records the detailed reference
design and identifies the remaining interoperable choices:

| Area | Property to preserve | Detail left to implementation design |
| --- | --- | --- |
| Cryptographic profile | A unique, secure VRF with stable note derivations and domain separation. | Hash parameters, curve-map algorithm, canonical encodings, and challenge framing. |
| Payment binding | Exact protected effects, network and segment binding, and safe offer merging. | Scope layout, reference representation, digest encoding, and validation data structures. |
| Compatibility proofs | Complete ownership checks, identical branch-visible format, and digest binding. | Witness layout, branch selection, transcript placement, and compiler gadgets. |
| Key holding | No key or nonce export to the prover, with the same public relation for local and MPC use. | Device APIs, secure nonce derivation, threshold protocol, and recovery tooling. |
| Deployment | Continued v1 spendability, deterministic verification, and bounded consensus work. | Version tags, artifact packaging, resource accounting, and upgrade scheduling. |

These choices are not independent per wallet or validator: consensus-relevant behavior must be
specified once and shared across implementations before activation. Flexibility in this MIP is
not permission for incompatible encodings or different accepted spending relations.

## Out of Scope

Contract-owned coins retain their existing ownership rules; they do not acquire a v2 user
spending key. Dormant user-claim functionality is not reactivated by this MIP, and the separate
rewards-claim transaction remains unaffected.

Dust is unchanged. Its current proving path still uses its own secret, so v2 shielded key
isolation does not make every request key-free. No new allowances, delegate registry, recovery
override, or on-chain authorization lifecycle is introduced. The MIP does not protect against
total custody compromise, hide witnesses from a remote prover, or guarantee service availability.

## References

- [Shielded Note V2: Supporting Specification](mip-xxxx/specification.md), the detailed
  reference construction, code baseline, and unresolved protocol choices supporting this MIP.
- [MPS-0035: Shielded Spend Authorization Requires Exposing the Spend Key](../mps/mps-0035-shielded-spend-key-exposure.md).
- [MPS-0024: Custodian-Safe Native Shielded Asset Transfer](../mps/mps-0024-custodian-safe-shielded-spends.md)
  and [MPS-0016: Custodian-Compatible Shielded Wallet Generation](../mps/mps-0016-custodian-shielded-wallet-generation.md),
  related custody problems addressed by this design.
- [MIP-0005: Offer Files](mip-0005-offer-files.md) and
  [MIP-0006: P2P Atomic Swaps](mip-0006-p2p-atomic-swaps.md), related merge-based flows, not dependencies.
- Chaum and Pedersen, "Wallet Databases with Observers," CRYPTO 1992, for DLEQ proofs.
- [RFC 9381: Verifiable Random Functions](https://www.rfc-editor.org/rfc/rfc9381), for security
  definitions; this MIP does not specify an RFC 9381 wire-compatible ciphersuite.
- [RFC 9591: FROST](https://www.rfc-editor.org/rfc/rfc9591), for threshold Schnorr background,
  not a complete protocol for the two-base relation used here.
- [Zcash Protocol Specification (NU5)](https://zips.z.cash/protocol/protocol.pdf), for comparison
  with shielded ownership, nullifiers, and spend authorization.
- [Nightpaper: A litepaper introducing Midnight](https://45047878.fs1.hubspotusercontent-na1.net/hubfs/45047878/Midnight%20litepaper.pdf),
  architecture, pp. 10-11, for private/public state separation and Compact.
- Thomas Kerber, Aggelos Kiayias, and Markulf Kohlweiss,
  [Kachina - Foundations of Private Smart Contracts](https://eprint.iacr.org/2020/543.pdf),
  revision 4, 2021, Cryptology ePrint 2020/543, IEEE CSF 2021. Sections 4-5 and Appendix C.4
  describe transcripts and private payments; Section 4.2 and Appendix I discuss composition
  and trust-model limits.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement,
which includes the assignment of copyright for your contributions to the Foundation.
