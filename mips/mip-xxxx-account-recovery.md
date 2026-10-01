---
MIP: MIP-XXXX
Title: Recovery Paths for Custody Accounts
Authors:
  - Nicolas Di Prima (NicolasDP)
  - Raphael Toledo (rrtoledo)
Status: Draft
Category: Standards
Created: 2026-08-24
License: Apache-2.0
Requires: MIP-0012, MIP-0013
Replaces: N/A
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
 See the License for the specific language governing permissions and
 limitations under the License.
-->

<!-- WORKING DRAFT. Strip this comment at submission. Markers:
     [CRYPTO-MEMO Qn] = a clause whose final normative form awaits the
     commissioned cryptographic review (sent, response pending);
     [EXP: met: ...] = an acceptance criterion whose evidence run has
     completed on the reference implementation (contract/). -->

## Table of contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Specification](#specification)
  - [1. Roles and objects](#1-roles-and-objects)
  - [2. Keys, secrets, and domain separation](#2-keys-secrets-and-domain-separation)
  - [3. Guardian identity and key derivation](#3-guardian-identity-and-key-derivation)
  - [4. Share derivation](#4-share-derivation)
  - [5. The session operation](#5-the-session-operation)
  - [6. The recovery operation](#6-the-recovery-operation)
  - [7. Guardian transport](#7-guardian-transport)
  - [8. On-chain structures](#8-on-chain-structures)
  - [9. Invariants](#9-invariants)
  - [10. Versioning](#10-versioning)
- [Rationale](#rationale)
- [Path to Active](#path-to-active)
- [Backwards Compatibility Assessment](#backwards-compatibility-assessment)
- [Security Considerations](#security-considerations)
- [Implementation](#implementation)
- [Testing](#testing)
- [References](#references)
- [Acknowledgements](#acknowledgements)
- [Copyright Waiver](#copyright-waiver)

## Abstract

Custody accounts under MIP-0012 and MIP-0013 are controlled by rolling
per-device keys derived from platform authenticators. There is no seed
phrase: loss of every enrolled device is, without further provision,
loss of the account, and loss of the account's viewing secret orphans
its shielded holdings even when control is regained. This proposal
specifies recovery paths for such accounts. The core mechanism is a
community-based backup of an ephemeral recovery secret using bottom-up
secret sharing, in which guardians persist no per-account state: each
guardian's share is recomputed on demand from a secret bound to their
own authenticator credential. The account contract stores a public
recovery artefact, a commitment gating a recovery operation, and an
authenticated encryption of the account viewing key under a key derived
from the recovery secret. Recovering the secret therefore restores both
control (a new device key is enrolled through the recovery gate) and
visibility (the viewing key decrypts). The specification defines the
guardian model, share derivation, the session operation that publishes
and refreshes the artefact set, the recovery operation with its veto
window, the off-ledger guardian transport, and the invariant family a
conforming implementation must satisfy. Profiles instantiate the
guardian secret for WebAuthn PRF credentials, for cold signers, and
for paper keys.

## Motivation

MPS-0018 names recovery as an unresolved obligation of the custody
model: an account whose authorisation set (the MIP-0013 device set)
can be lost needs a specified path back that does not reintroduce a
seed phrase or a custodian.

### The failure mode is total and silent

An account under MIP-0013 is controlled by a set of device entries,
each a commitment to a key that the device's platform authenticator
holds and that never leaves it. This is deliberate. The seed phrase is
the single most common cause of both loss and theft in self-custody,
and an account that never mints one cannot leak one. The cost of that
choice is concentrated in one place: when the last enrolled device is
gone, nothing in the protocol can distinguish the legitimate owner from
anybody else, and the account becomes permanently unreachable.

Under contract custody the assets are not destroyed. They remain held
by the contract exactly as they were, correctly accounted, and visible
in the ledger. They are simply beyond reach forever. This is the
zombie-state failure in its sharpest form: public custody state that
survives in perfect health while the private state that governs it has
ceased to exist.

### Restoring control is not sufficient

An account's shielded holdings are governed by a viewing secret as well
as by an authorisation set. An owner who regains control but not
visibility holds an account whose shielded balance cannot be
enumerated, whose incoming coins cannot be detected, and whose value
therefore cannot be spent. Any recovery path that restores only the
authority to sign leaves the account half-recovered. The specification
consequently treats visibility as part of the object to be recovered,
and states the retroactive-trust cost that this choice carries in
Security Considerations.

### Why existing Midnight capabilities are inadequate

MPS-0018 reserves a recovery-paths MIP in its Recommended MIPs
section, naming helper-stored social recovery and encrypted-blob
backup as candidate shapes; no MIP has been drafted against that
slot, and the merged register contains no other proposal addressing
recovery, guardians, or account backup. Two adjacent tracks, both in
review at the time of writing, explicitly defer to this one: the
soulbound attestation problem statement scopes itself to credential
re-binding, recommends a separate credential-recovery MIP for that
purpose, and refers asset recovery to the MPS-0018 orbit; the
soulbound credential primitive defers holder recovery to a companion
recovery document that has not been drafted. Neither covers custody
recovery, and this proposal does not cover credential re-binding.

This proposal also refines the problem statement's framing in one
respect. Of the two candidate shapes MPS-0018 names, the Rationale
selects neither, and instead a third: bottom-up secret sharing, which
keeps the distributed trust of helper-stored social recovery while
asking helpers to store nothing, a property neither named shape has.
The grounds are stated in Rationale.

MIP-0015 (Wallet-derived deterministic secrets) takes the opposite
branch of the same fork and is worth stating precisely, because its
rationale contains the objection this proposal answers. MIP-0015
derives application secrets deterministically from a BIP-32 seed, and
rejects the WebAuthn PRF extension as an alternative on the ground that
a PRF secret is credential-bound: lose the authenticator, lose the
secret. For a wallet that already has a seed, that reasoning is sound
and MIP-0015 is the right construction. It is unavailable to an account
whose design premise is that no seed exists anywhere. The two documents
are therefore complementary rather than competing: MIP-0015 obtains
durability from a mnemonic, and this proposal obtains it from a
community, so that credential-bound secrets can stay credential-bound
without the account becoming unrecoverable.

MPS-0035 (shielded spend authorisation requires exposing the spend key)
argues for signature-based spend authorisation so that keys can live in
a secure element rather than be witnessed in the clear. That is the
same seam this proposal gates, and a recovery path must not assume the
authorising key is extractable. The specification is written so that
the recovery gate binds a successor authority without ever requiring
the old authority to be exported.

### What a solution must not do

Three shapes are ruled out by the setting rather than by preference. A
seed phrase reintroduces the failure mode the account design exists to
remove. A custodian who can restore the account is a party who can also
take it, which contradicts the custody model. A recovery mechanism that
requires guardians to hold durable per-account state makes every
guardian a small custodian with a backup problem of their own, and the
empirical record of such designs is that the state is what rots.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in RFC 2119.

### 1. Roles and objects

- **Account**: a custody contract per MIP-0012 with authorisation per
  MIP-0013.
- **Owner**: the party controlling the account's enrolled devices.
- **Guardian**: a party assisting recovery. A guardian identity is a
  single authenticator credential (see section 3), not a person; one
  person MAY hold several guardian identities.
- **Recovery secret `s`**: an ephemeral field element. It exists during
  session setup and during recovery only. Implementations MUST NOT
  persist `s` (see REC-6 and Security Considerations).
- **Viewing key `vk`**: the account's stable shielded viewing secret:
  the private half of the encryption key MIP-0012 stores as `enc_key`
  and that inbox entries are encrypted to. It is created at account
  birth and is independent of the recovery subsystem.
- **Artefact set**: the on-chain triple published per session:
  public shares `phi`, recovery commitment, and viewing-key wrap.
- **Session**: one execution of the operation in section 5, identified
  by a session identifier and characterised by a freshly sampled `s`.
- **Roster**: the guardian set of a given session, of size `n`, with
  reconstruction threshold `t+1`.

### 2. Keys, secrets, and domain separation

`s` is a field element of the scalar field exposed by the Compact
`Field` type. It MUST be sampled uniformly at random from that field by
a cryptographically secure generator, independently for every session.
An implementation MUST NOT derive `s` from any prior `s`, from the
session identifier, or from any account-durable value: independence
across sessions is the property the whole construction rests on.

Two values are derived from `s`, and they MUST NOT share a construction:

- the **gate commitment**, stored on-chain and used inside the recover
  circuit (section 6): the curve point `P = s * G`, where `G` is the
  prime-order generator of the account's signature curve and `s` is
  reduced into that curve's scalar field. `P` is the public key of a
  signature the recover gate verifies; the gate proves knowledge of
  `s` by verifying a signature under `P` rather than by recomputing a
  hash preimage of `s`, so that `s` never enters the proof and the
  proof may be produced by a delegated prover (section 6, REC-11). The
  commitment is binding, and is computationally hiding under the
  discrete-logarithm assumption given the full entropy of `s`; the
  offline-search hardness this rests on is unchanged from a hash
  commitment and is analysed in Security Considerations. Its
  properties in this construction are put to [CRYPTO-MEMO Q7].
- the **wrap key**, used to encrypt `vk` (section 8): derived from `s`
  under the `wrap` tag below.

Domain separation follows the tag conventions MIP-0013 establishes
for this account family (section 3 for the entry tag, section 5.1 for
the per-circuit tags, as amended by the merged derivation erratum).
The tags below are to be registered under the registry recommended by
MPS-0027 once it ratifies. This proposal registers:

| Tag | Purpose |
|---|---|
| `midnight:account:recovery:guardian:v1` | guardian secret derivation (section 3) |
| `midnight:account:recovery:share:v1` | share derivation (section 4) |
| `midnight:account:recovery:submit:v1` | the recover-gate signature challenge (section 6) |
| `midnight:account:recovery:wrap:v1` | wrap key for `vk` (section 8) |

A conforming implementation MUST use these tags and MUST NOT reuse a
tag from another family for these purposes.

The two seam-gated operations this proposal adds are circuits on the
MIP-0013 seam and take their per-circuit tags from MIP-0013 section
5.1, which derives the tag from the circuit name. So that independent
implementations derive the same tags, the circuits are named
`publish_recovery_session` (section 5) and `recover_cancel` (section
6 step 5), and their tags are therefore
`midnight:account:auth:v1:publish_recovery_session` and
`midnight:account:auth:v1:recover_cancel` on the JubJub Schnorr arm,
with the corresponding `midnight:account:auth:k1:v1:` forms on the
interim secp256k1 arm the reference implementation also carries. The
recover gate itself (`recover_submit`, section 6 step 4) and the
permissionless finalisation (`recover_finalise`, step 6) are not seam
operations; the gate uses the `submit` tag above.

The exact key-derivation construction (an extract-then-expand
function), the uniform sampling procedure for `s`, and the mapping of a
32-byte authenticator PRF output to a field element without modulo bias
are the subject of [CRYPTO-MEMO Q4] and [CRYPTO-MEMO Q5], and are
stated here as MUSTs once that memo lands. Until then, implementations
SHOULD use a wide-reduction or rejection-sampling construction rather
than plain modular reduction of 256 bits against a modulus of
approximately 255 bits.

### 3. Guardian identity and key derivation

Core contract (normative): a guardian secret MUST be deterministically
recomputable on demand from the guardian's durable authenticator, MUST
require no stored per-account state, and MUST be domain-separated from
every other secret derived from the same authenticator.

- **Profile A (WebAuthn PRF)**: the guardian secret is the PRF output
  of the guardian's credential under the registered salt
  `midnight:account:recovery:guardian:v1`, mapped to a field element as
  specified in section 2. Evaluation requires user verification; the
  secret exists transiently per ceremony. A credential synced by its
  provider serves from any of the guardian's devices; a device-bound
  credential serves from that authenticator only. Wallets SHOULD
  surface which kind a guardian enrolled, because the two carry
  materially different availability and loss profiles.
- **Profile B (cold signer)**: the guardian secret is the hash, under
  the registered guardian tag, of a deterministic signature over the
  session binding: the byte string formed by the guardian tag, the
  account identifier `pk_i`, and the session identifier `sid_i`, each
  fixed-width encoded, in that order. The signature scheme MUST be
  deterministic in fact as well as in name: RFC 6979 qualifies as
  published; BIP-340 qualifies only with its auxiliary randomness
  input pinned to thirty-two zero bytes, because its default signing
  algorithm mixes fresh randomness into the nonce. WebAuthn
  signatures are not deterministic and MUST NOT be used with this
  profile. A Profile B guardian secret is per-session by
  construction, since the signed message covers `sid_i`; this
  strengthens cross-session independence and preserves REC-3,
  because the signature is recomputable on demand from the signer
  alone.
- **Profile C (paper key)**: the guardian secret is a uniformly
  sampled field element generated at enrolment, printed or otherwise
  recorded physically, and never stored by any device. It
  participates in share derivation (section 4) identically to the
  other profiles. Profile C exists for the cold-start case in which
  an owner has no community to draw guardians from: paper keys act
  as virtual guardians under physical custody, at the cost of
  reintroducing a physical artefact with the loss and theft profile
  the account design otherwise removes. A wallet offering Profile C
  MUST treat each paper key as a distinct guardian identity and
  SHOULD warn that paper keys held in one place collapse to a single
  effective guardian (see the people-counting rule in Security
  Considerations).

A guardian identity is the credential, not the person. Aggregation of
several credentials held by one person is a wallet presentation
concern; parameter guidance for collusion resistance counts people, not
credentials (see Security Considerations).

Cross-wallet portability of a guardian additionally depends on shared
relying-party infrastructure for credential evaluation; this is a
stated dependency, not delivered by this proposal. A WebAuthn PRF
output is evaluable only by a client asserting under the credential's
relying party, so a Profile A guardian identity is in effect the pair
of credential and relying party: a credential transferred to a new
device remains the same guardian, because the guardian secret follows
the credential, while a loss of the relying party loses the guardian
as surely as deleting the credential would. The section 7 liveness
attestation is the only detector for either loss. Authenticator
credential-exchange formats now shipping in platform authenticators
change what losing a credential means over the life of an account.

### 4. Share derivation

A guardian's share for a session is derived, not stored:

```
sigma_ij = H(DST_share, sid_i, pk_i, sk_j)
```

where `DST_share` is `midnight:account:recovery:share:v1`, `sid_i` is
the session identifier, `pk_i` identifies the owner account, and `sk_j`
is the guardian secret from section 3.

Inputs MUST be encoded unambiguously, with fixed-width or
length-prefixed encoding for every field, so that no two distinct input
tuples produce the same preimage. `H` MUST be the hash registered for
the account's profile.

The security statement under which this construction is claimed across
many sessions, and the precise conditions under which correlated
sessions would degrade the threshold, are the subject of
[CRYPTO-MEMO Q1] and [CRYPTO-MEMO Q2]. The published scheme's formal
model is single-session; the multi-session rules in section 5 are this
specification's own normative addition and the memo either establishes
or refutes them.

### 5. The session operation

There is exactly one artefact-publishing operation: **new recovery
session**. A session MUST use a session identifier distinct from every
identifier previously published for the account (REC-4), MUST sample a
fresh `s`, MUST collect fresh shares from every guardian in the new
roster, and MUST publish the artefact set atomically in one
transaction. There is no standalone rewrap or partial-update operation;
public-share update without a full re-share MUST NOT be used unless
[CRYPTO-MEMO Q3] affirms its safety.

The session operation is an authorised operation on the MIP-0013 seam:
a conforming contract gates it with the same in-circuit verification
as every other state-changing circuit, under its own per-circuit
domain-separation tag, and its execution advances the authorisation
nonce and the round counter (MIP-0012 INV-7). An artefact set that
could be published without an active device's authorisation would be
an account-takeover primitive; the recover operation of section 6 is
the single deliberate exception to the seam, and no session or cancel
operation shares that exception. Note the liveness asymmetry this
creates: a session requires every guardian in the roster to respond,
while a recovery requires only `t+1` of them, so the operation that
repairs a roster is strictly harder to run than the operation that
consumes it.

Variants:

- **Routine** (roster change, credential replacement): `vk` unchanged;
  the wrap re-encrypts the same `vk` under the new wrap key.
- **For cause** (a guardian is no longer trusted): the session SHOULD
  be accompanied by rotation of `vk` through the MIP-0012 encryption
  key lifecycle (the `rotate_enc_key` path) and re-encryption of the
  inbox entries for all held coins under the new key (churn), because
  artefact history is permanent and a hostile past quorum otherwise
  retains a decryption capability for the current `vk`.

Ordering: a session MUST complete before any churn that depends on it.

The contract MUST reject a session whose identifier equals the
currently stored one, MUST reject an all-zero session identifier
regardless of the stored value (it is what a broken generator
produces, and the stored value catches it on the first session only),
and MUST reject a recovery commitment equal to the currently stored
one (defence in depth against accidental reuse). The contract MUST
also reject a recovery commitment `P` that is the curve identity or
has small order, with the check of section 6 step 4: a signature under
a small-order key is forgeable, and a session that installed one would
hand the next recovery to anyone. A session MUST also be rejected
while a recovery is pending
(section 6): a session rotates the commitment, and a pending record
enacted after that rotation would enrol a successor the rotated
commitment never authorised. The owner cancels first, then publishes.
Equality with the stored value is the only reuse a contract can
observe: genuine freshness, meaning independent randomness in the
client, remains a client obligation and cannot be enforced on-chain.
This asymmetry is deliberate and is restated in Security
Considerations, because reading the on-chain check as the actual
defence is the most likely misreading of this section.

### 6. The recovery operation

Recovery is one seam path in two phases, submission and finalisation,
separated by the veto window. Together the two phases instantiate the
recovery seam of MIP-0013 section 8 and discharge its four obligations
as follows: submission verifies the recovery authorisation
(obligation a); finalisation increments the device epoch (b),
registers exactly one fresh device commitment at the new epoch and
sets the device count to one (c), and advances the authorisation
nonce and the round counter (d).

1. The recovering owner obtains `t+1` shares from guardians over the
   off-ledger transport (section 7), together with each responding
   guardian's evaluation index. If the owner's roster record
   (section 7) is lost, indices are recovered by trial assignment:
   the gate commitment gives candidate verification, so
   reconstruction is attempted over assignments of the returned
   shares to evaluation points, bounded by the number of injective
   assignments of `t+1` shares onto `n` points. This bound grows
   quickly with the roster; the parameter guidance in Security
   Considerations keeps it tractable for supported rosters.
2. Reconstruction of `s` from the shares, their indices, and the
   on-chain `phi`. The commitment provides candidate verification
   only: a lying guardian is detected as a failed reconstruction,
   not identified, and the recovery procedure on failure is retry
   over other `t+1`-subsets (see Testing).
3. Immediately upon reconstruction the wallet MUST derive everything
   the remainder of the procedure needs, and MUST then discard `s`:
   it decrypts the wrap to obtain `vk` and persists `vk` into the
   recovering wallet's private state, samples the successor recovery
   secret and derives its commitment `P' = s' * G`, and signs the
   submission challenge (step 4) with `s`. Nothing after this step
   requires `s`, so `s` never lives across the veto window (REC-6).
   The signature over the submission challenge, not `s` itself, is
   what the recover circuit consumes, so a wallet MAY discard `s`
   before the proof is produced and MAY delegate the proof to a
   third-party prover (REC-11).
4. **Submission**: the recover circuit proves knowledge of `s` by
   verifying a signature under the stored commitment `P = s * G`, not
   by recomputing a hash preimage of `s`. This is the property that
   makes the gate delegation-safe (REC-11): `s` is a takeover
   credential, and a preimage gate would have to witness it into the
   proof, disclosing it to any delegated prover; a signature gate
   discloses only a single-use signature. The circuit takes the
   successor device key `Q`, the successor recovery commitment
   `P' = s' * G`, the device epoch, the authorisation nonce, and a
   finalisation-time bound as public inputs, and takes two signatures
   as witnesses:
   - a signature by `s` over the submission challenge, verified
     against the stored `P`. The challenge is domain-separated under
     `midnight:account:recovery:submit:v1` and MUST bind the account
     address, the stored key `P`, both signatures' nonce points, the
     successor key `Q`, the successor commitment `P'`, the post-bump
     device epoch, the current authorisation nonce, and the
     finalisation-time bound, so that the signature can neither be
     replayed to enrol a different successor nor resubmitted after a
     cancel (a cancel is seam-gated and advances the nonce). Binding
     the time bound is what makes the submission exactly what the
     recovering party authorised: without it, a delegated prover
     could set the bound decades ahead, and in the total-loss case,
     where no device can cancel, the pending slot would then be held
     indefinitely by the prover.
   - a signature by the successor key `Q` over the same challenge,
     verified against `Q`. This is a proof of possession that reveals
     no successor private material, so the successor scalar, like `s`,
     never enters the proof.

   The challenge preimage is normative, so that independent signers
   and contracts interoperate. With `DST = persistentHash(pad(64,
   "midnight:account:recovery:submit:v1"))`, the challenge is

   ```
   persistentHash<[Bytes<32>, ContractAddress, JubjubPoint, JubjubPoint,
                   JubjubPoint, JubjubPoint, JubjubPoint,
                   Uint<32>, Uint<64>, Uint<64>, Uint<64>]>(
     [DST, kernel.self(), P, R_P, Q, R_Q, P', post_epoch, auth_nonce,
      now_upper, grind_nonce])
   ```

   where `R_P` and `R_Q` are the two signatures' nonce points, in that
   order, `post_epoch` is the device epoch plus one, `auth_nonce` is
   the current authorisation nonce, and `now_upper` is the
   finalisation-time bound (step 5). The grinding rule of MIP-0013
   section 5.2 applies: `grind_nonce` is incremented until the hash
   reads below the JubJub subgroup order, and one grind serves both
   signatures, because both are over the same challenge. Both nonce
   points are fixed before the grind. The reference implementation
   exports this hash as a pure circuit so that a wallet computes it
   with the contract's own arithmetic, and pins a test vector (see
   Testing).

   The gate MUST check, before any signature is examined, that the
   stored `P`, the successor key `Q`, and the successor commitment
   `P'` are each neither the curve identity nor of small order, and
   MUST reject `P'` equal to the stored `P` (the section 5 reuse
   rule, applied on the one other path that writes the commitment).
   Checking the stored `P` at the gate is what makes a weak birth key
   harmless (Backwards Compatibility): such an account is
   unrecoverable, not forgeable.

   The successor key `Q` MUST be validated, because this is the one
   enrolment path with no active device to answer for the key. The
   circuit asserts that `Q` is not the curve identity and that `[8]Q`
   is not the identity (a cofactor check rejecting the small-order
   points), and the co-signature establishes possession. What each
   check buys depends on the proving platform, and an implementation
   MUST know which case it is in. On the reference toolchain the proof
   system constrains every JubJub point that enters a circuit to the
   prime-order subgroup when it assigns it (the point assignment
   witnesses the cofactor root under the curve equation and multiplies
   by the cofactor in-circuit), so a point outside the subgroup is
   unprovable and the client runtime refuses to construct one; the
   identity is a subgroup point and passes that assignment, so the
   identity assertion is the load-bearing in-circuit check and the
   cofactor check is defence in depth. On a platform that assigns
   points as bare coordinates, the cofactor check is what rejects the
   small-order points, and full prime-order membership (the `[r] Q =
   O` check for the subgroup order `r`) is not expressible by the
   circuit author because `r` exceeds the embedded scalar bound; such
   a platform MUST add the membership check itself. In both cases a
   point the circuit only stores (a published `P`, a successor `P'`)
   MUST pass through at least one curve operation before it is
   written, because that operation is what subjects it to the
   platform's constraint. [CRYPTO-MEMO Q7] is asked to confirm that
   the platform guarantee, the identity assertion, and the
   co-signature suffice on the reference toolchain, and to state what
   a platform without the guarantee must add. Unlike a gate that
   derives the key from a witnessed scalar, the signature gate admits
   a threshold committee's joint key as the successor directly, since
   a committee can co-sign the challenge; the cost is an interactive
   co-signing ceremony at recovery time.
   Following MIP-0013, the gate is a dedicated circuit per successor
   device key scheme rather than an in-circuit conditional over
   schemes; the reference arm is Schnorr over JubJub, reusing the
   seam's own signature verification. Guardian key profiles
   (section 3) do not multiply circuits, because the guardian secret
   never enters a proof. Submission records the pending recovery
   (section 8). A submission arriving while another recovery is
   pending MUST be rejected: the pending slot is cleared only by a
   cancel or by finalisation, so a submission can neither silently
   reset a running window nor displace a competitor's pending attempt.
5. **Veto window**: finalisation MUST NOT occur until a block-time
   bound has elapsed from submission, during which an enrolled device
   MAY cancel the pending recovery. In the total-loss case there is,
   by hypothesis, no enrolled device: the window is then a delay and
   an alarm rather than a veto, and the trust root of the total-loss
   path is the guardian quorum itself (see Security Considerations).
   The window uses the standard-library block-time comparators over
   the kernel block-time predicates. Two properties of those
   comparators are load-bearing and MUST be accounted for by an
   implementation:
   - they compare wall-clock seconds, and there is no block-height
     primitive to use instead, so the window is a wall-clock window;
   - they do not incorporate the block-timestamp error bound, so a
     specification or deployment MUST state the timestamp tolerance it
     assumes, and MUST choose a window materially longer than that
     tolerance.

   Duration is a deployment parameter. It MUST be long enough that an
   owner in possession of an enrolled device can reasonably observe the
   pending recovery and act, and wallets SHOULD default to a window
   measured in days rather than hours. A cancel MUST invalidate the
   pending recovery immediately and MUST NOT itself be subject to the
   window. A cancel (`recover_cancel`) is a seam-gated operation
   available to any enrolled device, not only to the device that most
   recently authorised; it carries its own per-circuit tag (section
   2), advances the authorisation nonce and the round counter, clears
   the pending record, and leaves the device epoch unchanged. A cancel
   MUST NOT assert anything about the pending record's content: it
   must always be able to clear whatever a submission wrote.
6. **Finalisation**: once the window has elapsed, a finalising call
   (`recover_finalise`) enacts the pending record. It re-checks the
   successor recovery commitment for the identity and small order
   (defence in depth over the check at submission), enrols the
   successor device key at the new epoch, increments the device
   epoch, sets the device count to one, advances the authorisation
   nonce and the round counter, rotates the stored commitment to the
   successor recovery commitment, clears the published `phi`, and
   clears the pending record. Finalisation requires no authorisation beyond the elapsed
   window and the pending record itself: the authorisation was
   verified at submission, and the pending record fixes everything
   finalisation does, so a permissionless finalising call can enact
   only what the submitting party already proved. The epoch bump
   invalidates every previously enrolled device in a single step.
7. The wallet MUST retain `vk` (recovered at step 3) in the successor
   device's private state, and SHOULD run a fresh session promptly
   after finalisation. Until it does, the account has no recovery
   backup: the recovered secret has been rotated and the published
   `phi` cleared, so `vk` and control both rest on the single
   successor device. This is the strictest exposure in the protocol's
   lifecycle, because closing it requires a full-roster session
   (section 5) rather than a `t+1` quorum, and wallets MUST warn for
   as long as it lasts.

### 7. Guardian transport

Share requests, share responses, and liveness attestations MUST NOT be
written to the ledger. Guardian identities and the guardian graph
therefore never appear on-chain.

The transport MUST provide confidentiality and authentication of both
endpoints. A share response MUST NOT be sent to a party that has not
been authenticated as the owner of the named account, and the
specification of that authentication is a wallet responsibility rather
than a protocol one, for the same reason it is out of protocol in
comparable designs: it is a human-recognition problem, not a
cryptographic one.

Message shapes (fields, not encodings; encodings are a profile
concern):

| Message | Fields | Direction |
|---|---|---|
| `ShareRequest` | session identifier, account identifier, guardian evaluation index, requester authentication material | owner to guardian |
| `ShareResponse` | session identifier, guardian evaluation index, `sigma_ij` | guardian to owner |
| `LivenessAttestation` | session identifier, proof of ability to recompute `sigma_ij` without revealing it | guardian to owner |

The evaluation index travels in both directions because reconstruction
needs `(index, share)` pairs and the share derivation of section 4 is
index-free: a share carried without its index is unusable except by
the trial assignment of section 6. Wallets MUST maintain a durable
roster record: the roster size `n`, the threshold parameter `t`, each
guardian's evaluation index, and the owner's means of reaching them.
The record is not secret and SHOULD be replicated across the owner's
devices and backups; its loss does not defeat recovery but degrades
it to trial assignment. Publishing the record, or the pair `(n, t)`,
on-chain is rejected: it would widen the accepted leakage set of
Security Considerations from the redundancy parameter to the roster
size and threshold themselves.

Wallets MUST implement a periodic liveness attestation; its cadence is
a SHOULD. The attestation exists because the guardian-side failure in a
zero-storage design is silent: a guardian whose credential is gone
looks exactly like a guardian who has not been asked recently, and
without attestation an owner discovers an under-strength roster only at
the moment of recovery, which is the one moment at which it cannot be
repaired.

### 8. On-chain structures

The artefact set is three objects plus a session identifier, and the
contract additionally holds a pending-recovery record between
submission and finalisation:

| Object | Type | Notes |
|---|---|---|
| recovery commitment | curve point | `P = s * G`, the public key the recover gate verifies a signature under (section 6) |
| `phi` | bounded vector of field elements | public shares; length per the formula below |
| viewing-key wrap | authenticated ciphertext | `vk` under the wrap key of section 2 |
| session identifier | 32 bytes | distinct per session (REC-4) |
| pending recovery | record, present at most once | the successor device key's derived entry at the post-bump epoch, the successor recovery commitment `P'`, and the earliest finalisation time; written by submission, cleared by cancel or finalisation (section 6) |

For a roster of `n` guardians with reconstruction threshold `t+1`,
the published vector's length is `n - t`. The underlying scheme
counts the dealer as a party, so its `n' - t - 1` with `n' = n + 1`
is the same quantity. Two worked examples: three guardians at
threshold two publish two elements; five guardians at threshold three
publish three.

`phi` is a bounded vector because contract state must be statically
bounded; the bound is a profile parameter and caps the roster size
that a given deployment supports, and an implementation MUST document
which `(n, t)` pairs its bound admits. Unused slots MUST be zero and
MUST be gated by an explicit length field, so that a shorter roster
cannot be read as a longer one with zero-valued shares.

The wrap MUST be an authenticated encryption. Requirements on the AEAD
(key commitment, nonce policy) and on the separation of the wrap key
from the gate key are the subject of [CRYPTO-MEMO Q5].

Concrete figures from the reference implementation: the wrap
container is sixty-four bytes, the vector bound is four slots
(admitting every roster with `n - t <= 4`), and the artefact set
deploys and refreshes within current block limits, though the
contract as a whole does not deploy in one transaction (see the
deployment note in Implementation). Proof-cost measurements remain
future work.

### 9. Invariants

- **REC-1 (Safety)**: only a party reconstructing the current `s` can
  pass the recover gate.
- **REC-2 (Liveness)**: with `t+1` honest reachable guardians, the
  on-chain artefact set, and either the roster record or a tractable
  trial assignment (section 6), the owner can reconstruct `s` and
  complete recovery, provided no hostile enrolled device cancels
  within the window (REC-9 and Security Considerations).
- **REC-3 (No guardian storage)**: guardians persist no per-account
  state; shares are recomputable on demand.
- **REC-4 (Session freshness)**: every published artefact set uses a
  previously unused session identifier and a freshly sampled secret.
- **REC-5 (Removal effectiveness)**: after a session excluding a
  guardian, the excluded guardian contributes nothing to recovery of
  the current secret.
- **REC-6 (Ephemeral secret)**: `s` is never at rest; it exists during
  session setup and recovery only.
- **REC-7 (Viewing restoration)**: successful recovery yields the
  current `vk`, and with it the MIP-0012 inbox walk that rebuilds the
  coin store.
- **REC-8 (Recovered account, recovered assets)**: after finalisation
  the new device controls the account with all custodied assets and
  their visibility intact. REC-8 presupposes the successor-key
  validation of section 6.
- **REC-9 (Veto)**: a recovery cannot finalise while an enrolled device
  remains able to cancel within the window.
- **REC-10 (Backup continuity)**: `vk` survives recovery: it is
  persisted to the recovering wallet's private state before `s` is
  discarded, and remains held by the successor device until a fresh
  session republishes a wrap.
- **REC-11 (Delegation safety)**: no recovery operation requires the
  prover to learn `s` or any successor private material. The recover
  gate consumes signatures under `s` and under the successor key, not
  the scalars themselves, so a delegated prover can at most produce
  the exact submission the recovering party authorised, and cannot
  reconstruct a takeover credential from what it is given.

REC-1 and REC-2 refine the user-level safety and liveness properties
tracked by the formal specification of the architecture; that work is
tracked separately (see Implementation Plan). REC-4 and REC-5 rest on
the multi-session security statement commissioned as
[CRYPTO-MEMO Q1] and [CRYPTO-MEMO Q2]: an implementer building
against this section should know that removal effectiveness is
established for the unchanged-roster reuse case and commissioned, not
yet proven, for the general multi-session case.

### 10. Versioning

The artefact set carries a version tag. A conforming reader MUST reject
an artefact set whose version it does not implement rather than attempt
a best-effort parse, because a misparsed `phi` yields a reconstruction
failure that is indistinguishable from an under-strength roster.

Guardian key profiles (section 3) are a registry: a profile fixes the
guardian secret derivation, the hash for section 4, and the field
mapping, and a profile registration MUST state the minimum entropy of
the guardian secret it produces, because the offline-search argument
in Security Considerations is conditional on every registered profile
meeting the full-entropy expectation stated there. Adding a profile
is a registry addition and does not require a new version of the
artefact set; guardian profiles also do not multiply recover
circuits, because the guardian secret never enters a proof
(section 6). An account's roster MAY mix profiles, because a share is
a field element regardless of how the guardian secret was obtained.

A future traceable arm, in which a leaked share can be attributed to
the guardian that leaked it, is anticipated as a v2 artefact set. It
changes the artefact layout rather than the roles, the transport, or
the invariants, so v1 and v2 accounts can coexist and an account can
migrate by running one session under the new version. Parameter
guidance that would keep v2 reachable without re-parameterising v1 is
[CRYPTO-MEMO Q6].

## Rationale

### Choice of scheme

The mechanism is bottom-up secret sharing as introduced by the
ANARKey work (References), in which guardian shares are derived from
guardian-held secrets and the dealer publishes a public correction
vector, rather than a design in which the dealer distributes shares
that guardians must keep.

| Design | Guardian storage | Guardian graph | On-chain artefact | External validation | Why not selected |
|---|---|---|---|---|---|
| **Bottom-up secret sharing (selected)** | none | off-chain | compact public vector | peer-reviewed 2026; no audit-grade implementation | selected |
| Helper-stored shares (DeRec) | per-owner state, with challenge-response upkeep | fully off-chain | none | alliance specification, alpha implementation | every helper acquires a durable backup problem; see below |
| Smart-account guardian modules | none | fully public on-chain | guardian set | most mature by far: multiple audits, formal verification, large deployed user base | publishes the guardian graph, which is the privacy property this account family exists to protect |
| Proof-of-email guardians | none | partially public | proof artefacts | audited, deployed on mainnet wallets | inherits the guardian graph exposure and an external trust root |
| Threshold-key resharing | key share, durable | off-chain, deniable | none | standardised signing, non-standard distributed key generation and resharing | heaviest guardian burden, and refresh is the empirically fragile step |
| Provider-held encrypted backup | none | none | none | very large deployed scale | reintroduces a party that can restore, and therefore take, the account |

The selection follows from the setting rather than from maturity: it is
the only design in the table that combines zero guardian storage,
off-chain guardian identities, and a compact public artefact. That
combination is what allows a guardian to be an ordinary person with an
ordinary phone who is never asked to keep anything, and whose
participation is not published.

Maturity is the honest cost of that choice, and this proposal does not
minimise it. The selected scheme is peer-reviewed but thinly cited and
has no audit-grade implementation, whereas the guardian-module design
it displaces has audits, formal verification, and users. The mitigations
are the commissioned review that Path to Active makes a gate, the
conformance suite, and a specification written to be implementable
without the reference library.

### Relationship to helper-stored designs

Helper-stored recovery is the ecosystem's default mental model, and an
account holder who has met recovery before has probably met it in that
form. The difference is narrow and worth stating plainly rather than
leaving implicit: both designs distribute trust across a threshold of
helpers, and both keep the helper graph off-chain. They differ in what
a helper is asked to do between ceremonies. A helper that stores state
must keep it, prove periodically that it still has it, and re-receive
it whenever membership changes. A guardian that stores nothing is asked
only to be reachable and to re-derive on demand. This proposal takes
the second position because durable state held by many
loosely-motivated parties is, on the available evidence, the component
that decays.

The two are not exclusive at the ecosystem level: an account can have
guardians under this proposal and an independent encrypted backup
elsewhere, and nothing here forbids that.

### Wrap, not derive

`vk` is independent of `s` because their lifecycles differ. The viewing
key exists from the account's first shielded coin; guardians are
optional and typically arrive later. Deriving `vk` from `s` would make
every roster change a re-keying of the account's entire shielded view,
with a re-encryption of shielded state behind it. Wrapping instead
leaves `vk` stable and makes a roster change a cheap operation, at the
cost of the historical-artefact residual documented in Security
Considerations.

### Ephemeral `s`

`s` passes the recover gate and is therefore a takeover credential
equivalent to the account itself. Holding it at rest would convert any
device compromise into a permanent, undetectable account takeover, and
would do so in exactly the population least able to notice. No
operation requires an old `s`: every artefact write mints a fresh
secret in a fresh session, so the value never needs to survive its
ceremony.

### Signature gate, not preimage witness

An earlier arm of this design proved knowledge of `s` by witnessing it
into the recover circuit and recomputing a hash commitment. It is the
simpler construction, and it verifies on today's toolchain with no new
primitive, but it fails the delegation test. Midnight proving delegates
by handing the prover the full witness, so a preimage gate hands `s` to
whoever produces the proof. That is the wrong disclosure for exactly
this operation: `s` is a takeover credential, and the party most likely
to need a delegated prover is a recovering owner on a fresh, weak
device with no proving resources of its own. The gate is therefore
specified over a signature: the commitment is the public key
`P = s * G`, the circuit verifies a signature under `P`, and `s` stays
on the recovering wallet. Successor possession moves from
derive-in-circuit to a co-signature by the successor key for the same
reason, so the successor scalar is not disclosed either. The cost is a
public key that is a discrete-logarithm commitment rather than a hash
commitment (Security Considerations shows the offline-search hardness
is unchanged), and a successor-key validation that the circuit no
longer guarantees by construction: on the reference toolchain the
proof system constrains every point it assigns to the prime-order
subgroup, so what is left to the circuit is the identity assertion,
with a cofactor check as defence in depth and the co-signature for
possession (section 6 step 4 states the two platform cases). The gate is
specified scheme-agnostically, with per-scheme circuits as in the
signature-schemes work; the reference arm is Schnorr over JubJub,
which reuses the seam's existing verification and re-proves on the
current toolchain, and an ECDSA arm registers when its in-circuit
verification surface ships. This invariant is REC-11.

### Guardian is a credential

The protocol binds shares to the credential that computed them, so the
credential is the natural unit of identity. Presenting several
credentials held by one person as one guardian is a wallet concern.
Security parameters are the place where the distinction bites, and the
guidance there counts people, because a threshold of credentials held
by one person is a threshold of one person.

### Padding considered and rejected

The artefact length reveals only the redundancy parameter `n - t`.
The corruption cost `t+1` follows only for an observer who learns the
roster size by other means, and neither the targets nor the channels
appear on-ledger. Padding to fixed capacities would add trial
reconstruction at recovery time and protocol complexity
disproportionate to what it conceals. Implementations MAY pad;
nothing in this specification depends on it.

### Why the freshness rule is client-side

A contract can compare a submitted value with the value it stores, and
nothing more. It cannot observe that a client sampled independently,
which is the property that actually matters, and it cannot see the
history of values it no longer holds. Specifying the on-chain check as
a tripwire rather than as the defence is therefore not a weakening: it
is an accurate description of what the enforcement point can do, and
stating it accurately is what allows an implementer to see that the
client obligation is load-bearing.

## Path to Active

### Acceptance Criteria

- [ ] MIP number assigned by an editor; listed alongside MIP-0012 and
      MIP-0013 in MPS-0018's header as a proposed solution.
- [ ] Cryptographic review memo (commissioned; response pending)
      covering the multi-session security statement, the
      public-share-update verdict, the sampling, mapping, commitment,
      and AEAD constructions, and the delegation-safe gate's
      successor-key validation ([CRYPTO-MEMO Q7]: the point commitment,
      and whether the platform's subgroup constraint, the identity
      assertion, and the co-signature suffice for a public-key input),
      with its conclusions folded into sections 2,
      4, 5, 6, 8, and 10, Security Considerations, and Implementation.
- [ ] Domain-separation tags (`midnight:account:recovery:*:v1`)
      registered under the MPS-0027 registry once it ratifies.
- [x] Reference implementation of the session and recovery lifecycle
      in the custody reference contract, with conformance suites
      passing on a devnet-matching network. [EXP: met: session
      publish, on-chain freshness rejection, reconstruction from
      chain data, wrap round-trip, the signature gate, veto window,
      cancel, finalisation, and epoch-bump revocation ran end to end
      on a ledger 9 local network; evidence recorded with the
      reference implementation. See Implementation.]
- [x] Signature (delegation-safe) recover gate of section 6
      implemented on the reference contract and re-proven on a local
      network. [EXP: met: the gate takes the stored public key, a
      signature under it, and the successor co-signature, with the
      identity and cofactor checks and the reuse and zero-identifier
      guards; the full behaviour matrix, including every refusal of
      section 6 step 4 and the delegation-safety and finalisation
      rows of Testing, ran in the runtime simulator, and the lifecycle
      ran end to end on a ledger 9 local network through real proofs
      and a real veto window; the evidence record accompanies the
      reference implementation. See Implementation.]
- [x] Veto window and cancel path implemented and exercised end to end
      on a local network. [EXP: met: the window held against an
      early finalisation on real block time, a cancel cleared the
      pending record with the epoch unchanged, and finalisation
      succeeded once the window elapsed]
- [x] Viewing-key wrap published, recovered, and round-tripped through
      the artefact set. [EXP: met: the wrap read back from the
      ledger decrypted under the reconstructed secret on a local
      network]
- [ ] Liveness attestation and transport messages implemented by at
      least one wallet provider.
- [ ] Community review period completed.

### Implementation Plan

The specification stacks on the MIP-0012 and MIP-0013 reference
implementation, which now carries the whole contract tranche (see
Implementation). Remaining work divides into three independent
tranches that can proceed in parallel:

1. **Contract**: landed and evidenced (the session operation and the
   two-phase signature gate on the MIP-0013 seam, on both of the
   reference contract's authorisation arms, the pending record, veto
   window, and cancel path, the wrap cell, and the version tag, with
   the simulator matrix and the on-node conformance suite passing).
2. **Cryptographic**: the commissioned memo, and the normative
   constructions it fixes in sections 2 and 4.
3. **Wallet**: transport messages, liveness attestation, the roster
   record, and the guardian-facing ceremony, with at least one wallet
   provider.

The formal specification of the architecture refines REC-1 and REC-2;
that work is tracked separately and is not a gate on this document.

## Backwards Compatibility Assessment

At the protocol level the proposal is additive: it requires no
ledger, consensus, or node change and no hard fork. At the contract
level it is not additive for deployed accounts: the artefact set and
the pending record are new contract ledger cells, a deployed
contract's ledger state schema is fixed at deploy even where its
circuits are evolvable in place, and recovery at birth is a
constructor-signature change. An account deployed before this
standard therefore adopts it by redeploying under a recovery-carrying
version and migrating its custodied assets; an account deployed under
a recovery-carrying version adopts by running a first session; and an
account that never runs a session is unaffected and behaves exactly
as it does today. Recovery at birth applies to new deployments, which
accept an initial artefact set at deployment. The birth commitment is
not validated by the constructor but by the gate: a deploying client
that publishes the identity or a small-order key produces an account
that cannot be recovered through that key, never one that anyone can
recover (section 6 step 4).

The veto window relies on block-time comparators that are present in
current toolchains, including the line the reference implementation
pins, so the window introduces no new platform dependency. Normative
text cites primitives with an applicability note for ledger 9
toolchains rather than pinning a toolchain version, because the
release line has moved and a version pin would date the document.

Contract state survives the ledger 9 hard fork, which migrates a
running chain rather than requiring a fresh one, so an account that
publishes an artefact set today does not need a redeploy story to keep
it. Implementations SHOULD NOT, however, assume that auxiliary ledger
subsystem state survives a fork on the same terms: the same upgrade
rebuilds dust-generation state, which is precedent that not every
subsystem is carried across untouched. Nothing in the artefact set
depends on such state, and this note exists so that a future profile
does not introduce such a dependency inadvertently.

Dormant accounts: an account whose artefact set was published under an
earlier profile or version remains recoverable under that profile for
as long as the profile is registered. Retiring a profile therefore
requires a deprecation path and MUST NOT be done by removal alone,
because the population that most needs recovery is exactly the
population that has not been running sessions.

## Security Considerations

- **Session-identifier reuse**: reuse collapses the independence of
  the ceremonies it correlates. With an unchanged roster the
  difference of the two secrets becomes computable from public data
  alone, so one quorum opens both; a correlated pair costs the
  adversary at least one corruption fewer in general; and once the
  roster reaches `2t+2` under a reused identifier, the public vectors
  alone determine the secrets with no corruption at all. Adding a
  sixth guardian to a five-guardian, threshold-three roster without a
  fresh identifier hands the secret to a passive chain observer, and
  adding a guardian is exactly the routine variant of section 5. The
  freshness requirement (REC-4) is the actual defence; the on-chain
  inequality check is a tripwire only, for the reason given in
  Rationale. The reuse algebra for the unchanged-roster case is
  established; the positive statement for the distinct-identifier
  regime is commissioned as [CRYPTO-MEMO Q1] and [CRYPTO-MEMO Q2].
- **The quorum is the total-loss trust root**: a colluding `t+1`
  quorum reconstructs the current `s` and passes the gate
  legitimately, and in total loss there is no enrolled device to
  cancel, so the veto window is a delay and an alarm rather than a
  veto. The guardians know the owner is in total loss precisely
  because the owner asked them for shares. The threshold MUST
  therefore be chosen as the number of distinct people the owner is
  prepared to trust jointly with the account, under the
  people-counting rule below.
- **A hostile enrolled device**: an adversary holding an active
  device can cancel any pending recovery for as long as the device
  remains enrolled, and REC-2 is conditional on no such device
  existing. This grants the adversary nothing the device does not
  already confer: under the one-of-n authorisation surface an active
  device already controls the account and its assets outright, so
  the veto is not an escalation, and bounding it (a cancel budget, a
  shrinking window) would convert compromise of `s` plus patience
  into guaranteed takeover, which is strictly worse. The remedy for
  a stolen device is removal from a surviving device; where none
  survives, the account's exposure is the theft itself, not the
  veto.
- **Historical artefacts**: chain history is permanent. A past `t+1`
  quorum retains the ability to decrypt the wrap of its era; with a
  stable `vk` that capability reaches the present, which is why
  for-cause removal SHOULD churn. Retroactive viewing of pre-rotation
  history is unavoidable in any design in which recovery restores
  viewing; this residual is accepted and stated rather than mitigated.
- **Leakage inventory** (accepted, documented): existence of a
  recovery configuration; the public-share count; session timing and
  frequency; distinguishability of routine sessions from for-cause
  sessions with churn. Guardian identities, channels, and the guardian
  graph never appear on-chain (section 7 transport MUST).
- **Coercion**: the on-chain data yields the redundancy parameter
  `n - t`, not targets. The corruption cost follows only for an
  adversary who learns the roster size by other means, and guardian
  identities and channels stay off-ledger either way.
- **Offline verification oracles**: the commitment and the wrap allow
  candidate verification of guesses at `s`; a single missing share
  carries full field entropy, which is what makes offline search
  infeasible, and every registered profile MUST meet that entropy
  expectation (section 10). The commitment is the public key
  `P = s * G`, so a guess is tested by one scalar multiplication and a
  point comparison, exactly as a hash commitment is tested by one hash
  and a comparison: the oracle and its cost are unchanged by the move
  from a hash commitment to a curve point, and the hardness rests on
  the entropy of `s` either way. Publishing `P` additionally assumes
  the discrete-logarithm hardness of the curve for the hiding of `s`,
  which the full entropy of `s` already required for the wrap key. The guardian secret `sk_j` persists
  across sessions under Profile A, so every session's artefacts test
  the same guess against it; the search-hardness argument must
  therefore hold over `sk_j` as well as over `s`. [CRYPTO-MEMO Q5] is
  commissioned to confirm this argument and to fix the AEAD
  requirements it depends on; a refutation would change the wrap
  construction, not the sharing.
- **Delegated proving and successor-key validation**: the recover
  gate is delegation-safe by REC-11: it consumes a signature under `s`
  and a co-signature under the successor key, never the scalars, so a
  hostile prover learns only two single-use signatures and the public
  inputs, and can at most submit the exact recovery the recovering
  party authorised. What the prover can do with the public inputs is
  also bounded: every value the submission fixes, including the
  finalisation-time bound, is signed, so the prover cannot stretch the
  veto window or redirect the successor. The residual is the
  successor-key validation. A gate that derived the successor key from
  a witnessed scalar would guarantee subgroup membership by
  construction but would disclose the successor scalar to the prover,
  defeating REC-11; the gate therefore takes the successor key as a
  public input. On the reference toolchain the proof system constrains
  every assigned JubJub point to the prime-order subgroup, so the
  identity assertion is the load-bearing check and the cofactor check
  is defence in depth; on a platform without that guarantee the
  cofactor check rejects the small-order points and full membership
  must be added by the platform, since the `[r] Q = O` check is not
  expressible by the circuit author (section 6 step 4). The same
  analysis applies to the ordinary device-addition path, which is why
  a platform subgroup-check primitive would benefit the whole account
  family, not this gate alone. [CRYPTO-MEMO Q7] is asked to confirm
  that the platform guarantee, the identity assertion, and the
  co-signature suffice on the reference toolchain, and to state what a
  platform without the guarantee must add.
- **Local leakage during reconstruction**: published results on
  linear-reconstruction secret sharing show that a few bits of local
  leakage per share can defeat the sharing (References). Under this
  proposal reconstruction concentrates `t+1` shares in one wallet at
  one moment, so the relevant surface is that wallet's execution
  environment rather than the guardians'; implementations SHOULD
  reconstruct in the same protected context that holds device keys.
  Whether the published bound applies to this scheme's reconstruction
  as performed here is put to the commissioned review as a follow-up
  question.
- **Threshold counted in people**: a roster of `n` credentials held by
  fewer than `n` people has a lower effective threshold than its
  parameters suggest. Wallets SHOULD encourage rosters spanning
  distinct people and SHOULD warn when several guardian credentials
  share an obvious provider or device.
- **Guardian authenticity and Sybil resistance**: establishing that a
  guardian is the person the owner believes them to be is out of scope
  of the mechanism, as it is in comparable designs. It is a wallet and
  social concern, and the mechanism's guarantees are conditional on the
  owner having chosen guardians correctly. This proposal does not claim
  otherwise, and a wallet SHOULD make the trust assumption explicit at
  enrolment rather than presenting guardianship as a purely technical
  step.
- **Cold start**: an account with no community at hand has no roster
  to publish, and is the population most likely to be lost. Profile C
  (section 3) preserves the mechanism for that population with paper
  keys as virtual guardians, at the cost of reintroducing a physical
  artefact. The trust profile differs in kind: paper keys under one
  roof are one guardian in effect, and the people-counting rule
  applies to their custodians, not to the sheets.
- **Emergency viewing-key rotation** while guardians are unreachable
  leaves recovery restoring a stale `vk` until a session completes;
  wallets MUST warn.
- **Denial of service against the window**: a party holding `s` can
  submit repeatedly, forcing an owner to cancel repeatedly. Every
  cancel advances the authorisation nonce and the gate proof binds
  it, so a cancelled proof cannot be resubmitted: each attempt costs
  the adversary a fresh proof, and each cancel costs the owner an
  authenticator ceremony and a seam-gated call. The durable remedy is
  not this exchange but a fresh session, which rotates `s` and
  invalidates the adversary's reconstruction outright. A wallet
  SHOULD surface repeated attempts and SHOULD propose that session,
  because a pattern of attempts is evidence that `s` is compromised.
  A cancelled attempt leaves the device epoch unchanged.
- **Comparable-design failure precedent**: a published vulnerability
  in a widely used threshold-signature library (CVE-2025-58359;
  References) arose specifically in share refresh, where a refresh
  performed with different parameters degraded security, and
  remediation required migrating to a fresh key. Refresh is the
  fragile operation in this class of protocol, which is why this
  proposal admits exactly one artefact-publishing operation with a
  mandatory fresh secret rather than a family of partial updates.

## Implementation

Two contract artefacts exist, and this section names them separately,
because the gap between them is what Path to Active tracks.

**The reference implementation** (`contract/` in the authoring
repository) implements MIP-0012 and MIP-0013 and carries this
specification's contract tranche: the artefact-set ledger cells (the
recovery public key as a curve-point cell) and version tag, the
seam-gated session operation with the freshness backstops, the
zero-identifier and small-order guards, the unused-slot asserts, and
the pending-recovery rejection, the signature gate of section 6 step
4 with the explicit epoch, nonce, and time-bound binding and the
successor co-signature, the seam-gated cancel, the permissionless
finalisation behind the block-time window, and the wrap cell. The
seam-gated operations exist on both of the contract's co-resident
authorisation arms (`publish_recovery_session_with_jubjub` and
`_with_k256`, `recover_cancel_with_jubjub` and `_with_k256`); the
gate and the finalisation are arm-independent. The whole lifecycle is
exercised twice: in the toolchain's runtime simulator under explicit
wall-clock control, where the behaviour matrix covers every refusal
of section 6 step 4 (identity and small-order keys, a missing
co-signature, a wrong recovery signature, each signed binding
tampered after signing, a reused successor commitment, a forgery
against an identity birth key) and the delegation-safety and
finalisation rows of Testing; and end to end on a ledger 9 local
network (node, indexer, and proof server), where it ran through real
proofs and real block time: a four-wave deploy with birth artefacts,
session publish through the seam, on-node freshness rejection,
reconstruction from chain data, the wrap round-trip, the gate
submission, an early finalisation held by the window, cancel,
resubmission, finalisation after the window, and a post-recovery
session from the successor device. The evidence record accompanies
the reference implementation.

The gate the reference contract carried before this tranche was the
possession construction of an earlier draft, which witnessed `s` and
derived the successor key in-circuit from a witnessed scalar; it was
replaced by the signature gate so that neither scalar enters a proof
(REC-11). Two toolchain observations from the port are recorded with
the reference implementation. The proving toolchain constrains every
JubJub point it assigns to the prime-order subgroup and refuses to
construct one outside it, which is the platform case section 6 step 4
describes. And key generation enables a circuit's curve gadget only
when the circuit performs a curve operation on a runtime value, so a
circuit whose only JubJub use is writing a point literal type-checks
but cannot be proven; the reference contract clears its pending key
slot with a curve multiplication by zero for that reason, and a keyed
compile is the only build that shows a contract builds.
The client side ships the second independent share-derivation
implementation (pure TypeScript over the field), the wrap container
v1, the roster record with trial-assignment fallback verified against
the stored recovery key, a version-checked reader for the artefact
set, and the gate's signing ceremony, which consumes `s` to produce
the recovery signature and holds it nowhere afterwards, so REC-6 is
exercised rather than asserted.

**The prototype** (`experiments/account-custody-prototype/` in the
authoring repository) is the evidence base. It predates this
specification and diverges from it in known ways, and it demonstrated
the mechanism end to end on a local network: a device-authorised
publish of the artefact set (commitment, typed public-share vector,
session identifier, explicit length field); the on-chain freshness
backstop of section 5, rejecting reuse of the stored session
identifier and of the stored commitment; and a recovery gate that
witnesses the recovery secret, recomputes the commitment through a
pure derivation circuit, asserts equality, bumps the device epoch,
enrols the successor device, rotates the recovery commitment, and
zeroes the vector's length field (the superseded entries remain in
public state, inert because the commitment rotates with them). The
demonstrated flow includes total loss, off-chain reconstruction from
the on-chain vector, and re-entry with a fresh device.

**Known divergences of the prototype from this specification**,
recorded so its evidence is weighed correctly: its circuits sit
behind a hash-preimage placeholder rather than the MIP-0013 signature
seam, a construction MIP-0013 rules non-conforming; it holds the
recovery secret in wallet private state, so REC-6 is unexercised; it
uses ad-hoc tags of an earlier convention over a hash that is not
stable across toolchain versions, against MIP-0012's stable-hash
requirement; it neither asserts that unused slots are zero nor
validates the successor key; and it has no veto window, no cancel
path, no wrap, no transport messages, and no version tag.

**Bound**: both implementations cap the public-share vector at four
slots, admitting every roster with `n - t <= 4`: three guardians at
threshold two and five at threshold three among them. The bound is an
implementation parameter rather than a property of the scheme, and
section 8 specifies it as a profile parameter for that reason.

**Deployment note**: with the recovery circuits the reference contract
carries thirty-six operations across its two authorisation arms and
the scoped-grant surface, and a deploy carrying all their verifier
keys exceeds the node's per-block limits and can never be included in
a block. The reference implementation deploys in waves: the deposits
and the initial device's arm with the constructor's full ledger state,
then the remaining verifier keys packed under a measured
per-maintenance-update budget into batched contract maintenance
updates (four waves in the on-node run), the six recovery keys riding
last, and the final update retiring the maintenance authority, because
a live authority can replace an asset-releasing circuit's verifier key
and is therefore a path around the authorisation seam. A retired
authority also closes the account's upgrade path, and a recovered
account cannot reopen it, so a deployment that keeps the authority for
later circuit fixes must hold it as a committee whose members are no
weaker than the device set. The block-limit constraint is a platform
finding worth reporting upstream in its own right: it binds any
contract with this many entry points, not this design specifically.

**The scheme library**: a proof-of-concept library implementing the
underlying scheme validated the approach. It is unaudited, declares
itself unsuitable for production, and has been dormant since shortly
after our adoption of it. The authoring workspace does not fork it:
the reference component of the scheme layer is a thin profile crate
that depends on the library pinned to a fixed revision and adds only
what this specification requires on top of it, namely the share
serialisation, typed session identifiers, and secret zeroisation the
upstream library lacks, held bit-exact against the client
implementation by cross-implementation vectors. This specification
remains implementable from its own text and does not normatively
depend on either the library or the profile crate: the papers cited
in References are the normative basis for the scheme, and the code is
evidence that the construction works, not a component of the
standard. Capabilities present in the library but unused here (a
traceable variant, a signature-derived guardian secret, and
public-share update without re-share) informed section 3 and section
10, and the last of them is precisely what [CRYPTO-MEMO Q3] must rule
on before it could be admitted.

## Testing

A conforming implementation SHOULD provide:

- **Authorisation**: an artefact set submitted without a valid device
  signature is rejected with no state change; a cancelled gate proof
  resubmitted verbatim is rejected, because the authorisation nonce
  has advanced.
- **Session freshness**: rejection of a repeated session identifier and
  of a repeated recovery commitment; acceptance of a fresh pair;
  rejection of an all-zero identifier from a broken generator.
- **Recovery lifecycle**: publish, reconstruct from exactly `t+1`
  shares, pass the gate, finalise, and confirm the epoch bump
  invalidates prior devices.
- **Threshold behaviour**: reconstruction fails with `t` shares and
  succeeds with `t+1`; an excluded guardian's share from a prior
  session does not contribute (REC-5); a reconstruction attempted
  with one wrong share fails the commitment check and succeeds after
  substituting the correct share.
- **Successor validation**: a submission whose successor key is the
  curve identity is rejected; a submission whose successor key or
  successor commitment is a small-order point is refused (on the
  reference toolchain the platform refuses to construct the point; on
  a platform without that guarantee the cofactor check rejects it, and
  the test records which); a submission carrying a valid successor
  key but no valid co-signature under it is rejected, so possession is
  actually tested; a submission whose recovery signature does not
  verify under the stored public key is rejected; a submission whose
  successor commitment equals the stored one is rejected; and a
  forgery against a stored key that is the identity (a weak birth key)
  is refused at the gate before any signature is examined. A gate arm
  that instead derives the successor key from a witnessed scalar
  needs the zero-scalar case in place of the identity and cofactor
  cases.
- **Delegation safety**: the submission's circuit arguments contain
  neither the recovery secret nor the successor scalar; altering any
  signed binding after signing (the finalisation-time bound, the
  successor key, the successor commitment) is rejected with no state
  change, whether at the canonical cast of the challenge or at the
  verification; a cancelled submission replayed verbatim is rejected,
  and so is a fresh signature over the pre-cancel nonce.
- **Finalisation effects**: finalisation advances the authorisation
  nonce and the round counter by one each, as well as bumping the
  epoch, setting the device count to one, rotating the commitment,
  and clearing the vector and the pending record.
- **Veto and cancel**: finalisation blocked before the window elapses;
  cancel invalidates a pending recovery; cancel is itself not subject
  to the window; a cancelled attempt leaves the epoch unchanged; a
  second submission while one is pending is rejected; a session
  submitted while a recovery is pending is rejected.
- **Wrap round-trip**: `vk` published under a session, recovered
  through that session's secret, and shown to decrypt current shielded
  state (REC-7).
- **Malformed artefacts**: non-zero unused slots rejected; a length
  field of zero and a length field beyond the slot bound rejected;
  an all-zero session identifier rejected after a real session has
  been published, not only against the constructor sentinel; an
  artefact set of an unknown version refused by the reader rather
  than best-effort parsed.
- **Liveness attestation**: a guardian able to recompute attests
  successfully; a guardian whose credential is gone fails, and the
  failure is visible to the owner before recovery is needed.
- **Cross-implementation vectors**: share-derivation test vectors, so
  that two independent wallets derive identical shares for the same
  session, account, and guardian secret; for Profile B, vectors
  covering the deterministic-signature layer as well, so that two
  independent signers derive identical guardian secrets for the same
  signer key and session binding; and a recover-gate challenge
  vector. The reference implementation pins, for the account address
  `0x22` repeated, `P = 1·G`, `R_P = 2·G`, `Q = 3·G`, `R_Q = 4·G`,
  `P' = 5·G`, post-bump epoch 1, authorisation nonce 7, time bound
  1800000000, and grind nonce 0, the challenge
  `16ebd49d1f2b69139f06c46587eacd4f0bbdb7fb203113503143f624b65afae2`.

## References

- [MPS-0018](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md):
  Multi-key Account Custody for Midnight-Native Assets (the parent
  problem statement, whose Recommended MIPs section reserves the
  recovery-paths slot this proposal fills).
- [MIP-0012](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0012-native-asset-custody.md):
  Contract Custody of Midnight-Native Assets (the asset surface, the
  encryption-key lifecycle behind `vk`, and the inbox walk that
  restores visibility).
- [MIP-0013](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0013-account-authorisation.md):
  Multi-key Account Authorisation for Custody Contracts (the seam,
  the device entry model, the epoch mechanism, and the section 8
  recovery seam this proposal instantiates).
- [MIP-0015](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0015-wallet-derived-deterministic-secrets.md):
  Wallet-derived deterministic secrets (the seed-anchored branch of
  the durability fork; see Motivation).
- [MPS-0035](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0035-shielded-spend-key-exposure.md):
  Shielded Spend Authorization Requires Exposing the Spend Key (the
  non-extractable-key constraint the recovery gate respects).
- [MPS-0027](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0027-domain-separation.md):
  Domain Separation for Midnight Hash Constructions (the registry for
  the section 2 tags).
- ANARKey: A New Approach to (Socially) Recover Keys
  ([IACR ePrint 2025/551](https://eprint.iacr.org/2025/551); IEEE
  EuroS&P 2026, DOI 10.1109/eurosp68448.2026.00037): the bottom-up
  secret sharing construction, its single-session model (section 4),
  and the share-derivation remark (6.1) this proposal builds its
  multi-session rules on.
- Traceable bottom-up secret sharing
  ([IACR ePrint 2025/2089](https://eprint.iacr.org/2025/2089);
  INDOCRYPT 2025, DOI 10.1007/978-3-032-13301-4_18): the anticipated
  v2 artefact arm of section 10.
- Leakage resilience of linear-reconstruction secret sharing
  ([IACR ePrint 2026/833](https://eprint.iacr.org/2026/833)): the
  local-leakage bound behind the reconstruction-environment guidance
  in Security Considerations.
- [CVE-2025-58359](https://nvd.nist.gov/vuln/detail/CVE-2025-58359):
  share-refresh parameter flaw in a threshold-signature library (the
  production instance of the refresh hazard class; see Security
  Considerations).
- [W3C Web Authentication](https://www.w3.org/TR/webauthn-3/), PRF
  extension (Profile A's guardian secret).
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119): requirement
  levels; [RFC 6979](https://www.rfc-editor.org/rfc/rfc6979):
  deterministic ECDSA (Profile B);
  [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki):
  Schnorr signatures for secp256k1 (Profile B, auxiliary randomness
  pinned).
- [DeRec Alliance](https://derecalliance.org/) protocol
  specification: the helper-stored design of the Rationale
  comparison.
- Midnight Passport workspace: the account-custody prototype
  (`experiments/account-custody-prototype/`), the reference
  implementation (`contract/`), and the scheme-library profile crate;
  to be linked at their public locations on submission.

## Acknowledgements

The IOG Advanced Research and Creativity department and the Midnight
Foundation.

## Copyright Waiver

This document is licensed under the Apache License, Version 2.0, and
its authors have signed the Midnight Foundation Contributor License
Agreement. Portions of this document were drafted with the assistance
of a large language model; the named authors reviewed and are
accountable for its entire content.
