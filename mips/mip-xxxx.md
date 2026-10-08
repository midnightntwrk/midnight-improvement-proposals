---
MIP: X
Title: ECDSA Authorisation for Custody Accounts
Authors:
  - Nicolas Di Prima (NicolasDP)
Status: Draft
Category: Standards
Created: 2026-08-25
Requires: MIP-0012, MIP-0013
Replaces: none
MPS: MPS-0018
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

This draft extends MIP-0013's signing boundary with the current secp256k1
(`k1`) and WebAuthn secp256r1 (`r1`) constructions and proposes bounded
verification obligations. Its value is interoperable approval without
giving the signing key to the prover. It is not yet a complete,
independently implementable standard: per-operation serialization,
common policy and cryptographic enforcement mapping remain open.

## Motivation

Independent wallets and custody contracts need to agree on exactly what
an ECDSA credential signs and what the circuit verifies. A mismatched
hash layer, envelope selector or WebAuthn policy breaks that boundary.
MIP-0013 specifies JubJub Schnorr; existing ECDSA account routes need an
explicit conformance delta and byte-exact recipes. The reference already
provides implementation and bounded evidence for this narrow extension.

## Specification

### 1. Scope and conformance delta

[MIP-0013 §10](mip-0013-account-authorisation.md#10-versioning)
permits a successor under a **new or revising MIP**. Numbering versus
amendment is editorial. AUTH-1 explicitly requires Schnorr: ECDSA routes
therefore require a conformance delta, replacing its signature relation,
key encodings and challenge construction with those below.

Device routes MUST preserve active-epoch membership, single-use entry
rolling, account-nonce advancement, last-device protection and atomic
abort. This preserves state invariants rather than MIP-0013's literal
gate order: the r1 reference computes the challenge and verifies before
rolling the entry. Mixed-scheme devices share the account's lifecycle
state; scheme-specific commitments distinguish their entries.

A scheme does not confer device or grant authority. Viewing and recovery
are separate capabilities. Grant lifecycle, scope and replay rules belong
to the [scoped-grants companion draft](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/docs/mps-mip/mips/mip-xxxx-scoped-grants.md).

MUST, MUST NOT, SHOULD and MAY use RFC 2119 meanings. Requirements below
are proposed obligations; reference recipes describe existing bytes.

### 2. Reference signing bytes

Here `H` is SHA-256 and `||` is byte concatenation. `pad(w, tag)` is
ASCII without embedded NUL, followed by zero bytes to exactly width `w`;
tags MUST fit without truncation. Salt, account address and hashes are
32 bytes. Salt is fresh per account. Coordinates `x`, `y` are canonical
32-byte little-endian integers; `epoch` is LE4 and `counter` is LE8.

#### Boot and device commitments

The existing boot/device recipes use **v2** for both ECDSA schemes:

```text
k1_boot   = H(pad(32, "midnight:account:boot:k1:v2") || salt || x || y || envelope)
k1_device = H(pad(32, "midnight:account:device:k1:v2") || account || x || y || envelope || epoch || counter)
r1_boot   = H(pad(32, "midnight:account:boot:r1:v2") || salt || x || y || rpIdHash || origin)
r1_device = H(pad(32, "midnight:account:device:r1:v2") || account || x || y || rpIdHash || origin || epoch || counter)
```

`envelope` is one byte (`Uint<8>`). `rpIdHash` is 32 bytes; `origin` is
the exact origin bytes, 21 bytes in the current reference profile.
Signed origins are never padded. These recipes bind the k1 envelope
and r1 RP/origin policy to the credential.

Sources: [k1 commitments](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account.compact#L439-L503),
[r1 commitments](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account-p256.compact#L28-L45).

#### Device-operation challenge

Operation tags retain `midnight:account:auth:k1:v1:<operation>` and
`midnight:account:auth:r1:v1:<operation>`. The shared device-route schema is:

```text
dst = H(pad(64, operation_tag))
c   = H(dst || account || x || y || encode(operation_values) || auth_nonce_LE8)
```

The first element is the **32-byte tag hash**, not the padded tag.
`encode(operation_values)` denotes the operation's typed Compact
encoding, including consumed witness values in the specified order.
The circuit MUST recompute `c` from the actual consumed inputs and the
pre-increment account nonce.
Consumed private values MUST be bound even when secret; the exclusion
is authorising material, not all secrets. A digest may substitute for
declared operation values only when the circuit recomputes it from
those values, as `issue_grant` does for its scope digest. ECDSA
challenges contain neither signature material nor a grinding nonce.

This schema is not a complete per-operation serialization specification.
The specification work under [Path to Active](#path-to-active) must supply
all types, widths, ordering and independent vectors, including native
Fields and points. Current recipes are in
[k1 challenge builders](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account.compact#L712)
and [r1 builders](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account-p256.compact#L74-L174);
[grant issuance](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account-p256.compact#L224-L241)
shows the recomputed-digest substitution.

Grantee calls use their own challenge domain and state. The
[r1 grant builder](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account-p256.compact#L278-L313)
hashes, in order, the padded-tag hash, account, x, y, grant ID,
`issued_at` (`Uint<64>`, LE8), typed operation inputs including consumed
witnesses, and `g.nonce` (LE8). `issued_at` is not the 32-bit revocation
generation. Grantee calls do not consume a device entry or use device
`auth_nonce`; the companion specifies their authorisation rules.

#### Digest passed to ECDSA verification

- **k1 envelope 0:** `H(c)`.
- **k1 envelope 1:** `H(ASCII("midnight_signed_message:32:") || c)`;
  the prefix is exactly **27 bytes**. Unknown selectors are rejected.
- **r1:** `H(authenticatorData || H(exact clientDataJSON))`, with the
  WebAuthn challenge equal to `c`, encoded as 43 unpadded base64url
  characters in client data.

k1 **never verifies the raw challenge directly**. Its authorisation
API carries the enrolled envelope selector. See
[digest construction](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account.compact#L541-L547),
[gate](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/account.compact#L881-L907),
[wallet signer](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/src/wallet/signer.ts#L278-L350) and
[independent raw-challenge rejection](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/src/tests/unit-offline.ts#L324-L339).

Both schemes MUST enforce canonical coordinates in the curve's base
field, on-curve non-identity keys and canonical scalars `1 <= r,s < n`.
Verification interprets the digest
big-endian modulo the curve order, rejects an identity result, and
checks `R.x mod n == r`. Both high-S and low-S are accepted. Equivalent
signatures remain bound to the same single execution by replay state;
signature bytes are private authorising material, not operation IDs.

### 3. Bounded WebAuthn verification

Named, versioned, bounded profiles are permitted. Each MUST declare
accepted forms and byte lengths, required fields and boundary checks,
flag/extension rules, wire-to-circuit conversions and conformance
vectors. Wallet and contract MUST establish compatible capabilities
before enrolment or grant issuance.

Every r1 gate MUST enforce in-circuit:

1. The authorised key, recomputed operation challenge, `webauthn.get`
   type, challenge encoding, and expected RP hash and signed origin.
   RP/origin policy comes from authenticated configuration or credential
   binding, never unchecked prover input. For grants, the signed origin
   must also match the companion's committed `origin_hash`.
2. User presence and the authorised UV policy, together with declared
   flags, cross-origin/top-origin and extension rules, including
   BS-implies-BE. The signed authenticator header is RP hash32, flags1,
   signCount4 (big-endian). Any accepted suffix requires specified checks.
3. Both hash layers over the **complete, exact signed bytes** and ECDSA
   verification. Fixed-shape reconstruction is permitted if every byte
   is constrained. Buffered implementations MUST constrain logical
   lengths and exclude unused capacity. Hashes, offsets and lengths
   supplied as advice MUST be independently constrained.

A full-envelope profile constrains its entire accepted serialization.
A profile using the
[W3C limited-verification algorithm](https://www.w3.org/TR/webauthn-3/#clientdatajson-verification)
MUST identify that algorithm and enforce its field/boundary checks
while hashing the full message. It does not establish arbitrary
trailing JSON syntax; profiles MUST state that suffix guarantee
accurately. Wallet-side parsing adds no proof guarantee. Unsupported
assertions MUST be rejected, never truncated, reserialized or padded
to fit.

**Downgrade protection:** every reachable verifier accepting a credential
MUST enforce its authorised policy. One installed verifier is insufficient
if a weaker sibling accepts the same credential. A single immutable
profile, a common enforced floor or authenticated routing may suffice;
otherwise credential/routing binding is required. This does not mandate
a stored profile ID. Updates MUST preserve policy or use an authorised
policy-change procedure.

**UV recommendation:** UP+UV on every WebAuthn gate is the simplest policy
and matches the reference. This recommendation is not an adopted common
floor. Requiring UV only for asset release leaves UP-only `add_device`,
`issue_grant` or recovery-authority replacement able to enable indirect
spending; any weaker proposal must address that escalation.

`signCount` is signed but MUST NOT supply account replay protection;
device/grant state supplies it. ES256 verification does not establish
hardware custody, device binding or a particular biometric.

### 4. Versioning

This specification is versioned by its MIP number and revision history.
Substantive changes after acceptance require a new or revising MIP with
an explicit conformance delta. Preimage changes require new domain tags
and explicit migration; named profiles carry their own versions.

## Rationale

Existing k1 integrations and authenticator-held r1 keys need different
signing envelopes but can use the same account lifecycle. Recording the
current bytes preserves signer/contract agreement. P-256 availability
alone does not establish that existing k1 consumers can migrate.

Named bounded profiles make accepted WebAuthn forms and proof obligations
explicit. They accommodate measured circuit constraints without claiming
universal browser support or adopting one implementation as the baseline.

## Path to Active

### Acceptance Criteria

The following specification decisions remain open in this Draft:

| Item | Required resolution |
|---|---|
| Complete signing recipes | Pin per-operation typed encodings and link existing byte-exact vectors; identify any uncovered cases. |
| Security policy | Decide the common UV floor and specify authorised policy binding across verifier routes without prescribing storage layout. |
| k1 support | Continued coexistence is recommended; justify any retirement using actual consumer and migration evidence, not P-256 availability alone. |
| Review | Map key/scalar admissibility, profile boundary guarantees and the high-S/replay argument to enforcing layers and pinned versions; retain unresolved cryptographic questions explicitly. |
| Editorial alignment | Agree successor versus amendment and reconcile MIP-0021 tags with explicit compatibility treatment. |

Active status requires the agreed specification, completed cryptographic
review and conformance evidence for its specified routes on a pinned
implementation. Existing evidence counts only for the cases it covers.

### Implementation Plan

Complete the recipe/vector and enforcement mapping from the linked
reference, resolve the table through review, then record conformance to
the accepted delta. Numbering and MPS-0018's solution-header update are
editorial steps under MIP-0001.

## Backwards Compatibility Assessment

Existing bytes remain the reference recipes. Changing a committed
preimage requires explicit versioning and migration; profile metadata
cannot silently change it. This draft sets no k1 retirement policy.
MIP-0013 continues to define the JubJub Schnorr/FROST profile. No ledger
or consensus change or hard fork is proposed by this extension.

[MIP-0021](mip-0021-domain-separation.md)
is Proposed and explicitly rejects a central registry. Existing zero
padding, colon-version tags and underscore operation names conflict with
its convention. Editors must reconcile these constructions without
silently migrating cryptographic bytes.

## Security Considerations

Sections 2–3 require call-bound signatures, replay state, admissible keys
and scalars, exact signed-byte verification and policy enforcement across
all reachable verifiers. Those are proof obligations, not consequences
of wallet-side parsing. The enforcement mapping and cryptographic review
above remain necessary; runtime rejection alone does not establish native
proof constraints. Device authority, grant authority, viewing access and
recovery remain distinct.

## Implementation

Reference sources and evidence below are pinned to Passport commit
`d3152081c35bb151d8fbf4fdfced0e23afa256a2`, except the separately pinned
variable-length experiment.

Compact 0.35 provides native P-256 verification. The
[reference `wa-json134` profile](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/WEBAUTHN.md#supported-profile-wa-json134)
accepts exactly 134-byte client JSON, a 21-byte origin and 37-byte
authenticator data: fixed field order and punctuation, unescaped ASCII
origin, `crossOrigin:false`, no extra members or extensions, and flags
5, 13 or 29 (UP+UV). It is
implementation-specific evidence, not an adopted W3C or MIP baseline.
The [verifier](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/contracts/webauthn.compact) and
[independent recipe checks](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/src/tests/p256-offline.ts)
document that construction.

The reference's specialised circuits use a **15,000-verifier-byte
deployment-wave budget**; implementation costs are recorded in
[P256-MEASUREMENTS](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/P256-MEASUREMENTS.md).

## Testing

The linked recipe checks and raw-challenge rejection provide existing
signing-boundary tests. Complete their coverage mapping under Path to
Active, including malformed keys/scalars, profile boundaries, policy
downgrades and replay; record uncovered cases explicitly.

One [live Safari localhost account-key rotation](https://github.com/midnightntwrk/passport/blob/d3152081c35bb151d8fbf4fdfced0e23afa256a2/contract/evidence/p256-webauthn/browser-flow.md)
was accepted. This is separate from software fixtures, grant tests and
caller-binding validation; it establishes no broader browser coverage.
Local runtime malformed-key negatives do not demonstrate native proof
constraints. Key-admissibility enforcement still needs the review above.

The standalone Passport #179 bounded256/origin96 experiment saved **zero**
proof or verifier-key bytes (6,364/2,745 unchanged), with 8.9× rows and 8.1×
prover-key bytes. Its fee-unbalanced software proofs were verified in
memory, not node-accepted account calls; see
[pinned findings](https://github.com/midnightntwrk/passport/blob/cd08b4b104668833654a7de3b2a96f252b38f950/experiments/webauthn-variable-json/FINDINGS.md).

## Acknowledgements

Hector Bulgarini reviewed the local signing-boundary draft in
[Passport #180](https://github.com/midnightntwrk/passport/pull/180).
That review is distinct from the outstanding cryptographic review.

## Copyright Waiver

This document is licensed under the Apache License, Version 2.0, and
its authors have signed the Midnight Foundation Contributor License
Agreement. Portions of this document were drafted with the assistance
of a large language model; the named authors reviewed and are
accountable for its entire content.
