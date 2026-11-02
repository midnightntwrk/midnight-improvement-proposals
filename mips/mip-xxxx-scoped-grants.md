---
MIP: xxxx
Title: Scoped Grants and dApp Connection for Custody Accounts
Authors:
  - Hector Bulgarini (hbulgarini)
  - Nicolas Di Prima (NicolasDP)
Status: Draft
Category: Standards
Created: 2026-11-02
License: Apache-2.0
Requires: [MIP-0012, MIP-0013, MIP-0030]
Replaces: none
MPS: MPS-0018
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

> **Draft scope.** This is a proposed grant-enabled account profile. Section
> 12 defines its composition with MIP-0013, including the explicit changes
> to AUTH-1 and AUTH-2 for grant calls. The reference schema is
> `spec_version = 3`; historical v2 measurements are identified as such.
> Implementation evidence is partial. The unchecked Path to Active items
> are acceptance work, not claims of completed conformance.

## Table of contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Specification](#specification)
  - [1. Scope of this document](#1-scope-of-this-document)
  - [2. Terminology and notation](#2-terminology-and-notation)
  - [3. Grantee keys and arms](#3-grantee-keys-and-arms)
  - [4. Ledger state and grant identity](#4-ledger-state-and-grant-identity)
  - [5. Scope semantics](#5-scope-semantics)
  - [6. The grant seam](#6-the-grant-seam)
  - [7. Lifecycle](#7-lifecycle)
  - [8. Read-scope delegation](#8-read-scope-delegation)
  - [9. GrantRequest and the redirect binding](#9-grantrequest-and-the-redirect-binding)
  - [10. Sign-in](#10-sign-in)
  - [11. Ecosystem carriage](#11-ecosystem-carriage)
  - [12. Relationship to MIP-0013 and combined conformance](#12-relationship-to-mip-0013-and-combined-conformance)
  - [13. Invariants](#13-invariants)
  - [14. Versioning](#14-versioning)
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

This MIP specifies a second authoriser class behind the authorisation
seam of MIP-0013, the **scoped grant**, and the ceremony by which a
third party obtains one. A grant is a contract-maintained record in a
custody account conforming to MIP-0012 and the combined MIP-0013
profile of section 12 that admits one
grantee key of a signature scheme specified here to a bounded subset of
the account's asset-facing operations: three spend circuits, one token
color, a per-call cap, a cumulative cap, a bound on the value of any
coin the grantee may touch, an optional recipient pin, and an expiry
that is always stated, with zero meaning never. Scope is enforced
in-circuit on every call, and any active device can revoke a grant from
chain state. A read-only grant is the anchor that makes an otherwise
invisible viewing capability enumerable and revocable.

The record is keyed by a contract-recomputable identity over the account
address, the grantee key and, on the secp256k1 arm, its envelope id, its
origin, and a slot number. Fields that would name a held coin's color, a
counterparty, a released value, or the grantee's origin are stored as
salted commitments, so the custody invariants of MIP-0012 hold
unchanged. The connection flow is the normative redirect binding of one
transport-independent `GrantRequest` object, with a browser-attested
proof of possession in place of a self-asserted origin and a chain
record in place of a bearer code. The dominant residual risk is that the
ledger enforces spend scope and not read scope.

## Motivation

MIP-0012 fixes how a custody contract holds and releases value behind
one abstract seam, `require_authorised()`, and its section 2 names
"delegated or scoped spending policies (grants, allowances, session
permissions)" as authorisation-policy objects behind that seam, deferred
to a conforming extension. MIP-0013 instantiates the seam with device
keys and states in its section 4 that "richer on-ledger policies (m-of-n
across device entries, scoped grants) are permitted extensions that
interact only with this seam and MUST NOT weaken any invariant of
section 9". Its S7 records the gap: `add_device` grants full authority,
so the only way to let a third party act on an account today is to make
it a device. This document specifies that extension and states the required AUTH-1
and AUTH-2 conformance changes in section 12; it does not silently
reinterpret the parent specification.

Today a dApp can hold a full device key (total compromise of asset
release if the dApp is compromised, the authorisation MIP's S5) or hold
nothing. The account-custody prototype showed a color-scoped,
value-capped, epoch-bound withdraw grant enforced in-circuit on a devnet
node; origin binding, expiry, signature-based grantee authentication,
and a connection protocol are what a standard must supply. The dApp
connector API and the Open Wallet Standard are wallet-side and
ephemeral. Compact 0.35.0 now exposes `kernel.caller()`: this revision
adds an optional immediate-contract caller restriction, checked by the
account callee itself (section 6.2.1). It is independent of browser-origin
attestation and retains the grantee signature and all spend bounds.

NEAR's access-key login flow is the closest deployed analogue of the
connection this MIP specifies: a dApp generates a key, redirects to the
wallet with its public key and return URL, and the wallet adds the key.

| NEAR defect | Closed by |
|---|---|
| login cross-site request forgery (CSRF): nothing proves the dApp possesses the key or binds the return to the requesting session | a possession proof by the grantee key, a browser-written origin attestation, and a session-bound `state` (section 9) |
| an open redirector through attacker-controlled return URLs | `redirect_uri` same-origin with the proven origin, matched exactly (section 9.7) |
| disclosure of the account's whole key list to every dApp | no key list in the response; the dApp reads its own record from chain |
| a non-expiring default | expiry always stated; never-expiring is an explicit zero |

OAuth 2.0 and its security best current practice fixed these classes a
decade ago; this MIP borrows the fixes and adds a chain record in place
of the bearer code an OAuth authorisation server returns (Rationale R9,
R13).

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in RFC 2119.

### 1. Scope of this document

This MIP standardises the grant record and its identity (section 4), its
scope semantics (section 5), the grant seam through which a grantee key
satisfies `require_authorised()` for exactly three asset-releasing
circuits (section 6), the lifecycle (section 7), read-scope delegation
(section 8), the `GrantRequest` object and its normative https redirect
binding (section 9), account-linked sign-in (section 10), and carriage
in Chain Agnostic Improvement Proposal (CAIP) sessions and Open Wallet
Standard policy terms (section 11). `issue_grant` is a device-gated
circuit the owner can call with no ceremony from their own client; a
conforming account is fully usable with no authoriser in existence. The
same record, seam, and lifecycle serve intra-user delegation through the
`self:` grantee form.

Out of scope:

- **Successor documents.** Unlinkable, address-hiding sign-in and any
  authoriser-signed identity assertion belong to a separate, not yet
  filed proposal, referred to below as "the sign-in MIP". Per-grantee or
  epoch-scoped viewing keys and the per-coin mirror are the MIP-0012 R9
  successor, the named dependency for spend without full disclosure.
  Device-wide retirement after duplicate enrolment is a separate problem:
  MIP-0013 section 6 retires a supplied device entry, not every entry
  that might represent the same key. This MIP does not claim to repair
  that device lifecycle limitation.
- **Deferred features.** A specified and evidenced `schnorr_bip340`
  grantee profile; complete r1 grantee conformance under MIP-0030
  (native secp256r1 is available); spend through wallet-provider keys (a connector surface
  displaying a decoded grant call); `.well-known` client metadata; a
  CAIP-10 account form for Midnight.
- **Non-goals.** A new injected wallet API or connector method; a
  client-side policy language (Open Wallet Standard PE-1); proving,
  submission, and fee sponsorship; credential proofs and non-asset
  objects; chain-agnostic grants (a grant authorises calls on one
  contract on one chain by construction); a "callable contracts" object;
  delegation chains; m-of-n device policy and threshold grantee keys;
  window-bounded rate limits (schema reserved), tombstone pruning, and
  automatic expiry cleanup; authoriser discovery; Related Origin
  Requests and passkeys shared across the authoriser and dApp origins;
  merging device and grant twins; `deriveSecret` counterparts
  (MIP-0015); migration of `spec_version = 1` accounts; how the viewing
  secret reaches a second authoriser device (MIP-0012); Zswap user-held
  coins; address-only disclosure ceremonies.

### 2. Terminology and notation

- **Account**: a custody contract instance conforming to MIP-0012 ("the
  custody MIP") and MIP-0013 ("the authorisation MIP"). **Device**,
  **device entry**, **epoch**, **`auth_nonce`**, **viewing
  capability**, **viewing secret**, **inbox**, **coin store**, **color**:
  as defined there. A grantee is not a device and never counts toward
  `device_count`.
- **Grant**: a record in the account's ledger state admitting one grantee
  key to a bounded subset of operations. **Grantee**: its holder: a
  browser dApp, a wallet provider, an agent, or the owner through a
  delegate key. **Grantee key**: a `(scheme, pk)` pair from the
  signature schemes of section 3, plus, for k1 only, an envelope id.
- **Arm**: the per-scheme circuit family: `v1` (Schnorr over JubJub),
  `k1` (ECDSA over secp256k1), `r1` (ECDSA over secp256r1 through the
  WebAuthn envelope).
- **Origin**: the normalised identity of the grantee (section 4.4): an
  RFC 6454 web origin for browser clients; a reverse-DNS name (`rdns:`),
  a Midnight Agent Identity Standard identifier (`mais:`), or an
  owner-chosen label (`self:`) for non-browser clients.
- **Grant identity** (`grant_id`): the contract-recomputable hash keying
  the record (section 4.3). **Slot**: a dApp-chosen byte, default `0`,
  distinguishing up to 256 records of one key at one origin.
- **Scope**: the immutable bounds of a grant; **plaintext scope**: the
  scope as the approver consents to it, before commitment.
- **Grant twin**: a gated circuit `<operation>_with_grant_<arm>` that
  verifies a grant where the device twin verifies a device signature.
- **Authoriser**: a role (section 11.3): a party holding a device key of
  the account that renders a `GrantRequest`, drives `issue_grant`, and,
  to serve `read`, holds the viewing secret.
- **Browser binding**: delivery of a `GrantRequest` by top-level
  navigation to an authoriser page. **Non-browser binding**: any other
  delivery (a pasted or scanned object, a local channel).
- **Proof**: the possession-and-origin proof object that travels beside
  a `GrantRequest` as a separate parameter and is never part of the
  signed bytes (section 9.2).
- **Notation.** `H(x)` is SHA-256 of `x`. `pad(N, s)` is the ASCII bytes
  of `s` followed by zero bytes to `N` bytes; `s` longer than `N` is
  invalid. `u8(n)`, `u64(n)`, `u128(n)` serialise `n` little-endian at 1,
  8, and 16 bytes. `flag(b)` is one byte, `0x01` for true and `0x00`
  for false, which is the compiled encoding of a `Boolean` tuple
  element. `int_le(x)` is the integer whose little-endian encoding is `x`. `||`
  is byte concatenation. `base64url(x)` is the RFC 4648 section 5
  encoding of `x` without padding.

### 3. Grantee keys and arms

#### 3.1 Grantee arms

A grantee key is a `(scheme, pk)` pair from the arm table below, plus,
for `k1` only, an `envelope` id. MIP-0013 specifies the JubJub signing
boundary. [MIP-0030](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0030-ecdsa-account-authorisation.md)
(hereafter "the schemes MIP") specifies the ECDSA digest and bounded
WebAuthn obligations; it is a Draft, not a scheme registry. This MIP
defines the grantee wire names and grant-specific byte recipes. Its
conformance requirements are stated here, including key admissibility
and profile checks in sections 3.3 and 6.2; existing reference support
does not establish complete conformance to those requirements.

| Arm | Scheme | Envelope | Specification basis | Reference support |
|---|---|---|---|---|
| `v1` | Schnorr over JubJub | intrinsic (none) | MIP-0013 and this MIP's grant recipes | implemented; evidence limits in Implementation |
| `k1` | ECDSA over secp256k1 | `0` unprefixed digest envelope, `1` connector prefix | MIP-0030 and this MIP; no adopted sunset | implemented; enforcement mapping remains open |
| `r1` | ECDSA over secp256r1 through the WebAuthn envelope | intrinsic (WebAuthn message) | MIP-0030 bounded profiles and this MIP | Compact 0.35 native P-256; bounded reference profile implemented; full conformance remains open |

Before issuance, the authoriser and grantee MUST establish that the
account implements the requested arm and, for `r1`, a compatible named,
versioned WebAuthn profile. Each profile MUST state accepted forms and
byte lengths, required fields and boundary checks, flag and extension
rules, wire-to-circuit conversions, and conformance vectors. Unsupported
forms MUST be refused, never truncated, reserialized, or padded to fit.
An account's advertised capability is not evidence that its verifier
enforces the profile; that enforcement is a proof obligation.

`envelope` is the k1 digest selector and nothing else: the k1 arm
verifies over `envelope_digest(envelope, h)`, where
`envelope_digest(0, h) = H(h)` and
`envelope_digest(1, h) = H("midnight_signed_message:32:" || h)`, the
27-byte mandatory prefix of the dApp connector's `signData` surface.
Because `envelope` also enters `grant_id` (section 4.3), an envelope-1
grantee has a different `grant_id` and therefore a different challenge
from an envelope-0 grantee with the same key, origin, and slot; the two
envelopes are not two wrappings of one challenge `h`. `envelope` is
absent from the wire form and the identity preimage for `v1` and
`r1`. No `2 = webauthn` envelope id is filed. The connector's
`schnorr_bip340` scheme is reserved; this MIP does not define its grant
arm.

#### 3.2 Grantee forms

| Form | Arm, envelope | Availability condition | Origin proof at issuance | Scopes | Serves |
|---|---|---|---|---|---|
| Per-dApp WebAuthn credential on the dApp origin | `r1` | compatible deployed profile | the assertion itself (`clientDataJSON.origin`) | all | browser dApps whose assertions fit the profile |
| Per-connection software key gated by a companion passkey on the dApp origin | `k1` envelope `0`, or `v1` | deployed arm and companion ceremony | WebAuthn assertion by the companion passkey over `request_digest`, which covers the software key | all | browser dApps |
| Wallet-provider key through the connector `signData` | `k1` envelope `1` | connector support and companion ceremony | as the software form; `signData` supplies possession only | `read` only | "connect my wallet to my account" |
| Agent or vaulted key (`rdns:`, `mais:`) | `v1` or `k1` envelope `0` | deployed arm | operator assertion in the non-browser binding, displayed as such | all | agents, servers |
| Intra-user delegate (`self:<label>`) | `v1` or `k1` envelope `0` | deployed arm | none: approved in person on the owner's own device | all | a second device, a script, a household member |

Rules:

- Grantee keys MUST be generated per connection: one key per account per
  origin; a key MUST NOT be reused across accounts or origins.
- An authoriser MUST refuse to issue a grant to a `pk` it knows to be
  enrolled as a device key of the same account (GR-2); the contract
  cannot check this, since device keys are not stored.
- **Wallet-provider keys are restricted to `read`.** Connector
  `signData` under envelope `1` is a blind 32-byte signing surface, and
  every element of the unshielded twin's challenge is public or
  attacker-chosen (the shielded twins additionally need only the viewing
  secret, which every read-granted party holds), so any other site the
  same wallet connects to could obtain a valid spend signature by asking
  the wallet to sign an opaque message. An authoriser MUST return
  `unsupported_scheme` for any withdraw string requested by an
  envelope-1 grantee, and the `k1` grant seam enforces the same
  restriction in-circuit: every `k1` grant twin asserts `envelope == 0`
  before any other check (section 6.2 step 1), so a record issued to an
  envelope-1 key admits no spend whatever its flags say. Spend through
  envelope `1` waits on a connector surface that displays a decoded
  grant call with per-call confirmation, and on lifting that assert in
  a circuit revision.

#### 3.3 Key validation

The authoriser MUST validate the grantee key before issuance. The grant
seam MUST enforce key admissibility on every use, independently of that
client check. `issue_grant` takes `grant_id`, not the grantee key, so
the issuance circuit cannot validate the key: a record's existence is
not proof of a valid credential. A directly issued record for an invalid
key MUST confer no spend authority.

| Arm | Required checks | Enforcement evidence |
|---|---|---|
| `k1` | canonical coordinates `0 <= x,y < p`; on-curve `y^2 == x^3 + 7 (mod p)`; non-identity key | Compact 0.35 runtime and native-input validation in the October review below; no full ECDSA proof coverage claimed |
| `v1` | canonical, on-curve `pk` and `sig_r`, per MIP-0013 S2; `[8]pk != O`; subgroup semantics must prevent small-order or mixed-order components from admitting a forged signature | October review below distinguishes runtime rejection, native canonical-input checks and targeted Jubjub constraint/full-proof checks; arbitrary malicious internal witnesses are not exhaustively covered |
| `r1` | canonical coordinates `0 <= x,y < p`; on-curve `y^2 == x^3 - 3x + b (mod p)`; non-identity key; the issuing client accepts only a CBOR Object Signing and Encryption (COSE) EC2 P-256 key with canonical coordinates | Compact 0.35 runtime and native-input validation in the October review below; bounded WebAuthn profile and grant-route conformance remain separate obligations |

For both ECDSA arms every verifier MUST enforce canonical scalars
`1 <= r,s < n`, interpret the digest big-endian modulo the curve order,
reject an identity verification result, and compare `R.x mod n` with
`r`. Both high-S and low-S signatures are accepted. Implementations MAY
discharge checks through native types or primitives only where their
proof constraints enforce the requirement.

The 15 October 2026 validation in [Passport PR #182](https://github.com/midnightntwrk/passport/pull/182)
supersedes the September E2 observations for the pinned implementation.
Its [report](https://github.com/midnightntwrk/passport/blob/1932fab4c422e301706d422648d558277e00af64/research/passport-ecc-validation.md#4-evidence-and-remaining-checks)
and [recorded evidence](https://github.com/midnightntwrk/passport/blob/1932fab4c422e301706d422648d558277e00af64/contract/evidence/ecc-validation/README.md#results)
cover the unchanged account compiled with Compact 0.35.0
(`debb05f9414b9d1e176741c2be289bb32233f0fc`) and runtime 0.20.0.
Both ECDSA arms rejected the tested identity, off-curve, wrong-curve and
out-of-range signer keys at bootstrap, device and grant gates. Device
signature tests rejected `r = 0`, `s = 0`, `r = n` and `s = n`, and
accepted both high-S and low-S signatures. Native bootstrap-input checks
also rejected malformed foreign-curve encodings; accepted ECDSA variants
passed native preprocessing. The review claims no full ECDSA proof run
or exhaustive malicious internal-witness coverage. Sampled rejection is
evidence about those inputs, not a substitute for the remaining
enforcing-layer review and grant-route conformance required by MIP-0030
and Path to Active; it does not identify a missing scalar or point guard.

Key disjointness is a different obligation. The authoriser MUST refuse
known device-key reuse under section 3.2, but the contract cannot infer
all enrolled keys from rolling device entries. Historical E2 issued a
grant to an enrolled device key and exercised it successfully. Device
and grant records and signature domains are separate; that separation
does not establish that their underlying keys differ.

#### 3.4 Wire forms of keys and off-chain signatures

`pk` on the wire is lowercase hexadecimal:

| Arm | `pk` | Preimage element |
|---|---|---|
| `k1`, `r1` | 128 hex: affine `x \|\| y`, each a 32-byte little-endian integer; SEC 1 inputs (compressed or uncompressed, big-endian) MUST be decompressed and byte-reversed per coordinate before use | `x`, `y` as given |
| `v1` | 128 hex: affine `x \|\| y`, each a 32-byte little-endian canonical element of the BLS12-381 scalar field (the JubJub base field); `JubjubPoint` is opaque in Compact with no compression builtin, so the circuit binds the two coordinates, as the device family already does for `pk` and `sig_r` | `x`, `y` as given (64 bytes) |

A wire coordinate at or above the applicable field modulus MUST be
rejected, never reduced. ECDSA signature scalars outside `1 <= r,s < n`
MUST likewise be rejected rather than reduced. Only canonical values
enter the byte recipes. The historical client encoder rejected one
cross-arm key before building a transaction; that observation does not
establish the in-circuit requirements of section 3.3 for other inputs.

Wire `scheme` names, keyed to the arm. The arm, not the wire
name, selects the DST marker of section 6.3 and the identity tag of
section 4.3:

| Wire `scheme` | Arm | DST marker | `envelope` member | Source |
|---|---|---|---|---|
| `ecdsa_secp256k1_sha256` | `k1` | `k1:` | MUST, `0` or `1` | dApp connector specification |
| `schnorr_bip340` | reserved | none | absent | dApp connector specification |
| `schnorr_jubjub` | `v1` | empty | absent | this MIP |
| `ecdsa_secp256r1_webauthn` | `r1` | `r1:` | absent | this MIP |

Off-chain signatures by a grantee key over a fixed 32-byte digest `d`
(`request_digest`, section 9.2; `signin_digest`, section 10). Every
verifier MUST accept both `s` forms of an ECDSA signature, matching the
in-circuit policy of section 3.3 and MIP-0030 section 2. Testing item 5
requires a high-S vector per ECDSA arm. Some verifier stacks enforce low-S by
default and refuse a high-S signature as presented; such a verifier MUST
normalise `s` to the low form before verifying, and the Rust reference
does so.

| Arm | Signs | Encoding |
|---|---|---|
| `k1` | `envelope_digest(envelope, d)` | 128 hex: `r \|\| s`, each a 32-byte little-endian integer |
| `r1` | a WebAuthn assertion with `challenge = d` on the dApp origin | the authenticator's ASN.1 DER `ECDSA-Sig-Value`, base64url, with `clientDataJSON` and `authenticatorData` beside it |
| `v1` | key-prefixed Schnorr: `c = int_le(H(R \|\| pk \|\| d)) mod r_J`, `s = r + c * sk mod r_J`, with `R` and `pk` each the 64-byte `x \|\| y` form | 192 hex: `R.x \|\| R.y \|\| s`, `s` a 32-byte little-endian integer below `r_J` |

The `v1` off-chain form differs from the in-circuit form (unprefixed,
ground challenge); separation rests on the tag and the key prefix, a
cryptographer-review item.

### 4. Ledger state and grant identity

#### 4.1 New cells and structs (`spec_version = 3`)

A grant-capable account extends the parent MIPs' ledger state with two
cells; the caller-bound schema exposes `spec_version = 3`
(Backwards Compatibility Assessment). Ledger-schema excerpt, not
implementation code:

```compact
export ledger grants:           Map<Bytes<32>, GrantRecord>;  // keyed by grant_id
export ledger grant_generation: Uint<32>;  // bumped only by revoke_all_grants
```

`GrantScope` (immutable after issue; 196 bytes):

| Field | Type | Width | Meaning |
|---|---|---|---|
| `op_withdraw_unshielded` | `Boolean` | 1 | admits the `withdraw_unshielded` twin |
| `op_withdraw_shielded` | `Boolean` | 1 | admits the `withdraw_shielded` twin |
| `op_withdraw_shielded_to_contract` | `Boolean` | 1 | admits the `withdraw_shielded_to_contract` twin |
| `read` | `Boolean` | 1 | declarative: the viewing capability was delegated (section 8) |
| `object_commit` | `Bytes<32>` | 32 | salted commitment to `color`, `recipient_kind`, `recipient`, `max_coin_value` (section 4.5) |
| `per_call_cap` | `Uint<128>` | 16 | `amount <= per_call_cap` on every call; MUST be `<= cap` |
| `cap` | `Uint<128>` | 16 | `spent + amount <= cap` cumulatively |
| `expires_at` | `Uint<64>` | 8 | whole seconds since the UNIX epoch, the unit of `kernel.blockTimeLessThan` (section 5.1); `0` means never |
| `rp_commit` | `Bytes<32>` | 32 | salted commitment to `rp_id_hash` (SHA-256 of the dApp host for `r1`, zero otherwise); committed for every arm so the record reveals neither host nor arm |
| `read_pk_hash` | `Bytes<32>` | 32 | SHA-256 of the delegate's X25519 `read_pk` when `read`; zero otherwise |
| `window_len` | `Uint<64>` | 8 | reserved; MUST be `0` |
| `window_cap` | `Uint<128>` | 16 | reserved; MUST be `0` |
| `caller_commit` | `Bytes<32>` | 32 | zero for unrestricted calls; otherwise salted commitment to the required immediate contract caller |

`GrantRecord` (81 bytes plus the scope):

| Field | Type | Width | Written by | Meaning |
|---|---|---|---|---|
| `epoch` | `Uint<32>` | 4 | contract, at issue | `device_epoch` observed inside `issue_grant` |
| `gen` | `Uint<32>` | 4 | contract, at issue | `grant_generation` observed inside `issue_grant` |
| `issued_at` | `Uint<64>` | 8 | contract, at issue | `auth_nonce` as advanced by the issuing call's device seam; unique per issuance |
| `nonce` | `Uint<64>` | 8 | contract, every grant call | per-grant freshness; read into the challenge and advanced |
| `spent_commit` | `Bytes<32>` | 32 | contract, at issue and every grant call | salted commitment to the cumulative value released |
| `window_start` | `Uint<64>` | 8 | reserved | `0` |
| `window_spent` | `Uint<128>` | 16 | reserved | `0` |
| `active` | `Boolean` | 1 | contract | `false` is a tombstone (revoked) |
| `scope` | `GrantScope` | 196 | contract, at issue | immutable after issue |

About 309 bytes per grant including the 32-byte map key (`grant_id`);
one hundred grants are about 31 KB. The record stores no key, no origin,
and no scheme. A struct embedding a struct as a `Map` value compiles,
and its lookup and insert execute, on the reference toolchain
(Implementation); the generated client type exposes the record and its
scope as one nested value.

#### 4.2 Plaintext scope

The plaintext scope the approver consents to, and that `issue_grant`
takes as arguments:

| Argument | Type | Rule |
|---|---|---|
| the four flags | `Boolean` each | at least one operation flag or `read` MUST be set; shielded flags imply `read` |
| `color` | `Bytes<32>` | the token color; all-zero is Night |
| `recipient_kind` | `Uint<8>` | `0` any, `1` `UserAddress`, `2` `ZswapCoinPublicKey`, `3` `ContractAddress` |
| `recipient` | `Bytes<32>` | when `recipient_kind != 0`, the `bytes` field of the recipient struct, as is (each of the three Compact types is `{ bytes: Bytes<32> }`); zero otherwise |
| `max_coin_value` | `Uint<128>` | a shielded twin aborts if the consumed coin's value exceeds it; MUST be `>= per_call_cap` when any spend flag is set |
| `rp_id_hash` | `Bytes<32>` | `H(host(client_id))` for an `r1` grantee; zero otherwise |
| `per_call_cap`, `cap`, `expires_at`, `read_pk_hash`, `window_len`, `window_cap` | as in `GrantScope` | as in `GrantScope` |
| `caller` | `Maybe<ContractAddress>` | optional immediate calling contract; absent encodes `is_some = false` and an all-zero address |
| `scope_salt` | `Bytes<32>` | see below |

`scope_salt` MUST be 32 bytes drawn fresh from a cryptographically secure
random source at every issuance, MUST NOT be reused across grants,
accounts, or re-issues of the same id, and MUST NOT be derived from any
public value. It is necessary to open the record's commitments and
therefore required to construct any grant call, but not sufficient for
authority, which rests on the grantee signing key alone. The circuit
computes `object_commit`, `rp_commit`, `caller_commit`, and the initial `spent_commit`
from these arguments; `color`, `recipient`, `max_coin_value`, and
`rp_id_hash` never appear in ledger state in the clear. The salt is bound
in the device challenge, delivered to the grantee in the response, and
kept in the owner's roster (section 7.5); the authoriser SHOULD NOT
retain it after the response is delivered.

#### 4.3 Grant identity

`grant_id` is a deterministic commitment recomputed in-circuit from the
presented key at every use and pre-derived by the authoriser at issue.
Every preimage in this MIP is a byte concatenation of fixed-width
elements hashed with SHA-256 (`persistentHash` over byte atoms is
SHA-256 of their raw concatenation); the exported pure circuits are one
implementation of the recipe, not its definition, so a dApp recomputes
`grant_id` with plain SHA-256. Integers are little-endian at their
Compact width; the vectors of Testing item 5 pin the compiled encoding,
and a divergence is corrected in the recipe, never in the vectors.

| Arm | `grant_id` preimage (in order) | Width |
|---|---|---|
| `k1` | `pad(32, "midnight:account:grant:id:k1:v1") \|\| self \|\| x \|\| y \|\| u8(envelope) \|\| origin_hash \|\| u8(slot)` | 162 |
| `r1` | `pad(32, "midnight:account:grant:id:r1:v1") \|\| self \|\| x \|\| y \|\| origin_hash \|\| u8(slot)` | 161 |
| `v1` | `pad(32, "midnight:account:grant:id:v1") \|\| self \|\| x \|\| y \|\| origin_hash \|\| u8(slot)` | 161 |

where `self` is the account's contract address (32 bytes) and the key
elements are the wire forms of section 3.4 (`x`, `y` on every arm, 64
bytes; the compiled `k1` preimage is 162 bytes with `envelope` and
`slot` as single bytes, and the `v1` one 161 bytes over the two
coordinates of the `JubjubPoint` element, neither coordinate alone
reproducing it). The `v1` recipe does not reject the JubJub identity
`(0, 1)`, which has coordinates and yields a well-formed `grant_id`;
the rejection is section 3.3's `[8]pk != O` guard, which any
implementation lifting the derivation on its own MUST also apply. The
consequences are GR-3:
one tuple has at most one live record, every record of a key at an
origin is enumerable in at most 256 lookups, and any mismatch of key,
origin, slot, arm, or envelope fails at the membership assert. No secret
lives in the identity; `grant_id` hides in the entropy of a
per-connection key, and for a publicly known key the test "is key K
connected to origin X on account A" is trivially answerable (S27).

#### 4.4 Origin normalisation and `origin_hash`

`origin_hash = H(pad(32, "midnight:account:grant:origin:v1") || client_id_bytes)`,
computed off-chain, where `client_id_bytes` are the ASCII bytes of the
normalised `client_id` appended raw, with no length prefix and no
padding, immediately after the 32-byte tag pad. The tag fills the pad
exactly, so it is not extensible without a new version. `client_id` is
normalised exactly:

- browser clients: the lowercase ASCII RFC 6454 serialisation of the
  origin (`https://bank.example`): scheme and host lowercase, default
  port omitted, non-default port kept, no path, no trailing slash,
  internationalised domain names in punycode;
- non-browser clients: one of
  - `rdns:<reverse-dns>`, a dot-separated sequence of LDH labels
    (letters, digits, and hyphen), lowercase, at most 253 bytes;
  - `mais:<id>`, the agent identifier of the Midnight Agent Identity
    Standard proposed under MPS-0015, lowercase, at most 128 bytes,
    drawn from the unreserved URI characters of RFC 3986 section 2.3;
  - `self:<label>`, at most 64 bytes of lowercase ASCII letters, digits,
    hyphen, and underscore, chosen by the owner.

A `client_id` matching none of these productions, or carrying any other
byte, MUST be rejected with `invalid_request`. No Unicode normalisation
applies, because no non-ASCII byte is admitted. `origin_hash` is a
private argument of every grant twin and of nothing else, never appears
in ledger state, and is derived from public information. `slot`,
`scope_salt`, `rp_id_hash`, and `spent_prev` are likewise private
arguments.

#### 4.5 Commitments and the scope digest

| Derivation | Preimage (in order) |
|---|---|
| `object_commit` | `pad(32, "midnight:account:grant:obj:v1") \|\| scope_salt \|\| color \|\| u8(recipient_kind) \|\| recipient \|\| u128(max_coin_value)` |
| `spent_commit` | `pad(32, "midnight:account:grant:spent:v1") \|\| scope_salt \|\| u128(spent)` |
| `rp_commit` | `pad(32, "midnight:account:grant:rp:v1") \|\| scope_salt \|\| rp_id_hash` |
| `caller_commit` (present) | `pad(32, "midnight:account:grant:caller:v1") \|\| scope_salt \|\| caller.value` (the contract address's 32 bytes); absent is the all-zero sentinel, not a hash |
| `scope_digest` | `pad(32, "midnight:account:grant:scope:v2") \|\| scope_salt \|\| flag(op_withdraw_unshielded) \|\| flag(op_withdraw_shielded) \|\| flag(op_withdraw_shielded_to_contract) \|\| flag(read) \|\| color \|\| u8(recipient_kind) \|\| recipient \|\| u128(max_coin_value) \|\| u128(per_call_cap) \|\| u128(cap) \|\| u64(expires_at) \|\| rp_id_hash \|\| read_pk_hash \|\| u64(window_len) \|\| u128(window_cap) \|\| caller_commit` |

`caller_commit` MUST reject an absent value whose address is not zero,
and a present value whose hash equals the reserved zero sentinel. At use,
the callee obtains the address from `kernel.caller()`, not from a grant
argument. The device's issue signature binds the pin through `scope_digest`.

The hiding of the nonzero commitments rests on `scope_salt` meeting
section 4.2 and staying with the owner and the grantee; a constant or
reused salt makes `object_commit` dictionary-testable over the public
list of colors, turns `spent_commit` into a lookup table publishing
every released amount, and equates grants. Salt reuse is a negative
conformance item (Testing item 2). `scope_digest` is the single element
through which the `issue_grant` device challenge binds the whole
plaintext scope (AUTH-3 by collision resistance); flags enter as single
bytes. The `scope` tag family's naming and compatibility review under
MIP-0021 is an acceptance criterion (section 14). Its eighteen elements compile and hash as the
raw concatenation, a 309-byte preimage; the other preimages of this
section are 145 (`object_commit`), 80 (`spent_commit`), and 96
(`rp_commit` and present `caller_commit`) bytes.

### 5. Scope semantics

#### 5.1 Axes and issue rules

A grant realises three axes: the **operation** axis as three Booleans
and a declarative `read`; the **object** axis as one token color and an
optional recipient pin, committed; the **quantitative** axis as
`per_call_cap`, `cap`, `max_coin_value`, and `expires_at`. Rate windows
are reserved in the schema and not enforced.

An optional **caller** axis pins the immediate contract that must call
the account (section 6.2.1). It is independent of the recipient and
`client_id`. A requested pin may be retained exactly, never removed or
replaced under attenuation; an unrestricted request may be narrowed to
one contract with explicit consent.

Rules asserted at issue, in this order; the `expires_at` checks that
fall on the client and the authoriser rather than on the circuit are
stated below:

1. at least one operation flag or `read` MUST be set (no "empty means
   all");
2. `op_withdraw_shielded` or `op_withdraw_shielded_to_contract` implies
   `read` (a grantee must hold the viewing secret to select a coin);
3. `per_call_cap <= cap`;
4. any spend flag requires `cap > 0` and `max_coin_value >= per_call_cap`;
5. `read` requires a non-zero `read_pk_hash`;
6. `window_len == 0` and `window_cap == 0`;
7. a record with no operation flag set MUST carry an all-zero `color`,
   `recipient_kind = 0`, an all-zero `recipient`, and `max_coin_value`,
   `per_call_cap`, and `cap` all zero, so that `object_commit` over a
   read-only grant is determined by `scope_salt` alone; `caller` MUST be absent.

`max_coin_value` bounds the value at risk from a shielded grant, which
is the coin the grantee selects and could leave without change (R7);
clients SHOULD recommend `max_coin_value == cap` where the owner can
pre-split coins.

`expires_at` is a count of whole seconds since the UNIX epoch, the unit
of `kernel.blockTimeLessThan` on ledger 9 (node 2.1.0, recorded on-node,
E3), with `0` meaning never, stated explicitly. The comparison is a
ledger read whose Boolean result the transaction records in its public
transcript and the node re-executes at mempool admission; it is not a
proof constraint (one public input, no constraints). A record with
`expires_at == 0` records no time read at all (the generated code and
the ZKIR guard the query on `expires_at != 0`), so a never-expiring
grant is insensitive to block time on the node exactly as in the
simulator. The client evaluates the seam against its own wall clock,
not a block time: a clock running ahead of the network refuses calls
the chain would still accept, and a clock running behind builds calls
the network refuses at admission. The node's admission-time block time
runs about one block interval plus a network-defined tolerance ahead of
the wall clock (measured 8.2 to 10.2 s ahead on a 6 s-block localnet;
the tolerance is the node's and is not pinned here), so `expires_at` is
a bound. The usable horizon is narrower than that margin, because the
grantee's own proving time falls between its clock reading and the
node's evaluation: a grant call on a shielded twin at k = 17 measured
14.5 s from build to the node's refusal, nearly all of it the proof. The
horizon is therefore one block interval plus the network tolerance plus
the call's own prove-and-balance time, of the order of 20 s for a
shielded twin on the reference stack, and a wallet that offers a
five-second grant offers one that cannot be exercised. Clients MUST
sanity-check a non-zero `expires_at` against their own wall clock and a
freshly read head block time before signing, SHOULD treat a value less
than one block interval plus the tolerance plus their own proving time
ahead of that head block time as already expired, and an authoriser
SHOULD refuse a value already in the past. There is no `network_id` cell
(R15); the wire `chain` member remains normative for the request.

#### 5.2 Feature strings and the mapping table

Scope travels on the wire as feature strings named after the circuit
exports. Each maps to exactly one Boolean; the string is canonical on
the wire and the Boolean in the record:

| Feature string | Record field | Gated twin | Implies | Connection scope string (Open Wallet Standard) |
|---|---|---|---|---|
| `midnight:account:withdraw_unshielded` | `op_withdraw_unshielded` | `withdraw_unshielded_with_grant_<arm>` | | `midnight:unshielded` |
| `midnight:account:withdraw_shielded` | `op_withdraw_shielded` | `withdraw_shielded_with_grant_<arm>` | `read` | `midnight:shielded` |
| `midnight:account:withdraw_shielded_to_contract` | `op_withdraw_shielded_to_contract` | `withdraw_shielded_to_contract_with_grant_<arm>` | `read` | `midnight:shielded` |
| `midnight:account:read` | `read` | none (section 8); unshielded balances are public and need no capability | | `midnight:shielded` (read side) |
| `midnight:account:signin` | none | none; reserved for the sign-in MIP, refused with `invalid_scope` | | none |

The mapping is many-to-one, so a peer translating from a coarse
connection scope MUST ask for the grant strings it needs. The strings
travel in the CAIP-25 `scopedProperties` member (section 11.1), never in
`methods`, and a CAIP-25 peer MUST NOT treat one as invokable. A spend
string without `color`, `cap`, `max_coin_value`, and `expires_at`,
`per_call_cap > cap`, `max_coin_value < per_call_cap`, or a recipient
kind no requested operation can honour is `invalid_scope`; a withdraw
string from an envelope-1 grantee is `unsupported_scheme`.

### 6. The grant seam

#### 6.1 Circuit list

For each supported grantee arm `s` in `{jubjub, k256, p256}`, a
conforming contract exports exactly three
grant twins over the unchanged custody chips, plus the pure circuits
they and their callers need:

- `withdraw_unshielded_with_grant_<s>(color: Bytes<32>, amount: Uint<128>, recipient: UserAddress, ...grant auth): []`
- `withdraw_shielded_with_grant_<s>(recipient: ZswapCoinPublicKey, color: Bytes<32>, amount: Uint<128>, change_entry: Bytes<192>, enc_pk: Bytes<32>, ...grant auth): Maybe<ShieldedCoinInfo>`
- `withdraw_shielded_to_contract_with_grant_<s>(recipient: ContractAddress, color: Bytes<32>, amount: Uint<128>, change_entry: Bytes<192>, enc_pk: Bytes<32>, ...grant auth): [ShieldedCoinInfo, Maybe<ShieldedCoinInfo>]`
- exported pure `challenge_<operation>_with_grant_<s>` for each of the
  three, and `derive_grant_id_with_<s>`.

`enc_pk` is the value of the `enc_key` cell the grantee encrypted
`change_entry` to (section 6.2 step 5). `...grant auth` trails the
operation arguments, in this order:

| Arm | Grant authorising material |
|---|---|
| `k256` | `pk: Secp256k1Point, envelope: Uint<8>, origin_hash: Bytes<32>, slot: Uint<8>, scope_salt: Bytes<32>, recipient_kind: Uint<8>, pinned_recipient: Bytes<32>, max_coin_value: Uint<128>, spent_prev: Uint<128>, sig: Secp256k1EcdsaSignature` |
| `jubjub` | `pk: JubjubPoint, origin_hash: Bytes<32>, slot: Uint<8>, scope_salt: Bytes<32>, recipient_kind: Uint<8>, pinned_recipient: Bytes<32>, max_coin_value: Uint<128>, spent_prev: Uint<128>, sig_r: JubjubPoint, sig_s: Field, grind_nonce: Uint<64>` |
| `p256` | as `k256` without `envelope`, with `rp_id_hash: Bytes<32>` after `max_coin_value`, and with the WebAuthn witness set (`client_data_json` bytes, `authenticator_data: Bytes<37>`, `sig`) in place of `sig` |

There is no nonce argument: the record's `nonce` is read in-circuit.
The authorising material is witness data and MUST NOT be disclosed.

Per supported device arm `a` in `{jubjub, k256, p256}`, device-gated through the
unchanged device seam:

- `issue_grant_with_<a>(grant_id: Bytes<32>, <plaintext scope of section 4.2>, ...device auth): []`
- `revoke_grant_with_<a>(grant_id: Bytes<32>, ...device auth): []`
- `revoke_all_grants_with_<a>(...device auth): []`
- exported pure `derive_grant_scope_digest`, `derive_grant_object_commit`,
  `derive_grant_spent_commit`, `derive_grant_rp_commit`, `derive_grant_caller_commit`, and
  `challenge_<lifecycle circuit>_with_<a>` in the existing device tag
  family (`midnight:account:auth:v1:issue_grant`,
  `midnight:account:auth:k1:v1:issue_grant`, and so on); the
  `issue_grant` challenge's argument list is `[grant_id, scope_digest]`.

No grant twin exists for `rotate_enc_key`, `add_device`,
`remove_device`, `append_inbox`, or any lifecycle circuit. The
original `spec_version = 2` grant roster carried 30 non-pure circuits
(18 existing, 6 grant twins, 6 lifecycle circuits). The two-arm
caller/recovery snapshot carried 36. The current profiled P-256
reference carries 52; the caller restriction changes the schema and
existing gates without adding account entrypoints. Roster size is an
implementation inventory, not a requirement to deploy unsupported arms.

#### 6.2 Seam chip, ordered steps

The grant seam is written once per grantee arm as three chips
(`authenticate_grant`, `check_spend_scope`, `settle_grant`) shared by the
three twins. Every predicate is evaluated over the value the custody chip
will consume, never over an argument that merely selects it: shielded
twins invoke `held_coin(color)` before step 5 and test the returned
coin's `color` and `value`; the unshielded twin tests its `color`
argument, which is what its chip consumes. A grant twin performs, in
order:

1. **Envelope and key guard.** `k256` first asserts `envelope == 0`
   (the envelope-1 read-only rule of section 3.2, enforced in-circuit
   and not left to the authoriser), then enforces the ECDSA key
   requirements of section 3.3, including canonical coordinates, on-curve
   membership and non-identity. `jubjub` asserts `[8]pk != O`; `p256`
   enforces the same ECDSA key requirements on its curve.
2. **Identity.** `id = derive_grant_id_with_<s>(kernel.self(), pk, [envelope,] origin_hash, slot)`,
   disclosed; assert `grants.member(id)`; `g = grants.lookup(id)`. A
   lookup MUST be dominated by a membership assert, never combined with
   it in one boolean expression, since both operands are evaluated.
3. **Liveness.** Assert `g.active`, `g.epoch == device_epoch`,
   `g.gen == grant_generation`, and
   `g.scope.expires_at == 0 || kernel.blockTimeLessThan(g.scope.expires_at)`
   (the ledger read is guarded on `expires_at != 0`, so a never-expiring
   record records no time read; when the read happens its Boolean result
   is one public input the proof asserts and the node re-executes at
   admission, and the comparison argument reaches the public transcript,
   harmless because the record is public).
4. **Operation and caller.** Assert the twin's own flag, then enforce
   `caller_commit` as section 6.2.1 specifies.
5. **Object and bounds.** With `obj_color = coin.color` on the shielded
   twins and `obj_color = color` on the unshielded twin, the predicate
   set is normative and every predicate is asserted before any custody
   chip executes (GR-7); the order below is the reference order and is
   informative, with one exception: the widened sum MUST be tested
   against `cap` before it is narrowed, so that a wrap attempt fails the
   cap and never the cast (the two failures are distinct).
   - assert `derive_grant_object_commit(scope_salt, obj_color, recipient_kind, pinned_recipient, max_coin_value) == g.scope.object_commit`;
   - `p256` only: assert `derive_grant_rp_commit(scope_salt, rp_id_hash) == g.scope.rp_commit`;
   - assert `amount <= g.scope.per_call_cap`;
   - assert `derive_grant_spent_commit(scope_salt, spent_prev) == g.spent_commit`;
   - compute `wide = spent_prev + amount` in the widened type; assert
     `wide <= g.scope.cap`; then narrow to `new_spent: Uint<128>`;
   - assert `recipient_kind == 0 || (recipient_kind == <this twin's kind> && pinned_recipient == recipient.bytes)`;
   - shielded twins only: assert `coin.value <= max_coin_value` and
     `enc_pk == enc_key`, so a rotation between the grantee's signing
     and inclusion aborts the call instead of orphaning change.
6. **Challenge and verification.** `h = challenge_<operation>_with_grant_<s>(...)`
   over the preimage of section 6.3. `k256` verifies
   `secp256k1EcdsaVerify(envelope_digest(envelope, h), sig, pk)`;
   `jubjub` verifies
   `ecMulGenerator(sig_s) == ecAdd(sig_r, ecMul(pk, h as Field))` with
   the grinding rule of MIP-0013 section 5.2. Every arm also satisfies
   section 3.3's point and scalar admissibility requirements.

   `p256` MUST enforce a named, versioned bounded profile under
   MIP-0030 section 3. The expected RP hash MUST open `rp_commit`, and
   the signed origin MUST match the browser `client_id` committed by
   `origin_hash` using section 4.4's recipe. Neither policy value may be
   accepted as unchecked prover advice; checking the RP hash alone is
   insufficient. The signed type MUST be `webauthn.get`; the signed
   challenge MUST be the 43-character unpadded base64url encoding of
   the recomputed 32-byte `h`.

   This MIP requires UP and UV on every accepting WebAuthn route for
   a grant credential. Its 37-byte `authenticator_data` profile requires
   flags `0x05`, `0x0d`, or `0x1d`: UP and UV set, BS implies BE, and
   AT, ED and reserved bits clear. Cross-origin assertions are not
   accepted; each profile MUST declare and enforce its `topOrigin`
   handling. `signCount` is signed in its
   four-byte big-endian form but provides no replay protection.

   The verifier MUST hash the complete, exact signed client data:
   `z = int_be(H(authenticator_data || H(client_data_json))) mod n`.
   A full-envelope profile MUST constrain every accepted byte. A
   profile using W3C's limited-verification algorithm MUST name that
   algorithm and enforce its fields and boundaries while hashing the
   full message; it MUST state that this does not establish arbitrary
   trailing JSON syntax. Supplied hashes, offsets and logical lengths
   MUST be independently constrained, and unused buffer capacity MUST
   not enter the message. Unsupported assertions MUST be rejected,
   never truncated, reserialized or padded to fit.

   The reference `wa-json134` profile reconstructs exactly 134 signed
   bytes in `type`, `challenge`, `origin`, `crossOrigin` order, with a
   21-byte exact origin, `crossOrigin:false`, and no additional members.
   A supplied prefix alone does not establish conformance. Other named
   profiles require their own stated byte bounds and conformance vectors;
   a current reference profile is not a claim of general browser support.

   Every reachable verifier accepting the credential MUST enforce its
   authorised profile and this MIP's UP+UV policy. A single immutable
   profile, a common enforced floor, or authenticated routing may
   establish this; a weaker sibling verifier MUST NOT bypass it.
   WebAuthn device routes used to issue grants or change grant-issuing
   authority MUST also require UP+UV, so an UP-only management route
   cannot enable indirect spending. This is this MIP's policy choice;
   MIP-0030 leaves its common UV floor open.
   Changes require an authorised policy-change procedure and MUST NOT
   silently weaken an existing credential's policy.

   Both ECDSA arms enforce `1 <= r,s < n`, a non-identity verification
   result, and `R.x mod n == r`, accepting both high-S and low-S.
7. **Write-back.** `grants.insert(id, g')` where `g'` is `g` with
   `nonce + 1` and `spent_commit = derive_grant_spent_commit(scope_salt, new_spent)`,
   spelled out as a full struct literal, each field disclosed; then
   `round += 1`. `auth_nonce`, `devices`, `device_count`, and `enc_key`
   are never written by a grant twin; `enc_key` is read only for step 5.

The custody chip runs unchanged after step 7. The shielded twins then,
under a disclosed statement-level branch on whether the send produced
change, append `change_entry` through the inbox chip in the same circuit
and return the chip's result. The seam never checks `change_entry`
against the coin that send produces: it is an opaque 192-byte argument,
bound into the challenge of section 6.3 and appended verbatim, so a
grantee that seals the wrong description strands its own change and
nothing in the seam refuses it. Sealing the right description is the
grantee's obligation alone (section 6.5).

Because step 5 precedes step 6, a byte-identical resubmission is refused
at the scope predicates and never reaches the signature check: once
`nonce` has moved, the spent-commitment opening no longer matches and
either that opening or the cumulative cap dominates. A conformance suite
MUST admit any step-5 needle on a replay row rather than naming one
(Testing item 2), and exercising the signature on a replay would need an
opening that still matches, which the seam makes impossible.

This discharges the custody MIP's S5 (a seam MUST NOT be satisfiable by
circuit-unconstrained data): `origin_hash`, `slot`, `envelope`, and `pk`
are pinned by the `grant_id` membership assert; `scope_salt`,
`recipient_kind`, `pinned_recipient`, and `max_coin_value` by
`object_commit`; `rp_id_hash` by `rp_commit`; `spent_prev` by
`spent_commit`; `enc_pk` by `enc_key`. The openings are bound
transitively through `grant_id`, which determines the record and its
commitments; that this satisfies AUTH-3 is a cryptographer-review item,
and the fallback binds the openings directly at the cost of arity.

#### 6.2.1 Immediate contract caller restriction

When `g.scope.caller_commit` is zero, the callee MUST NOT read
`kernel.caller()`. The grant is unrestricted by caller; the remaining
seam predicates still apply. When it is nonzero, the callee MUST read
`kernel.caller(): Maybe<Either<ContractAddress, UserAddress>>`, require
`is_some` and the contract (`left`) arm, and recompute `caller_commit`
from that address and the grant's `scope_salt`. A mismatch aborts before
spend or settlement. A supplied address argument, witness or signature
over an address MUST NOT substitute for the kernel read.

This authenticates the ledger's **immediate claiming contract**, not the
transaction's root, a browser origin, a user address, a fee payer, or a
particular circuit. For `A -> B -> account`, a pin to B admits the call;
a pin to A does not. Absent and user-address callers fail closed. Caller
authorization is conjunctive: the correct contract without the grantee's
signature, fresh nonce, scope openings and spend bounds has no authority.

Compact 0.35.0 records `none` during top-level off-chain execution, while
balancing may add single-owner unshielded inputs from which the ledger
derives a user address. An unconditional root caller read can therefore
fail at admission. The zero branch above avoids that read entirely;
restricted grants are intended for composed contract calls. The proof
binds the read transcript; the ledger checks it against the actual call
context at admission. Injecting a caller into local execution is not
evidence of authenticated provenance.

The calling contract MUST authorize any `claimContractCall` it exposes.
A contract offering arbitrary claim tuples can lend its identity to a
call it did not execute; the [P9 provenance experiment](../../../experiments/cross-contract-calls/README.md)
demonstrates this ledger rule. A caller pin authenticates the claiming
address, not the integrity of that contract's claim policy or its code
version. Mutable caller contracts retain their address across updates.

The zero/nonzero restriction bit is public. Salting hides the pinned
address in an unused record; an exercised cross-contract call exposes
its caller/callee relationship in public transaction data. A caller pin
does not extend INV-2 to conceal that relationship or revoke a viewing
secret already handed out.

#### 6.3 Challenge preimages

Per-twin domain-separation tag, hashed from a 64-byte pad as the
authorisation MIP's amended section 5.1 requires:

`DST = H(pad(64, "midnight:account:grant:auth:<marker>v1:<operation>"))`

with `<marker>` taken from the arm of the table in section 3.4
and never derived from the wire `scheme` name: empty for `v1`, `k1:`
for `k1`, `r1:` for `r1`; and `<operation>` one of
`withdraw_unshielded`, `withdraw_shielded`,
`withdraw_shielded_to_contract`. The longest member is 63 of 64 bytes;
the 64-byte width is a normative budget on future operation names.

| Arm | Preimage (in order) |
|---|---|
| ECDSA arms (`k1`, `r1`) | `DST \|\| self \|\| x \|\| y \|\| grant_id \|\| u64(issued_at) \|\| ...args \|\| ...witness_values \|\| u64(g.nonce)` |
| JubJub arm (`v1`) | `DST \|\| self \|\| sig_r \|\| pk \|\| grant_id \|\| u64(issued_at) \|\| ...args \|\| ...witness_values \|\| u64(g.nonce) \|\| u64(grind_nonce)`, with `sig_r` and `pk` each the 64-byte `x \|\| y` form of section 3.4 |

`...args` are the operation arguments in declaration order at their
Compact widths: the unshielded twin's `color`, `u128(amount)`,
`recipient.bytes`; the shielded twins' `recipient.bytes`, `color`,
`u128(amount)`, `change_entry` (192 bytes), `enc_pk`.
`...witness_values` is empty on the unshielded twin and, on the shielded
twins, the `QualifiedShieldedCoinInfo` returned by `held_coin`,
flattened in declaration order: `nonce` (32 bytes), `color` (32 bytes),
`u128(value)`, `u64(mt_index)` (AUTH-10). `g.nonce` and `g.issued_at`
are read from the record inside the circuit. ECDSA preimages exclude
signature material (MIP-0013 section 5); the JubJub arm grinds `grind_nonce` until
the little-endian value of `h` is below the subgroup order (the
authorisation MIP section 5.2). The shielded ECDSA challenge has
thirteen declared tuple members (the `QualifiedShieldedCoinInfo` is one
member), sixteen encoded elements, and 568 preimage bytes; the
unshielded one 256 bytes. Both compile and hash as the raw
concatenation, so no pre-hashing of the operation arguments is needed
on the ECDSA arms. The JubJub shielded challenge has fourteen declared
members, nineteen encoded elements, and 640 preimage bytes, and compiles
and hashes the same way.

Preimage widths of the compiled encoding, in bytes, for the three grant
challenges and the three lifecycle challenges of section 6.1:

| Recipe | `k1` | `v1` |
|---|---|---|
| `withdraw_unshielded` grant challenge | 256 | 328 |
| `withdraw_shielded` grant challenge | 568 | 640 |
| `withdraw_shielded_to_contract` grant challenge | 568 | 640 |
| `issue_grant` | 200 | 272 |
| `revoke_grant` | 168 | 240 |
| `revoke_all_grants` | 136 | 208 |

Every `v1` recipe is its `k1` twin plus 72 bytes: the 64-byte `sig_r`
element and the 8-byte grinding nonce. A point element counts as one
declared member and two encoded elements, and the qualified coin as one
member and four elements.

#### 6.4 Signing (grantee side)

A grantee needs only curve arithmetic and SHA-256. In order:

1. read `grants[grant_id]`, `device_epoch`, `grant_generation`, and
   `enc_key` from chain state; take `nonce` and `issued_at` from the
   record and confirm it is live;
2. on the shielded twins, resolve the qualified coin the call will
   consume from the local coin store and precompute `change_entry`
   encrypted to the `enc_key` just read, which becomes `enc_pk`;
3. build the preimage of section 6.3 with those values and the openings
   of section 6.2 step 5;
4. on the `v1` arm, grind `grind_nonce` until the little-endian value of
   `h` is below the subgroup order;
5. sign in the arm's form of section 6.2 step 6.

A grantee MUST re-read `nonce` and `enc_key` and re-sign if its
transaction is not included, and MUST persist `scope_salt`, the
returned scope, and its running `spent` alongside the key.

On the `v1` arm an authorisation that does not match the call it is
presented with produces one of two refusals, and a conforming client
expects both. The seam casts the challenge to a field element before
evaluating the Schnorr equation, and a challenge the signer did not
grind is below the field modulus only about 45 per cent of the time, so
about 55 per cent of such calls abort at the cast's range check and the
rest at `invalid grant signature`. Which one fires is a property of the
challenge bytes and not of the seam, so a conformance suite MUST admit
either needle on that arm. The ECDSA arms do not show it, since their
challenge is a message and is never cast.

Every in-circuit refusal of section 6.2 arrives at build time in one
shape, `Unexpected error executing scoped transaction '<unnamed>':
Error: failed assert: <needle>`, where the needle is the assert's own
message as this MIP names it; no refusal text outside the specification
was observed on node. An implementer matches on the needle and MUST NOT
take the wrapper as the classifier, because the submission phase carries
a different one (`Unexpected error submitting ...`), which is what a
proving failure or a mempool refusal produces, and because refusals also
arrive from the runtime (section 3.3), from the client's encoder
(section 3.4), and from the node at admission.

Expiry surfaces as one of two signals, both leaving state untouched and
the signed challenge valid: a build-time abort (`failed assert: grant
expired`) when the grantee's own clock is at or past `expires_at`, and
a mempool refusal when the network's admission-time block time is (the
node answers `Invalid Transaction: Custom error: 104` on a transcript
read mismatch, which the client submission layer surfaces as a
`SubmissionError` whose reason is visible only in the RPC log line). On
either signal the grantee MUST re-read the grant record and the chain
head, and MUST re-sign only if `expires_at` is `0` or exceeds the head's
block time by more than the inclusion margin of section 5.1, which
includes the call's own proving time: a k = 17 shielded twin measured
14.5 s from build to the node's refusal, so a client that budgets only
the 8.2 to 10.2 s of section 5.1 will sign grants that cannot land.
Otherwise it treats the grant as expired and requests a new one.
Retrying an admission refusal without re-reading is pointless, since
the network's time only advances. A grantee SHOULD NOT take an
indexer's latest-block timestamp as the head time (measured two blocks
behind the node on the reference stack).

#### 6.5 Grantee private state and coin selection

A grantee spending shielded value maintains a wallet-local coin store per
the custody MIP section 6.5 (it holds the viewing secret, since shielded
scopes imply `read`) and reconstructs coin descriptions and `mt_index`
values from the inbox and chain data before it can prove. Owner and
grantee select coins independently; clients SHOULD apply a deterministic
rule (the smallest coin of the color that covers `amount`, ties broken
by `mt_index`). The failure modes are a fee-wasting race when both
submit and, when both select the same coin, a refusal of the loser by
the node as a double spend; a mis-spend is not one of them, and neither
is a proving failure, since proving a spend consults the commitment tree
and not the nullifier set, so the loser's call still proves against the
post-spend state (INV-5).

The change description is precomputable before proving, and is measured
so. The standard library derives both output nonces from the input
coin's nonce, the sent coin under the kernel tag
`midnight:kernel:nonce_evolve` and the change coin under
`midnight:kernel:nonce_evolve/2`, each the transient hash of the tag and
the degraded input nonce, upgraded from transient. A grantee therefore
computes the change coin, seals it as `change_entry`, and signs over it
before the call is built. Ten of ten predictions matched the coin the
circuit returned, on nonce and value alike, and a composed
contract-recipient call additionally matched the sent coin's nonce,
which is the pair a composing client needs to build the payee's claim in
the same transaction. Nothing in the seam checks the seal (section 6.2),
so a grantee that seals the wrong description strands its own change.

One qualification stands between that rule and a working client. The
change output is not reliably the last commitment of the spend
transaction's window, and a contract-owned deposit commitment is not
reliably the first of its own, so the qualified coin's `mt_index` is
resolved by trying candidates, and every retry of a signed call costs a
fresh signature as well as a proof, because the qualified coin is in the
challenge (section 6.3). Measured over one run: of eleven spends that
resolved their index by signing and submitting a candidate, six needed a
second candidate and none needed a third; of eight resolved by
prove-only trials, which submit nothing, four needed a second trial. A
grantee SHOULD therefore resolve `mt_index` by prove-only trials before
it signs. A wrong candidate is not self-describing and MUST NOT be read
as a diagnosis of the index: the call fails at the prover before
submission and carries the prover's own error text, or aborts in circuit
execution with a bare `unreachable`, and nothing reaches the node in
either case, which is what makes prove-only resolution safe.

#### 6.6 Disclosure

Disclosed per grant call: `grant_id` (lookup and write-back), the
rewritten `nonce` and `spent_commit`, the `expires_at` argument of the
kernel comparison (no read is recorded when it is `0`), and what the
custody chip already discloses
(`color`, `amount`, and `recipient` on the unshielded twin, whose
balances are public; the contract address of a contract-recipient
output; the change-entry ciphertext on a shielded twin). Not disclosed:
`pk`, `origin_hash`, `slot`, `scope_salt`, `rp_id_hash`, the openings,
`spent_prev`, `enc_pk`, the signature, the qualified coin, and on
shielded twins `color`, `amount`, and a `ZswapCoinPublicKey` recipient,
which feed the send path's commitments exactly as under a device twin.

The record publishes the operation flags, `read`, `per_call_cap`, `cap`,
`expires_at`, `read_pk_hash`, `nonce` (the call count), and `active`;
it commits to `color`, the recipient pin, `max_coin_value`, the dApp
host, and `spent`. `grant_id` is a stable per-grant pseudonym disclosed
on every call, unequal across accounts because `kernel.self()` is in the
preimage. AUTH-9 continues to protect device handles; grant disclosure
and the limits of its hiding argument are specified here and in section 12.

#### 6.7 Cost and deploy budget

Over the corresponding device twin a grant twin replaces two
device-entry hashes with one identity hash and three commitment hashes,
and adds one map lookup and insert, one widened comparison, one kernel
comparison, and on shielded twins one inbox insert. Measured on the
historical reference contract at `spec_version = 2` over its thirty-circuit
roster, both device arms and both grantee arms. These are not current v3
deploy budgets; current caller and P-256 inventories are linked in
Implementation. Prover keys are rounded from the measured bytes:

| Circuit | k | Rows | Prover key |
|---|---|---|---|
| `deposit_unshielded` | 9 | 311 | 0.4 MB |
| `deposit_shielded` | 13 | 6,484 | 11 MB |
| `activate_initial_device_with_jubjub` | 14 | 13,412 | 25 MB |
| `add_device_with_jubjub` | 15 | 26,771 | 49 MB |
| `remove_device_with_jubjub` | 15 | 32,730 | 49 MB |
| `rotate_enc_key_with_jubjub` | 15 | 26,725 | 49 MB |
| `append_inbox_with_jubjub` | 16 | 32,790 | 99 MB |
| `withdraw_unshielded_with_jubjub` | 15 | 28,877 | 49 MB |
| `withdraw_shielded_with_jubjub` | 16 | 50,055 | 99 MB |
| `withdraw_shielded_to_contract_with_jubjub` | 16 | 55,758 | 99 MB |
| `activate_initial_device_with_k256` | 14 | 14,094 | 29 MB |
| `add_device_with_k256` | 16 | 58,897 | 117 MB |
| `remove_device_with_k256` | 16 | 65,404 | 117 MB |
| `rotate_enc_key_with_k256` | 16 | 58,851 | 117 MB |
| `append_inbox_with_k256` | 16 | 64,924 | 117 MB |
| `withdraw_unshielded_with_k256` | 16 | 61,003 | 117 MB |
| `withdraw_shielded_with_k256` | 17 | 74,587 | 235 MB |
| `withdraw_shielded_to_contract_with_k256` | 17 | 80,290 | 235 MB |
| `issue_grant_with_jubjub` | 16 | 52,227 | 99 MB |
| `revoke_grant_with_jubjub` | 15 | 26,871 | 49 MB |
| `revoke_all_grants_with_jubjub` | 15 | 26,645 | 49 MB |
| `withdraw_unshielded_with_grant_jubjub` | 16 | 38,514 | 99 MB |
| `withdraw_shielded_with_grant_jubjub` | 17 | 66,014 | 197 MB |
| `withdraw_shielded_to_contract_with_grant_jubjub` | 17 | 71,717 | 197 MB |
| `issue_grant_with_k256` | 17 | 78,604 | 235 MB |
| `revoke_grant_with_k256` | 16 | 58,997 | 117 MB |
| `revoke_all_grants_with_k256` | 16 | 58,771 | 117 MB |
| `withdraw_unshielded_with_grant_k256` | 17 | 64,355 | 235 MB |
| `withdraw_shielded_with_grant_k256` | 17 | 91,865 | 235 MB |
| `withdraw_shielded_to_contract_with_grant_k256` | 17 | 97,568 | 235 MB |

The seam's cost over the corresponding device twin is constant within an
arm and a shape and differs across arms: on the k256 arm each shielded
twin costs 17,278 rows more than its device twin and the unshielded twin
3,352 more; on the jubjub arm each shielded twin costs 15,959 more and
the unshielded twin 9,637 more. The shielded figures carry the inbox
insert for the change entry, which no device twin performs. The seam
pushes four of the six twins up one k, the k256 unshielded twin to 17,
the jubjub unshielded twin to 16, and both jubjub shielded twins to 17,
while the two k256 shielded twins stay at their device twins' k=17.
Lifecycle is much cheaper on the normative arm: issuance, the most
expensive lifecycle circuit on both arms, is 52,227 rows at k=16 on
jubjub against 78,604 at k=17 on k256, and the two revocation circuits
about 26,800 rows at k=15 against about 58,900 at k=16.

The envelope is not a circuit-shape parameter (both digests are computed
on every call), so k does not vary by envelope. k is not a pure function
of the row count (device circuits of 64,924 and 65,404 rows fit k=16
while the k256 unshielded grant twin at 64,355 needs k=17), and the
prover key is not a function of k alone (at k=17 a jubjub circuit is
about 197 MB and a k256 circuit about 235 MB; at k=16 the split is about
99 MB against 117 MB), so an implementation quotes k, rows, and key
sizes as measured and never predicts one from another. These measurements predate
the native p256 twins. Their later compiled row/key inventory is in
`contract/P256-MEASUREMENTS.md`; it does not supply grant-spend proving
times on node.

Proving times, measured at the proof provider and so excluding building,
balancing, and submission, on the reference stack (node 2.1.0, proof
server 9.0.0-rc.6, indexer 4.4.0-rc.2, a single-authority localnet with
6 s blocks, one machine, one run): about 3.4 s at k=15, 5.6 to 8.0 s at
k=16, and 11.2 to 17.4 s at k=17; 15.1 s for a cross-contract pair of a
k=17 grant twin with a k=13 deposit proved in one call, and 19.9 and
30.8 s for the two proofs of a composed pair of k=17 grant twins, which
is dearer per proof than either twin alone. A grant spend and an
issuance cost the same order on the same arm, and the jubjub arm is the
cheaper one at every k the two share. The figures are one machine's,
and the proof server was the run's least stable component at k=17.
One of the six grant twins carries no on-node timing at all, the jubjub
contract-recipient twin, which the run never called. The two unshielded
twins were timed in a later run on the same stack, on a non-native
color, at 5.6 to 6.8 s for the jubjub twin at k=16 over four proofs and
17.2 s for the k256 twin at k=17 over one: what the reference localnet
refuses is the funding leg, a small contract call carrying a NIGHT
unshielded input that balancing pairs with a NIGHT change output, and
not a contract call paired with an unshielded offer as such
(`evidence/grants-e4-proving-*.json` and `evidence/dismiss-cost.json` of
the reference implementation).

Verifier keys depend on the circuit's shape and not on k: 2,745 bytes
for every k256 circuit and 2,313 for every jubjub circuit (the deposits
2,121 and 1,353). The historical thirty-circuit v2 roster carries
74,286 verifier bytes against a 50,000-byte per-block write limit, with
about 9,138 bytes of deploy overhead beyond the keys; waves are
mandatory independently of the grant circuits, since the eighteen
pre-existing ones alone price a deploy at 53,076 bytes written. A
one-transaction deploy of the whole roster is therefore refused before
any transaction exists and the roster reaches a live account in **three
waves**: the deploy, carrying the two deposits and the eight circuits of
the born arm (25,434 verifier bytes), then two hand-built maintenance
updates (23,994 and 24,858 verifier bytes), the last of which retires
the maintenance authority. Each update advances the maintenance
authority counter by exactly one, which a client reads from chain rather
than assuming.

The per-update ceiling is network-defined and this MIP bounds it only by
measurement: on node 2.1.0 an update of 29,484 verifier bytes is
accepted and one of 32,229 is refused, so the ceiling lies in that
interval, closed to one verifier key. An implementation MUST pack well
below it. The reference budget is 25,000 verifier bytes, the largest
accepted payload less about 15 per cent, the margin covering the Dust
spend the wallet adds when it balances the update, block fullness at
submission, and the per-block fee-price adjustment. A budget under the
ceiling does not by itself fix the wave count, since a greedy packer
over one key order lands the same roster in four waves at 24,000 and in
three at 25,000.

The two refusals are different mechanisms at different points, and a
specification that says an update must fit a block MUST say which one it
means. The all-operations deploy is refused client-side by the fee
computation, before a transaction exists. An over-large maintenance
update is refused by the node at admission (`Invalid Transaction:
Transaction would exhaust the block limits`) with nothing client-side
objecting first: the client priced every refused payload without
complaint, so an implementer gets no warning before submission and a
refused update leaves the authority counter untouched.

The grant circuits MUST be part of the `spec_version = 3` deploy wave
plan, and maintenance-authority retirement MUST follow the last grant
wave: a retired account can never receive a future arm's circuits, so
an account deployed without the grant circuits and then retired can
never gain grants and must migrate.
Pure circuits add no keys.

### 7. Lifecycle

#### 7.1 Circuit semantics

All three lifecycle circuits are device-gated through the unchanged
device seam, which advances `auth_nonce` and `round` before the body
runs; every `Map.lookup` is dominated by a `member` branch (section 6.2
step 2). The bodies compile and execute off-node on both device arms;
on node, issuance is exercised from both, and revocation and the kill
from the jubjub arm (Implementation).

`issue_grant(grant_id, <plaintext scope>)`:

1. Disclose `grant_id` as `id`.
2. If `grants.member(id)`: read the old record and assert that it is not
   live, that is,
   `!old.active || old.epoch != device_epoch || old.gen != grant_generation`
   ("grant already active"). Issue over an absent id, a tombstone, or an
   inert record succeeds; `issue_grant` does not evaluate expiry, so an
   expired but unrevoked record is live for this test (section 7.2).
3. Assert the issue rules of section 5.1 in the order listed there.
4. Write the whole record: `epoch = device_epoch`,
   `gen = grant_generation`, `issued_at = auth_nonce` (already advanced
   by the device seam, so unique per issuance), `nonce = 0`,
   `spent_commit = derive_grant_spent_commit(scope_salt, 0)`, window
   fields zero, `active = true`, and the scope with
   `object_commit = derive_grant_object_commit(scope_salt, color, recipient_kind, recipient, max_coin_value)`
   and `rp_commit = derive_grant_rp_commit(scope_salt, rp_id_hash)`,
   `caller_commit = derive_grant_caller_commit(scope_salt, caller)`, the
   clear fields disclosed. `color`, `recipient`, `max_coin_value`, and
   `rp_id_hash` and the plaintext caller pin are never disclosed at
   issuance; only their commitments are.

`revoke_grant(grant_id)`: disclose `grant_id`; assert `grants.member(id)`
("unknown grant"); assert `active` ("grant not live"); rewrite the record
unchanged except `active = false`.

`revoke_all_grants()`: `grant_generation += 1`; then MAY clear the map
for state relief through the `Map` reset primitive, which the reference
does (evidenced: the map is empty afterwards). With the reset, records
are absent and a later grant twin fails at the membership assert
("unknown grant") rather than at the generation check; tombstones and
enumerability go with them. The generation check remains the safety for
an implementation that omits the reset.

The reference contract uses no spread or update syntax, so a conforming
implementation spells every struct literal out in full.

#### 7.2 State transitions

| From | Event | To | Written |
|---|---|---|---|
| absent, tombstone, or inert | `issue_grant` | active (epoch `e`, gen `g`, `issued_at` `i`) | whole record |
| active | grant twin succeeds | active | `nonce + 1`, `spent_commit` |
| active or expired | `revoke_grant` | tombstone | `active = false` |
| active | block time reaches `expires_at` | expired (lazy; record unchanged) | nothing |
| any | recovery bumps `device_epoch` | inert by epoch | nothing (MAY clear the map) |
| any | `revoke_all_grants` bumps `grant_generation` | inert by generation, or absent where the implementation clears the map (the reference does) | `grant_generation`; MAY clear the map |
| tombstone, expired, inert, or absent | grant twin | abort, no state change (an absent record fails at the membership assert) | nothing |

Rules: every transition into active is device-gated and carries fresh
consent over the whole scope; no in-place modification exists;
modification, and renewal of an expired record, is `revoke_grant` then
`issue_grant`, composable in one transaction (section 7.4), or
`issue_grant` under a fresh `slot`; only `revoke_all_grants` and
recovery are O(1) over the register; `remove_device` has no edge. Re-
issue yields a new incarnation with `nonce = 0`, a fresh `spent_commit`,
a fresh `scope_salt`, and a fresh `issued_at`; dApps SHOULD choose a
fresh `slot` where one is free. Tombstones are kept for enumerability;
pruning is a revision item. Counters are `Uint<32>` or wider. No
`issued_by` is recorded, since it would disclose a stable device
identifier contrary to AUTH-9.

#### 7.3 Recovery and device removal

The recovery seam of the authorisation MIP section 8 is unchanged in its
obligation: it bumps `device_epoch`, and the equality check of section
6.2 step 3 makes every grant inert. Clearing `grants` at recovery is
hygiene a recovery circuit MAY perform; safety does not depend on it.

Single-device removal does not cascade to grants in the contract. On
removing a device for suspected compromise the owner's client MUST call
`revoke_all_grants` in the same transaction or immediately after,
because a briefly compromised device can have issued unbounded grants to
keys it controls (R19).

#### 7.4 Composition in one transaction

A grant call MAY be composed with other calls of the same or other
contracts, including a device call of the same account. Each grant call
consumes exactly one `nonce` increment, so `N` composed calls under one
grant need `N` consecutive counters and `N` signatures; the cumulative
cap is enforced per call against the record as it stands at that call's
execution, so a transaction cannot exceed `cap` in aggregate; the
per-call cap is per circuit invocation, never per transaction; the
recipient pin is evaluated per call and survives grafting.

Reordering composed calls does not invalidate their signatures: each
stays a valid signature over its own challenge. What fails is the
transcript, because the out-of-order call records reads of a record
state that does not hold when it executes, and the node refuses the
transaction at admission on a read mismatch. The two produce different
operator-visible behaviour, and an implementation MUST expect the
transcript refusal rather than a signature failure.

Two calls on one contract in one transaction are measured (a revocation
composed with a re-issue over one `grant_id`, a batch of two issuances,
and two grant calls under one grant at consecutive nonces, the last
appending both change entries in the same transaction). Three mechanics
are properties of the ledger and of the client software development kit
rather than of the seam, and a composing client meets every one:

- the second call MUST be built against the state the first produces.
  Two calls built independently read the same pre-state and record the
  same `auth_nonce`, or the same grant `nonce`, and the node refuses the
  pair with `Transcript(Execution(ReadMismatch ...))`;
- the second call's shielded offer MUST travel with its intent. An
  intent carries contract actions and unshielded offers only, so a
  contract call's shielded inputs and outputs live on the transaction;
  grafting the intent alone yields a transcript claiming nullifiers no
  offer carries, refused before execution with
  `Malformed(EffectsCheck(NullifiersNeqClaimedNullifiers))`;
- which transcript section a call's coins land in is not stable between
  runs. A guaranteed offer merges and leaves the intent free to be
  placed at any segment, while a fallible offer's proofs are bound to
  their own segment and re-keying it is refused with
  `Malformed(Zswap(InvalidProof))`, so a call whose coins are fallible
  keeps its own segment and the relative order of the two calls is
  whatever the segment draw gave.

A tool that reads transcripts MUST read the guaranteed and the fallible
section alike, since the same circuit was observed in each on
consecutive transactions.

#### 7.5 Owner-side records

Because the record holds no key, origin, or scheme, a conforming owner
client MUST maintain a grant roster of `grant_id`, `client_id`, grantee
key, `[envelope,]` `slot`, `scope_salt`, the approved plaintext scope,
and every re-seal, rebuildable from the issuing device, and MUST
reconcile the live ids in `grants` against it on every account view,
surfacing any unrecognised live id as possible compromise with
`revoke_all_grants` as the remedy. An owner holding only chain state can
revoke by id and revoke all but cannot attribute. An optional
`origin_hint`, the origin sealed under the account encryption secret and
written at issue, is a MAY; it discloses the account's other connections
to every read-granted party.

### 8. Read-scope delegation

Read is the custody MIP's viewing capability (its R9), not a circuit. The
on-chain `read` flag is declarative: it lets the owner enumerate which
grants hold the viewing secret, anchors the consent wording to a stored
field, and lets the shielded-implies-read rule be asserted at issue. The
ledger enforces spend scope and not read scope.

1. **Delegate key.** The request carries `read_pk`, a 32-byte X25519
   public key, hex, in the canonical little-endian encoding of a
   coordinate below `p` with the top bit clear; a non-canonical
   encoding, any of the twelve known low-order points, or an all-zero
   shared secret (the RFC 7748 contributory check) is `invalid_request`.
   `read_pk_hash = H(read_pk)` is stored in the scope. Browser grantees
   derive the secret from the WebAuthn pseudo-random function (PRF)
   extension of their per-origin passkey, pending PRF availability in
   target browsers:
   `sk = clamp(PRF(first = "midnight:account:grant:read:v1" || account))`,
   with `account` the 32-byte contract address and `clamp` the RFC 7748
   section 5 scalar clamp; `read_pk = X25519(sk, 9)`. The key is per
   origin and per account, so `read_pk_hash` correlates nothing across
   accounts, and any client holding the same credential derives the same
   key (section 9.8). The PRF evaluation is a separate `credentials.get`
   from any grant-call assertion (an extension-bearing assertion exceeds
   the 37 bytes the r1 gate verifies); its output never enters a circuit.
   Agents use a vault key.
2. **Rotate before share.** Before issuing any grant with `read = true`
   the authoriser MUST perform `rotate_enc_key` and re-encrypt live
   holdings into fresh inbox entries (custody MIP section 6.7), so the
   delegated secret reads current and future holdings but not spent
   history. Every rotation blinds every delegate holding the previous
   secret, so on any rotation performed under this MIP the authoriser
   MUST either re-seal the new secret to every other live grant whose
   `read_pk_hash` is non-zero, or mark those grants in the owner's roster
   as requiring reconnection and surface it. A grantee whose pending call
   encrypted change to the previous key aborts at section 6.2 step 5 and
   re-signs.
3. **Sealing.** `shared = X25519(eph_sk, read_pk)`;
   `key = HKDF-SHA256(ikm = shared, salt = empty, info = "midnight:account:grant:view:v1" || grant_id, L = 32)`;
   authenticated encryption with associated data (AEAD) AES-256-GCM with
   a random 12-byte nonce and associated data
   `version || suite || eph_pk || grant_id || account`. The container,
   **GrantViewSeal v1**, is 94 bytes with its own version and suite
   numbering space; readers MUST check the length before the version, so
   an InboxEntry is never parsed as a seal:

   | Offset | Length | Field |
   |---|---|---|
   | 0 | 1 | `version` = `0x01` |
   | 1 | 1 | `suite` = `0x01` (X25519 + HKDF-SHA256 + AES-256-GCM) |
   | 2 | 32 | ephemeral X25519 public key |
   | 34 | 12 | AEAD nonce |
   | 46 | 16 | AEAD tag |
   | 62 | 32 | ciphertext of the account encryption secret |

   It is delivered base64url in the response as `view`, never written
   to chain, never in a query component. The dApp MUST verify that
   `X25519(secret, 9)` equals the account's `enc_key` cell before
   treating the capability as live, and treat a mismatch as a faulty or
   compromised authoriser.
4. **Reading.** The dApp enumerates the account's contract actions and
   decrypts locally (custody MIP section 6.5), verifying every candidate
   coin by recomputing its commitment against chain data before treating
   the entry as authentic and quarantining the rest; no indexer receives
   the secret. This is a MUST for every inbox reader: `deposit_shielded`
   is permissionless, so a holder of the viewing secret can write
   plausible but unverifiable entries, which a conforming reader treats
   as inert noise (the companion note to the custody MIP's R9).
5. **Re-seal.** Any re-seal (silent reconnect, section 9.8; the rotation
   obligation of item 2) MUST target the key whose hash is
   `read_pk_hash`; a differing `read_pk` is `invalid_request`. Every
   re-seal is recorded in the owner's roster.
6. **Revocation.** On revocation of any grant with `read = true` the
   owner's client MUST `rotate_enc_key` and re-encrypt live holdings,
   with the re-seal obligation of item 2 toward the remaining delegates;
   the revoked delegate keeps what it already read (custody MIP S2); the
   consent screen says so in advance.
7. **Authoriser capability.** An authoriser holding only a device key
   and not the viewing secret MUST refuse a read request with
   `read_unavailable`.

`read` is total, and this is the dominant residual risk of the standard:
it delivers the account encryption secret itself, which decrypts the
whole inbox across every color, current and future, until rotation; no
schema field bounds it, every read-granted party holds the same secret,
and issuing or revoking any read grant blinds every other until
re-sealed. Spend without full disclosure awaits the custody MIP's R9
successor. Read-only grants are first-class; unshielded balances are
public and need no capability.

### 9. GrantRequest and the redirect binding

#### 9.1 The object

One transport-independent `GrantRequest` object, a JSON object in any
serialisation the dApp chooses, carried as `base64url(UTF-8 JSON)` in a
`request` parameter. No canonical form exists and none is computed:
what is signed is the parameter value as transmitted (section 9.2).
`Uint<128>` values are decimal strings. Field names follow RFC 6749
where a parameter has an OAuth analogue. The proof of possession is not
a member of the object; it is a separate `Proof` object carried beside
it in a `proof` parameter.

```json
{
  "v": 1,
  "chain": "midnight:mainnet",
  "aud": "https://passport.example",
  "iat": 1790000000,
  "exp": 1790000600,
  "client_id": "https://bank.example",
  "redirect_uri": "https://bank.example/passport/callback",
  "account": "<64 hex, MUST when known>",
  "grantee": { "scheme": "ecdsa_secp256k1_sha256", "envelope": 0, "pk": "<128 hex, x_le32 || y_le32>" },
  "grants": [
    {
      "slot": 0,
      "scope": ["midnight:account:read", "midnight:account:withdraw_shielded"],
      "bounds": {
        "color": "<64 hex>",
        "per_call_cap": "1000000",
        "cap": "5000000",
        "max_coin_value": "5000000",
        "expires_at": 1792600000,
        "recipient": null
      }
    }
  ],
  "read_pk": "<64 hex X25519>",
  "state": "<base64url, at least 128 bits>",
  "nonce": "<64 hex, 32 bytes>"
}
```

`iat`, `exp`, and `bounds.expires_at` are all whole seconds since the
UNIX epoch; the first two are evaluated by the authoriser and the third
by the ledger (section 5.1). `recipient`, when
present, is `{ "kind": "user" | "zswap" | "contract", "value": "<64 hex>" }`
mapping to `recipient_kind` `1`, `2`, and `3`. `grants` carries one or more
elements for the single grantee key, each with a distinct `slot`: one
ceremony, one consent screen rendering every element, one device
signature per record; the authoriser composes the `N` `issue_grant`
calls into one transaction or submits them in sequence.

| Member | Required | Rule |
|---|---|---|
| `v` | MUST | integer `1`; unknown is `unsupported_version` |
| `chain` | MUST | CAIP-2 `midnight:<reference>` (MIP-0008); a legacy bare network id is accepted and canonicalised |
| `aud` | MUST | the authoriser origin, exact string match |
| `iat`, `exp` | MUST | `exp - iat` at most 600; outside `[iat, exp]` is `invalid_request` |
| `client_id` | MUST | normalised per section 4.4; `https` (plain `http` only for `localhost`); `rdns:`, `mais:`, `self:` are `invalid_request` in the browser binding |
| `redirect_uri` | MUST in the browser binding; MUST be absent otherwise | absolute `https` URI whose origin equals `client_id`, no fragment; exact string match on return (RFC 9700 section 4.1.3); loopback exception only for native clients |
| `account` | MUST when the dApp knows it | contract address hex; when absent the user chooses and the consent screen says so |
| `grantee` | MUST | `{scheme, [envelope,] pk}` per section 3; `envelope` for `ecdsa_secp256k1_sha256` only |
| `grants` | MUST | non-empty array of `{slot, scope, bounds}`; distinct slots; unknown strings are `invalid_scope` |
| `bounds` | MUST when any withdraw string; MUST be absent otherwise | `color`, `per_call_cap`, `cap`, `max_coin_value`, `expires_at` (`0` = never, explicit), `recipient` (null or typed), optional `caller` (64 lowercase hex contract-address bytes, no prefix; omission means unrestricted); a read-only element is issued with the all-zero object and caps of section 5.1 rule 7 |
| `read_pk` | MUST when `read` is requested or implied | per section 8 item 1 |
| `state` | MUST in the browser binding; MUST be absent otherwise | opaque, at least 128 bits of entropy, at most 512 characters, bound to the dApp's browser session, echoed verbatim |
| `nonce` | MUST | 32 random bytes, hex; one-time within the validity window |

Every member listed is the complete member set: an object carrying an
unknown member, a duplicated member, or a member of the wrong JSON type
is `invalid_request`. Parsing happens after hashing (section 9.4).

**The `Proof` object.** The possession and origin proof over
`request_digest` (section 9.2), a JSON object carried as
`base64url(UTF-8 JSON)` in the `proof` parameter, subject to the same
member rules. It is validated, never hashed or signed:

```json
{
  "type": "webauthn",
  "client_data_json": "<base64url>",
  "authenticator_data": "<base64url>",
  "signature": "<base64url>",
  "credential_pk": "<128 hex, companion passkey only>",
  "key_signature": "<128 hex (k1) or 192 hex (v1), software and wallet-provider grantees only>"
}
```

| Member | Required | Rule |
|---|---|---|
| `type` | MUST | `webauthn` or `signature`; `signature` in the browser binding is `invalid_proof` |
| `client_data_json`, `authenticator_data`, `signature` | MUST when `type` is `webauthn`; MUST be absent otherwise | the assertion the authenticator returned, verified per section 9.2 |
| `credential_pk` | MUST for a software or wallet-provider grantee under `webauthn`; MUST be absent for an `r1` grantee, whose assertion is by `grantee.pk` itself, and under `signature` | the companion passkey |
| `key_signature` | MUST for every grantee other than `r1`; MUST be absent for `r1` | a signature by `grantee.pk` over `request_digest` in the arm's off-chain form |

Encodings of the `Proof` members:

| Member | Encoding |
|---|---|
| `client_data_json`, `authenticator_data` | base64url of the raw bytes the authenticator returned |
| `signature` | base64url of the authenticator's ASN.1 DER `ECDSA-Sig-Value`, both `s` forms accepted |
| `credential_pk` | 128 lowercase hex, affine `x \|\| y`, each coordinate a 32-byte little-endian integer (section 3.4) |
| `key_signature` | the arm's off-chain form of section 3.4, lowercase hex: `k1` `r \|\| s`, 128 hex; `v1` `R.x \|\| R.y \|\| s`, 192 hex |

#### 9.2 Request digest and proof rules

`request_digest = H(pad(64, "midnight:account:grant:request:v1") || request_param_bytes)`,

where `request_param_bytes` is the ASCII bytes of the `request`
parameter value exactly as received: the base64url text, not the
decoded JSON. The dApp signs the very string it places in the URL, and
an authoriser MUST hash the received parameter value before any
decoding and MUST NOT re-serialise, re-encode, or otherwise transform it
first. There is no canonicalisation step: two serialisations of the
same object are two different requests, and a request whose bytes were
altered in transit, by whitespace, key order, or percent-encoding,
fails its proof and nothing else. A base64url value MUST consist of the
RFC 4648 section 5 alphabet only, without padding, so no
percent-encoding applies to it and the bytes the dApp wrote are the
bytes the authoriser receives. The `Proof` object is outside the signed
bytes; a change to it fails verification without changing
`request_digest`. In a WebAuthn assertion the challenge is
`request_digest` itself, so `clientDataJSON.challenge` is the 43-byte
unpadded `base64url(request_digest)`.

Proof rules in the browser binding:

- **`ecdsa_secp256r1_webauthn` grantee.** `proof.type = "webauthn"`: an
  assertion by the grantee credential with `challenge = request_digest`,
  requested without extensions. The authoriser validates `grantee.pk`
  per section 3.3, verifies the signature under it,
  `clientDataJSON.type == "webauthn.get"`,
  `clientDataJSON.challenge == base64url(request_digest)`,
  `clientDataJSON.origin == client_id`, and
  `rpIdHash == H(host(client_id))`, and takes `rpIdHash` as
  `rp_id_hash`.
- **Software grantee** (`schnorr_jubjub`, or `ecdsa_secp256k1_sha256`
  envelope `0`). `proof.type = "webauthn"` by a companion credential
  created on the dApp origin, verified under `proof.credential_pk` with
  the same `clientDataJSON` checks, plus `proof.key_signature`, a
  signature by `grantee.pk` over `request_digest` in the arm's off-chain
  form. The assertion attests the origin and binds the software key
  through the digest; the key signature proves possession.
- **Wallet-provider grantee** (`ecdsa_secp256k1_sha256` envelope `1`).
  As the software grantee, with `key_signature` obtained through the
  connector's `signData` and verified over
  `envelope_digest(1, request_digest)`; only `midnight:account:read`
  may be requested.

Non-browser bindings (agents, `self:` delegates) carry
`proof.type = "signature"` with `key_signature` only; the identity is
displayed as operator-asserted or as the owner's own label.
`proof.type = "signature"` in the browser binding is `invalid_proof`.
In every binding the same two strings travel, `request` and `proof`,
and the signed bytes are the `request` string as transmitted over that
channel.

**The binding is a property of the transport.** A request received by
top-level navigation MUST get the browser binding regardless of the
`client_id` form: the possession proof MUST be a WebAuthn assertion on
the dApp origin, and `rdns:`, `mais:`, and `self:` clients MUST be
rejected with `invalid_request`. Non-browser bindings MUST use a
non-redirect delivery, MUST NOT accept a `redirect_uri`, and admit
key-only proofs because an operator or the owner approves in person.

**What the origin proof delivers.** A verified assertion proves that a
credential registered on the requesting origin vouches for the grantee
key. It does not prove that the request came from that origin's
authentic front-end: any registered user of the dApp, or a script
injected into it, can produce a valid request naming the origin and its
own key. The consent screen naming the proven origin and the owner's
roster are the remaining barriers; a dApp MAY publish an allow-list of
its grantee keys, and `.well-known` client metadata is the recorded
extension.

#### 9.3 Browser binding: request delivery

Top-level GET navigation to
`https://<authoriser>/grant#request=<base64url(UTF-8 JSON GrantRequest)>&proof=<base64url(UTF-8 JSON Proof)>`.
Both travel in the fragment, never in the query component, as two
form-encoded parameters in either order. The dApp serialises the
`GrantRequest` once, base64url-encodes that serialisation, signs the
resulting string (section 9.2), and places that same string in the URL
unchanged; it MUST NOT re-serialise between signing and navigation.
Each parameter MUST appear exactly once; a missing or repeated
`request` or `proof` parameter is `invalid_request`.

#### 9.4 Authoriser processing

In order. Steps 1 and 2 apply only to the browser binding. A non-browser
authoriser instead receives the exact `request` and `proof` strings over
the non-redirect channel of section 9.2; it MUST reject `redirect_uri`
and `state`, and returns its response over that same channel.

1. Verify that the request arrived by top-level navigation in a
   top-level browsing context: the page sets
   `Content-Security-Policy: frame-ancestors 'none'` and
   `Cross-Origin-Opener-Policy: same-origin`, and checks that it is the
   top window.
2. Read and strip the fragment with `history.replaceState`; take the
   `request` and `proof` parameter values as received.
3. Hash, then decode. Compute `request_digest` over the ASCII bytes of
   the `request` value before any decoding (section 9.2). Then
   base64url-decode and JSON-parse both values and validate them
   against the member sets of section 9.1; a value that does not decode
   or parse, or an object with an unknown, duplicated, or wrongly typed
   member, is `invalid_request`.
4. Verify `v`, `chain`, `aud`, `iat`, `exp`, `nonce` freshness, and
   the `Proof` object against `request_digest`. In the browser binding,
   reject with an in-place error page (no redirect) if `redirect_uri`
   fails section 9.1's scheme, same-origin or fragment rules.
5. Verify the scheme is specified in section 3 and the account is capable
   (`spec_version >= 3` for this schema, and the arm deployed per the authoriser's own
   record, since verifier keys are not a specified chain read); check
   `bounds`, the implications, and the envelope-1 read-only rule.
6. Sign the user in with an authoriser credential whose relying-party
   (RP) identifier equals the full authoriser host, never a parent
   domain; this sign-in assertion yields no signing material.
7. Render the consent screen of section 9.5.
8. On approval, and only then, perform the authorising ceremony: the
   WebAuthn assertion whose PRF output derives the device key is
   requested with `challenge = challenge_issue_grant_with_<device arm>(...)`,
   so that `clientDataJSON` itself binds the approved scope and the
   user-verification flag is the consent evidence; derive `grant_id`,
   draw `scope_salt`, and compute the commitments; if any element
   implies `read`, sign and submit `rotate_enc_key` and the re-encrypted
   inbox entries first, then `issue_grant`, both with the derived device
   key and both covered by the consent screen's read sentence; use that
   key for exactly the signatures of this ceremony and discard it;
   prove; submit; wait for inclusion (SHOULD; `pending` is the
   fallback); seal the viewing secret if `read`; deliver the response
   through the applicable binding (section 9.6).

The order matters because the derived device key is signing material
for any device-gated challenge (S18).

In the browser binding the authoriser MUST NOT display or log `state`, MUST set
`Referrer-Policy: no-referrer`, MUST NOT load third-party resources on
the consent page, and MUST redirect with status 303, never 307.

**Fees.** `issue_grant` is an ordinary account operation paid by the
account; an authoriser MUST surface the fee on the consent screen and
MUST return `temporarily_unavailable` rather than `granted` when the
account cannot fund the call. Sponsorship is out of scope; a conforming
flow MUST work account-funded.

#### 9.5 Consent screen

Rendered after sign-in and before the authorising ceremony, from the
exact plaintext scope that will enter the device challenge and from the
proven `client_id`. No dApp-supplied name, icon, statement, or color
name is displayed. The screen MUST display:

1. for any element that requests or implies `read`: that the dApp will
   be able to see every current and future shielded holding of the
   account, across all colors, until the encryption key is rotated;
   that this is not limited by the spend bounds; that
   rotate-before-share will be performed; and, when other live read
   grants exist, that they will be re-sealed or will need reconnection;
2. the authoriser's own origin;
3. the dApp identity: the proven origin host (punycode when scripts are
   mixed), or the operator-asserted identifier labelled as such, or the
   owner's own `self:` label;
4. the account (name and shortened address) and the network; when the
   request bound no `account`, the words that the dApp did not name an
   account and the approver is choosing one;
5. for each requested element: every requested operation in plain words
   ("withdraw unshielded", "withdraw shielded", "withdraw shielded to a
   contract");
6. the color as its full hex, and hex only: no color name is displayed,
   because no on-chain color-name registry exists that the authoriser
   can verify; a later revision MAY name one;
7. `per_call_cap`, `cap`, and `max_coin_value` in atomic units with an
   explicit smallest-unit label (no on-chain decimals metadata exists);
   on a re-issue, the tombstone's `spent` as the roster records it;
8. the recipient pin, or "to any recipient"; separately, the full caller
   contract address, or "via any caller", with the immediate-caller and
   public call-relationship meaning of section 6.2.1;
9. `expires_at` as a local date, or the words "never expires";
10. the implications (a shielded spend implies read);
11. the grantee key's scheme, envelope in words ("software key", "passkey
    created on bank.example", "your wallet's key"), and a short
    fingerprint the dApp is instructed to show on its side;
12. that the connection, its scope bounds, its expiry, and its call
    count are publicly visible on chain; the grant record hides its
    colour, recipient pin, spent amount, and dApp host, but unshielded
    transfers and exercised contract relationships retain the disclosures
    stated in Security Considerations;
13. that the transaction is paid by the account;
14. that the grant can be revoked from any device.

The response's approved bounds MUST retain the caller restriction, if
requested. The dApp MUST recompute `caller_commit` from the returned
scope and salt and compare it to the record before accepting the grant;
an omitted, removed, or substituted requested pin is a binding failure.

The approver MAY narrow any bound (lower cap, earlier expiry, narrower
recipient, fewer operations, smaller `max_coin_value`) and MUST NOT
widen one. The authorising ceremony MUST be performed with user
verification.

#### 9.6 Response and errors

303 to the exact `redirect_uri` with form-encoded parameters in the
fragment (never the query); the dApp strips the fragment with
`history.replaceState` on load. Per-record parameters are repeated once
per element of `grants`, in the request's `grants` order, so the
repetitions are index-aligned with the request.

| Parameter | Present | Value |
|---|---|---|
| `state` | always | echoed verbatim |
| `iss` | always | the authoriser origin, `https` (RFC 9207; compared by simple string equality) |
| `result` | always | `granted`, `pending`, `denied`, `error` |
| `chain` | granted, pending | canonical CAIP-2 |
| `account` | granted, pending | contract address hex |
| `grant_id` | granted, pending; per record | hex |
| `scope_salt` | granted, pending; per record | hex |
| `scope` | granted, pending; per record | the approved plaintext scope as a JSON object, `base64url(UTF-8 JSON)`, in any serialisation: the four flags, `color`, `recipient_kind`, `recipient`, `max_coin_value`, `per_call_cap`, `cap`, `expires_at`, and optional `caller` in the request's address encoding (omitted means unrestricted); it is not signed, and the dApp verifies it against chain state (section 9.7) |
| `tx` | pending, optionally granted | transaction identifier |
| `view` | granted with `read` | GrantViewSeal v1, base64url (section 8) |
| `error`, `error_description` | error, denied | from the tables below |

The approved scope is returned because the grantee needs
`recipient_kind`, `recipient`, and `max_coin_value` as commitment
openings and the approver may have narrowed them; a dApp MUST use the
returned values and MUST verify that each equals or attenuates its
request. No key list, no account list, no bearer artefact, nothing in a
query component; if `redirect_uri` failed validation the authoriser MUST
NOT redirect at all. In a non-browser binding the authoriser returns the
same parameter set as a structured object over the channel that carried
the request, omitting `state`, `iss`, and `redirect_uri`; the grantee
performs the recomputation and chain read of section 9.7. Nothing on
the return leg is signed, so no serialisation of it is normative;
authority rests on the chain record.

Errors rendered in place (never delivered to `redirect_uri`):

| `error` | When |
|---|---|
| `invalid_request` | a `request` or `proof` parameter missing, repeated, not base64url, or not JSON; an unknown, duplicated, or wrongly typed member; a missing member; `state` too short, `aud` mismatch, outside `[iat, exp]`, replayed `nonce`, `redirect_uri` invalid or not same-origin with `client_id`, `client_id` matching no production of section 4.4 or non-browser by navigation, invalid `read_pk`, re-seal to a `read_pk` other than the bound one |

Errors delivered to a validated `redirect_uri`:

| `error` | When | Connector analogue |
|---|---|---|
| `unsupported_version` | `v` unknown | |
| `unsupported_chain` | `chain` not served by this authoriser | |
| `unsupported_scheme` | scheme not specified in section 3, or arm not deployed on the account; a withdraw string requested by an envelope-1 grantee | |
| `invalid_scope` | unknown string, the reserved `signin` string, bounds inconsistent or present on a read-only element, implication violated, no on-chain effect requested, duplicate `slot` | |
| `origin_mismatch` | attested `clientDataJSON.origin` differs from `client_id` | |
| `invalid_proof` | signature, challenge, type, or `rpIdHash` check failed; key not on curve, the identity, or weak; extension-bearing assertion; `signature` type in the browser binding | |
| `account_not_capable` | account `spec_version < 3` | |
| `read_unavailable` | this authoriser does not hold the account viewing secret | |
| `access_denied` | the user declined this request | `Rejected` |
| `permission_rejected` | the user declined and asked not to be asked again for this `client_id` | `PermissionRejected` |
| `server_error`, `temporarily_unavailable` | as RFC 6749 section 4.1.2.1; the account cannot fund the issuance | `InternalError` |

A dApp that received `result=pending` MUST poll chain state for its
records and, once they are live, obtain its `view` through a silent
reconnect (section 9.8).

#### 9.7 Exact-match and return-leg rules

- `redirect_uri` is compared by exact string match; the only exception
  is the port of a `localhost` loopback URI for native clients
  (RFC 9700 section 4.1.3).
- The dApp MUST discard any response whose `state` matches no pending
  request in the same browser session or whose `iss` differs from the
  authoriser it navigated to.
- The dApp MUST recompute each `grant_id` from its pending key,
  `[envelope,]` normalised `client_id`, and `slot` by the byte recipe of
  section 4.3 and compare with the returned values; MUST read
  `grants[grant_id]`, `device_epoch`, and `grant_generation` from chain
  state (an indexer query or a replay of the account's contract
  actions); MUST verify `active`, `epoch`, `gen`, that `object_commit`
  opens under the returned `scope_salt` and the returned plaintext object
  fields, that `rp_commit` opens to its own `rp_id_hash` (r1) or to zero,
  that `caller_commit` opens under the same salt to the returned caller
  (or is zero for an unrestricted grant), and that every returned field
  equals or attenuates its request. A requested caller restriction MUST
  remain the same restriction in the returned record; and the dApp
  only then moves the pending key into use. Redirect parameters are
  hints.
- A `grant_id` or opening that does not match MUST be treated as
  evidence of a compromised authoriser: the dApp MUST NOT use the key
  and MUST prompt the user to revoke from another device.
- The pending private key SHOULD be non-extractable (WebCrypto or the
  passkey-gated store), never plain `localStorage`. Losing `scope_salt`
  costs the ability to make any further call under the grant, which must
  then be revoked and re-issued; losing `spent` costs the ability to
  open `spent_commit` until reconstructed from the grantee's own
  transaction history.

#### 9.8 Silent reconnect

A dApp holding a live grant reads chain state and does not redirect. An
authoriser receiving a request for which the dApp's recomputed
`grant_id` is already live MAY, after sign-in and the full read consent
screen, return `result=granted` with a freshly sealed `view` and no
transaction, provided `H(read_pk) == scope.read_pk_hash`; a differing
`read_pk` is `invalid_request`. This is how a dApp recovers its viewing
material on a new device that holds the same per-origin passkey and
therefore, by section 8 item 1, the same `read_pk`.

### 10. Sign-in

This MIP defines account-linked, grant-backed sign-in only. A dApp
holding a grant knows the account address, because it needs that address
to call the contract and to read its record; sign-in is the dApp
verifying a message signed by the grantee key against the registered key
and chain state, and never touches the authoriser or the chain. There is
no identity-only on-chain grant. Every grant discloses the account
address by construction; unlinkable sign-in belongs to the sign-in MIP,
for which `midnight:account:signin` is reserved.

Message layout (EIP-4361 shape; fixed line order, `LF` separated,
UTF-8):

```
{domain} wants you to sign in with your Midnight account:
{account}

URI: {uri}
Version: 1
Chain ID: midnight:{reference}
Nonce: {nonce, at least 128 bits of entropy, alphanumeric}
Issued At: {RFC 3339}
Expiration Time: {RFC 3339}
Grant ID: {grant_id hex}
Incarnation: {issued_at, decimal}
```

`signin_digest = H(pad(64, "midnight:account:grant:signin:v1") || H(message))`,
signed in the arm's off-chain form of section 3.4 (`r1` as a WebAuthn
assertion with `challenge = signin_digest` on the dApp origin).

Verification by the dApp: the signature is valid under the registered
`(scheme, [envelope,] pk)`; `domain` equals the dApp's own origin host
and `URI` is within the dApp's own origin; `Version` is `1`; `Chain ID`
equals the dApp's expected CAIP-2 identifier; `Issued At` is within the
dApp's tolerance and before `Expiration Time`; `Nonce` is unused; and,
from chain, `grants[grant_id]` exists, `active`, `epoch == device_epoch`,
`gen == grant_generation`, not expired, and `issued_at` equals
`Incarnation`, so a message signed against one incarnation never
verifies against a re-issue (GR-6). Cross-dApp collusion links users by account address; the
mitigation is multiple accounts, not this MIP.

### 11. Ecosystem carriage

#### 11.1 CAIP-25 carriage

The grant feature strings and their bounds travel in the CAIP-25
`scopedProperties` member, keyed by the MIP-0008 CAIP-2 identifier, with
the connector's bare network id accepted on input and canonicalised:

```json
{
  "scopedProperties": {
    "midnight:mainnet": {
      "midnight:account:grants": [ { "slot": 0, "scope": ["..."], "bounds": { } } ]
    }
  }
}
```

They never appear in `methods`, and a CAIP-25 peer MUST NOT treat a
grant feature string as invokable. This MIP defines no `accounts`
member; CAIP-10 for Midnight is not yet defined.

#### 11.2 Open Wallet Standard mapping

A `GrantRequest` is expressible in Open Wallet Standard PE-1 policy
terms field by field, and the grant record is the on-chain object PE-5
asks to pair with:

| PE-1 term | Grant field |
|---|---|
| target contract | the account address (`account`, `kernel.self()`) |
| circuits | the three operation flags |
| argument predicates | `color` equality; the recipient pin |
| value per color, per-call bound | `per_call_cap` |
| value per color, cumulative bound | `cap` |
| value per color, per-window bound | `window_len`, `window_cap` (reserved; client-side until enforced) |
| validity | `expires_at`, `active`, `epoch`, `gen` |

An Open Wallet Standard vaulted key is a first-class grantee through the
`rdns:` or `mais:` identity in the non-browser binding: the vault signs
the grant challenge, its policy engine mirrors the scope, and the
contract enforces it regardless. This MIP asks that standard for a
handshake capability flag ("I can present a `GrantRequest` and sign
grant challenges") and acceptance of the mapping table of section 5.2.

#### 11.3 The authoriser role

The authoriser is a role: it holds a device key of the account, renders
a `GrantRequest`, drives `issue_grant`, and, to serve `read`, holds the
viewing secret. A consent page hosted by an identity provider is one
instance; it holds no assets and no spend path. A conforming account
MUST be connectable through a locally run authoriser, self-hosting MUST
be documented, and a dApp MUST treat an unavailable authoriser as a
recoverable error carrying no state. A device key derived under one
authoriser's RP identifier cannot be derived by another host, so a
second authoriser needs its own `add_device` and interacts with AUTH-5;
one device key MUST NOT be enrolled for more than one authoriser host.

### 12. Relationship to MIP-0013 and combined conformance

This MIP defines an optional grant-enabled account profile over MIP-0012
and MIP-0013. MIP-0013 section 4 anticipates scoped grants, but its
AUTH-1 and AUTH-2 literally require a device signature and advancement
of `auth_nonce` for every gated call. Accounts implementing this MIP
MUST claim conformance to the combined profile defined here, not to
those two requirements unchanged. These adaptations apply only to
accounts implementing this MIP; device-only MIP-0013 accounts are
unaffected. A separate amendment to MIP-0013 is not a prerequisite for
this proposal; this section is the proposed normative conformance delta.

1. **Device authority is preserved.** Device-authorised calls MUST
   retain MIP-0013's credential validation, current-epoch membership,
   challenge binding, device-entry rotation and `auth_nonce`
   advancement. ECDSA device arms additionally follow MIP-0030. Grant
   issuance, revocation and revoke-all remain device-authorised under
   section 7. A grant is never a device entry and MUST NOT contribute
   to `device_count` or the last-device check.
2. **AUTH-1 for grant calls.** Only the three grant spend twins may use
   a grant as their authority. Each MUST verify the grantee signature
   against the key used to recompute `grant_id`, require a live record
   issued by a device, and enforce the record's operation, object,
   caller and quantitative bounds before releasing assets (GR-1,
   GR-3, GR-4, GR-7 and GR-19). A grantee signature alone does not
   establish authority. Device and grant records and signature domains
   remain separate; the authoriser's refusal of device-key reuse is
   an off-chain obligation, as section 3.3 states.
3. **AUTH-2 for grant calls.** A grant challenge MUST cover that
   record's contract-held `nonce` and `issued_at`. Every successful
   grant call MUST advance its own nonce, and MUST NOT read or advance
   `auth_nonce` or modify a device entry. A signature authorises at
   most one executed call and MUST NOT become usable after revocation
   and re-issuance (GR-5, GR-6 and GR-12). Other state changes, such as
   spending the same coin, may still invalidate a pending device call.
4. **Recovery and revocation.** Every grant call MUST enforce the
   current account epoch and grant generation. Recovery finalisation
   MUST render all pre-finalisation grants unusable, including grants
   issued while recovery was pending. No later epoch change may
   reactivate them without fresh device-authorised issuance. Clearing
   or rotating device entries alone is insufficient (GR-8 and GR-9).
5. **AUTH-9 remains a device rule.** Its device-handle privacy
   requirements are unchanged. Grant uses may disclose a stable
   `grant_id`, so uses of the same grant are linkable. The account-
   bound identity, per-connection keys and salted scope commitments
   provide only the hiding properties stated in sections 4 and 6.6
   and Security Considerations; publicly known candidate keys can
   weaken them. This permission does not permit a stable device
   handle or disclosure of a device key through a grant record.
6. **Other guarantees and tests.** AUTH-3 through AUTH-8 and AUTH-10
   continue to apply to device operations. Grant operations MUST
   satisfy the corresponding call-binding, non-exfiltration,
   revocation and state-isolation requirements in this MIP. The
   MIP-0013 device conformance suite remains required on an extended
   account; passing grant tests does not replace it. The grant tests
   additionally cover replay, scope rejection, recovery and the
   independence of device and grant freshness. Completion of those
   tests is an acceptance criterion, not an assertion that the current
   reference implementation has already passed every item.

### 13. Invariants

A conforming implementation MUST satisfy all of the following, in
addition to the custody MIP's INV-1 through INV-8, the authorisation
MIP's device requirements and the explicit combined profile of section
12. ECDSA arms MUST satisfy MIP-0030 sections 2 and 3, as applied to the
grant challenges and credential bindings specified here.

| Id | Normative statement | Upstream |
|---|---|---|
| GR-1 (single seam, closed operation set) | Every grant-authorised operation is verified inside the account contract by the grant seam, which exists only as the grant twins of the three withdraw circuits. | INV-1 |
| GR-2 (authority separation) | Grant authority is looked up only in `grants` under grant signature domains, separately from device authority. The authoriser MUST refuse to issue a grant to a key it knows is enrolled as a device of the same account; the contract does not enforce key-set disjointness (3.3). | section 12; MIP-0030 section 1 |
| GR-3 (contract-maintained identity) | `grant_id` is a deterministic commitment to `(account, arm, key, [envelope,] origin_hash, slot)` recomputed in-circuit at every use; one tuple has at most one live record; a key's records at one origin number at most 256 and are enumerable; only device-gated circuits create, retire, or re-scope a record; a grant twin writes only its own record's `nonce` and `spent_commit`. | |
| GR-4 (full binding) | Every grant challenge covers the account address, the per-operation tag, the grantee key, `grant_id`, `issued_at`, every operation argument, every consumed witness value, and the record's `nonce`; every private argument is pinned by equality against the record or the ledger. | AUTH-3, AUTH-10, MIP-0030 section 2, custody S5 |
| GR-5 (per-grant single use, owner liveness) | Every grant call binds the record's `nonce` as read from state and advances it; identical resubmission fails; grant calls never read or advance `auth_nonce`. | AUTH-2 under section 12 |
| GR-6 (incarnation isolation) | Every grant-call challenge and the sign-in message bind `issued_at`, so a signature against one issuance of an id verifies against no other. | |
| GR-7 (in-circuit scope over consumed values) | Operation flag, the color and value of the coin actually consumed, the caps, the recipient pin, `expires_at`, `active`, `epoch`, `gen`, and `enc_key` equality are asserted before any custody chip executes; an out-of-scope call fails at proving or verification, never at application discretion. | |
| GR-8 (contract-derived lifecycle fields) | `epoch`, `gen`, `issued_at`, and the commitments are written by the contract from ledger state and the plaintext the approver signed; no grant can be pre-planted for a future epoch or generation. | |
| GR-9 (kill totality) | After a `device_epoch` or `grant_generation` bump no grant recorded under a previous value authorises anything. | AUTH-6 |
| GR-10 (chain-verifiable status) | `active`, `epoch`, and `gen` are public state; revocation is effective from the including block; the status of a known record is verifiable from chain alone; the grantee's identity is held in the owner's roster (7.5). | |
| GR-11 (no widening) | A record's scope is immutable after issue; modification is revoke then issue under fresh device authorisation; the approver may narrow and never widen. | |
| GR-12 (owner-only lifecycle, non-interference) | The lifecycle circuits are device-gated; a grant call changes no device entry, `device_count`, `auth_nonce`, `enc_key`, or any other grant's record; grants never count toward the last-device rule. | AUTH-5 |
| GR-13 (round monotonicity) | Issue, revoke, revoke-all, and every grant call strictly increase `round`. | INV-7 |
| GR-14 (key validity) | The authoriser MUST perform the issuance key checks of section 3.3, and the grant seam MUST perform the specified in-circuit checks at every use. `issue_grant` receives an opaque grant id and does not itself validate the grantee key. Invalid keys cannot authorise a conforming spend. | MIP-0030 section 2; section 3.3 |
| GR-15 (bounded disclosure) | A grant call discloses `grant_id`, the record update, the kernel comparison argument, and the custody chip's disclosures, and never the grantee key, origin, dApp host, slot, salt, openings, or signature; no grant field records any part of a held coin's description in the clear. | INV-2; AUTH-9 device boundary in section 12 |
| GR-16 (consent equals record) | Every plaintext scope field is an argument of `issue_grant` and enters the device challenge through `scope_digest`; consent renders from those fields and a possession-and-origin-proven request and precedes the ceremony that produces signing material; the dApp verifies the recorded grantee on return. | |
| GR-17 (redirect binding) | The authoriser acts only on a request whose possession proof, computed over the request bytes as received, shows a credential on the requesting origin, whose `aud` names it, whose window is current and `nonce` unseen, whose `redirect_uri` is same-origin with the attested `client_id` and matched exactly, and whose `state` it echoes; the binding is chosen by the transport; the response carries no bearer artefact or key list; the dApp treats its grant as live only after recomputing `grant_id` and reading the record from chain. | RFC 9700 sections 2.1, 4.1.3, 4.10; RFC 9207 |
| GR-18 (read is a capability, and it is total) | `read` is declarative and the ledger does not enforce a read scope; a grant with `read` confers the whole viewing capability until rotation; the secret is delivered sealed to a bound `read_pk`; rotate-before-share precedes every read issuance and every rotation re-seals or flags the other live read grants; a read revocation rotates `enc_key`; every inbox reader verifies each entry against chain data. | custody MIP R9, S2, section 6.5 |

**GR-19 (caller restriction).** A nonzero `caller_commit` admits only
the matching immediate contract from `kernel.caller()`, in addition to
all existing grant predicates. A zero commitment causes no caller read.
The claim-policy and disclosure boundaries are stated in section 6.2.1.
GR-15 includes the authenticated caller-context read on restricted calls;
it does not conceal the exercised contract relationship.

### 14. Versioning

The caller extension advances the schema to `spec_version = 3` and the
scope digest to `midnight:account:grant:scope:v2`, including unrestricted
new grants. Grant identity and operation challenge domains stay at v1;
the record's `issued_at` and nonce continue to bind each signed spend.
Old scope:v1 issue signatures cannot issue v3 records. Historical v2
measurements below describe the base grants implementation.

- **Document.** Versioned by its MIP number and revision history;
  substantive changes after acceptance require a new MIP listing this
  one in `Replaces`. It extends the two parent MIPs and supersedes
  neither.
- **Contract schema.** Caller-capable accounts expose `spec_version = 3`.
  New cells and structs are a redeploy (Backwards Compatibility
  Assessment); twins, lifecycle circuits, pure derivations, and tag
  families are maintenance updates while the authority is live, under
  the wave rule of section 6.7. That is measured: the twelve grant
  circuits reached a live account entirely through maintenance updates,
  in two batches at the authority counter read from chain, with the
  authority retired in the same update as the last batch. Enabling the
  window bounds later is a circuit revision (`:v2` twins), not a
  redeploy.
- **Scheme.** A grant scheme is a new tag prefix under the authorisation
  MIP section 10's "policy structure" clause, with `grant` as the
  policy-structure segment; a revision of any construction is a new
  trailing version segment. This Draft specifies `v1`, `k1` and `r1`
  grant arms; it does not assign independent standards-lifecycle status
  to them. MIP-0030 is Draft and specifies no adopted k1 sunset.
  Envelope-1 grantees remain `read`-only. r1 grant conformance remains
  incomplete despite available native secp256r1; `schnorr_bip340` is
  reserved by this proposal. The evidence record
  records the block-time unit as measured (E3): whole seconds since the
  UNIX epoch on ledger 9 (node 2.1.0), enforced at client build and at
  node admission, with the admission-time block time about one block
  interval plus a network-defined tolerance ahead of the wall clock.
- **Tag families.** The byte recipes below preserve the reference
  implementation's domains. MIP-0021 specifies a `[vN]` suffix and
  explicitly rejects a central registry; this proposal's existing
  `:vN` forms are therefore not a claim of conformity to that naming
  convention. Before adoption, review MUST settle whether these domains
  are retained as explicitly documented compatibility forms or revised
  with new vectors and migration rules. Renaming a tag silently would
  change grant identities or signatures. No registry submission is
  assumed.

  | Family | Convention | Members |
  |---|---|---|
  | `midnight:account:grant:id:{v1,k1:v1,r1:v1}` | raw 32-byte pad, tuple element | identity preimage |
  | `midnight:account:grant:{obj,spent,rp,caller}:v1` | raw 32-byte pad, tuple element | scope and caller commitments |
  | `midnight:account:grant:scope:v2` | raw 32-byte pad, tuple element | current scope digest, including `caller_commit`; scope:v1 is historical only |
  | `midnight:account:grant:auth:{v1,k1:v1,r1:v1}:<operation>` | hashed from a 64-byte pad | per-twin DST; the 64-byte width is a normative budget on operation names |
  | `midnight:account:auth:{v1,k1:v1,r1:v1}:{issue_grant,revoke_grant,revoke_all_grants}` | the existing device family (the `k1` auth family stayed at `:k1:v1` when the device and boot families moved to `:k1:v2`; verified against the reference before publication) | lifecycle DSTs |
  | `midnight:account:grant:{origin:v1,request:v1,signin:v1}` | raw pad prefix at the width that fits (32, 64, 64), off-chain | `origin_hash`, `request_digest`, `signin_digest` |
  | `midnight:account:grant:{view,read}:v1` | ASCII HKDF `info` and PRF input | GrantViewSeal; `read_pk` derivation |

- **Containers.** GrantViewSeal carries its own leading version and
  suite bytes; readers MUST skip unknown values rather than treat them as
  errors, as the custody MIP requires of InboxEntry.
- **Request.** `v` in the `GrantRequest`; unknown versions fail with
  `unsupported_version`.

## Rationale

**R1. A second authoriser class behind the same seam.** The custody
MIP's INV-1 admits exactly one gate, so a grant must be a way to satisfy
`require_authorised()`. Widening "device" would make a grant count
toward `device_count` and weaken AUTH-5; a separate verifier contract
would add a cross-contract dependency. Compact 0.35.0 now supplies
immediate caller identity, used directly in the existing account seam;
a sender witness is the `ownPublicKey()` pattern the custody MIP section 4
forbids. The remaining shape, a distinct record class verified by grant
twins calling a grant seam chip where device twins call the device chip,
is what the account-custody prototype evidenced on a devnet node.
Merging device and grant twins was rejected: a circuit conditional
evaluates both sides and would double every call's cost.

**R2. An explicit map with a contract-recomputed identity.** The device
lifecycle limitation described in MIP-0013 section 6 is duplicate
enrolment: retiring one rolling entry does not necessarily retire every
entry associated with the same key. Grant identity is recomputed at use
so that one `(key, origin, slot)` tuple cannot select another live record. The alternatives:

| Shape | Enumerable | Targeted revocation | Duplicate-credential records | AUTH-9, INV-2 |
|---|---|---|---|---|
| Rolling single-use entries in a `Set` | no | races the grantee | avoided | complete |
| Grant entries in `devices` | no | as devices | inherited; drifts `device_count` | as devices |
| Grantee key stored in the record | yes | yes | avoided | stable grantee pseudonym |
| Contract-maintained grantee set | yes | yes | avoided | set member dictionary-testable |
| `Map` keyed by a contract-recomputed `grant_id` (this MIP) | by id | instant | one live record per `(key, origin, slot)` | per-grant pseudonym hiding in key entropy |

The rolling set is the privacy-preserving successor for call count and
status. Contract-written `epoch` and `gen` prevent a client from
pre-planting a grant for a future epoch or generation. In-circuit
derivation of `grant_id` inside `issue_grant` was rejected: it
multiplies the issue circuits by the grantee arms, and the authoriser
holds the derived device key during the ceremony anyway; the dApp's
recomputation of `grant_id` is the adopted mitigation.

**R6. Salted commitments, not clear fields with an INV-2 carve-out.**
The custody MIP section 6.1 forbids writing a held coin's color or
value, in whole or in part, into public state, and a shielded spend's
amount is not public today. A clear `spent` delta would publish each
output coin's value, a clear `color` a color the account holds, a clear
`ZswapCoinPublicKey` pin a counterparty, and an unsalted host hash every
browser connection, since host names are a small dictionary. A
successor cannot weaken an invariant, and the commitment design costs
the seam only the rolling-commitment shape the device seam already pays.
An "any color" sentinel makes the cap dimensionless; a recipient set is
not expressible in a struct field.

**R9. Origin by browser attestation.** Browsers send no `Origin` header
on a top-level GET and `Referer` is suppressed by `no-referrer`. A
plaintext origin on chain would publish every connection; the record
holds the origin only inside the `grant_id` preimage beside a
per-connection key and, for r1, inside a salted commitment. Fetched
`.well-known` metadata adds a network and cross-origin resource sharing
(CORS) dependency to consent and is the recorded extension. Passkeys are
relying-party scoped, so a passkey registered for the dApp is a second
credential on the dApp origin with a P-256 key, hence `r1` is the
first-class arm; a companion passkey attesting
the origin over a digest that covers a software key is the alternative
for clients not using the native r1 grant profile. A key-only proof with a
"self-asserted" label was rejected because a user shown it will click
approve. The authoriser performs the write because a dApp-performed
write would make the dApp balance fees at connect time, tie the artefact
to `auth_nonce`, and put a proven transaction in a URL fragment.

**R11. Read as the sealed viewing capability.** A WebAuthn credential
never performs a key agreement, so a separate X25519 `read_pk`,
PRF-derived for browser grantees, is the only construction every grantee
can operate. A per-coin mirror leaves later deposits invisible and hides
the grantee's own change; it is the direction of the custody MIP's R9
successor. Proxied reads put an operator on the read path; a sealed blob
on chain was rejected because silent reconnect delivers the same
portability at no schema cost. Rotate-before-share is a MUST because the
blast radius is otherwise the account's whole history.

**R13. Positioning.** The upstream private-mandate proposal standardises
a delegation record of the same shape, but authenticates its agent by a
sender witness and attaches to an admin-overwritten balance rather than
custodied assets; it can adopt a grant as its authority object. The dApp
connector and the Open Wallet Standard are wallet-side and ephemeral;
this MIP sits above them as the ceremony a user approves once and below
them as the object their signatures target, and reuses their vocabulary.
MIP-0015's `deriveSecret` is seed-anchored and a seedless account cannot
satisfy it.

**R21. Signed as received, not a JWS container.** The request object
has the shape of an OAuth request object (RFC 9101), and the obvious
container for a signed JSON object is a JWS (RFC 7515). It was rejected
for three reasons. First, the browser proof is a WebAuthn assertion,
whose signing input is `authenticatorData || SHA-256(clientDataJSON)`
with the challenge inside `clientDataJSON`; that is not the JWS signing
input, so no JOSE library could verify the proof this MIP relies on and
the container would be a wrapper around a signature it cannot check.
Second, JOSE registers no algorithm for Schnorr over JubJub, the `v1`
arm, and none for the connector's prefixed ECDSA envelope. Third, a JWT
is a bearer token verified by machines at scale, and its known failure
modes (`alg` confusion, `none`, key-id injection) are costs with no
matching benefit for a request that a human approves once on a consent
screen. What was borrowed from JWS is the rule that made it robust:
sign the transmitted bytes rather than a canonicalised object, so no
canonicalisation step exists to disagree about; carry the proof
detached from the payload, as an RFC 7515 detached content signature
does; and reuse the `aud`, `iat`, `exp`, and `nonce` claim names that
JWT-family request objects carry (RFC 9101). RFC 8785 canonical JSON
was the earlier choice; it added a specification every implementer had
to get byte-exact for no gain once the parameter value itself is the
signed unit.

**R22. Expiry as a transcript read, not a constraint.** The kernel
comparison compiles to a ledger read whose Boolean result is one public
input the proof asserts; it adds no constraints, and a never-expiring
record adds not even the read. The ledger applies no tolerance of its
own to the comparison: the client evaluates it against its wall clock,
and the node re-executes it at admission against an admission-time
block time that runs about one block interval plus a tolerance ahead of
the wall clock. That tolerance is the node's and is network-defined,
measured rather than specified here, which is why section 5.1 states
`expires_at` as a bound and leaves the margin to the client (E3).

| Decision | Chosen | Rejected | Why |
|---|---|---|---|
| R3 identity element | one-byte `slot`; request `nonce` bound only in `request_digest` | a 32-byte request nonce in `grant_id` | an unbounded nonce let one key hold records the owner could not enumerate and made the nonce a capability secret in URLs |
| R4 freshness | the record's `nonce`, read from state | the shared `auth_nonce`; a nonce argument | a grantee advancing `auth_nonce` could void the owner's pending signatures; a dead argument invites the omission of the check |
| R5 incarnation | contract-written `issued_at` in every challenge | nonce continuity; never reusing an id | closes a pre-revocation signature verifying against a re-issue; every issuance is a distinct credential |
| R7 change | `max_coin_value`; the change entry as a bound argument appended in the same circuit; `enc_pk == enc_key` | a standalone `append_inbox` twin | the twin wrote unbounded entries under a no-value scope; the equality makes a rotation abort the call rather than orphan change |
| R10 request transport | one base64url JSON object in the fragment on both legs, signed as transmitted, with a detached `proof` parameter | loose query parameters; a canonicalised object; a JWS container (R21) | a single parameter value is the signed unit and needs no canonicalisation; the fragment stays out of logs and `Referer`, not out of history (S19) |
| R12 sign-in | grant-backed, address-disclosing | an identity-only grant; a verifiable-random-function (VRF) identity | an identity-only grant costs a transaction for plain login; address hiding belongs to the sign-in MIP; the target credentials only sign |
| R14 wire names | connector spellings plus this MIP's two spellings | private short names | wallets already emit the connector strings |
| R15 network | no `network_id` cell; the wire `chain` member | a network cell; MIP-0008 in `Requires` | independent deployments already differ by address; a cell the ledger does not assert separates nothing; the record embeds no network identifier, so MIP-0008 is cited, not required; no cell defeats a state-preserving fork |
| R16 document shape | one MIP under MPS-0018 | a separate connection MIP | the grant is the reserved extension of two MIPs under MPS-0018; the split is offered if editors object |
| R17 lifecycle | tombstones kept; lazy expiry; a generation counter; renewal of an expired record is revoke then issue | deletion on revoke; automatic cleanup; an expiry test in `issue_grant` | enumerability needs a residue; a time-dependent `issue_grant` costs a kernel comparison for a transition composition already provides |
| R18 wallet keys | envelope-1 grantees `read`-only | spend through `signData` | a blind signing surface over a computable challenge is a signing oracle (3.2) |
| R19 device removal | no contract cascade; the client composes `revoke_all_grants` with a compromise removal | `remove_device` bumping `grant_generation` | a cascade severs every connection on routine retirement |
| R20 attribution | an owner-side roster (7.5) | key and origin in the clear | either publishes a stable pseudonym and every connection |

## Path to Active

### Acceptance Criteria

These criteria govern adoption and conformance maturity. An unchecked
item is not a prerequisite for submitting this document as a Draft
under MIP-0001. The proposal's requirements are distinct from the
partial implementation evidence below.

- [x] External co-author named: Hector Bulgarini.
- [ ] Prior-art offer in discussion #223 followed up and any adopted
      contribution attributed.
- [ ] MIP number assigned; MPS-0018's `MIP` header and Recommended MIPs
      list updated with this proposal.
- [ ] Editors accept the explicit combined conformance profile in
      section 12 and its relationship metadata. No separately merged
      MIP-0013 amendment is assumed.
- [ ] Domain-separator compatibility with MIP-0021 settled as described
      in section 14; any changed byte recipe has replacement vectors.
- [ ] Independent cryptographer review, findings addressed, of the grant
      challenge and `issued_at`, the transitive binding of openings, the
      hiding of `grant_id` and the salted commitments, the
      companion-passkey binding and the off-chain Schnorr form, the `r1`
      key validation, GrantViewSeal and the `read_pk` derivation,
      `signin_digest`, the `envelope_digest` separation, and the
      fork-replay statement.
- [x] E1 (historical v2 baseline): the grant arm on the reference
      contract at `spec_version = 2`, both stages complete: the two cells and structs, the pure
      derivations, the seam chips and three grant twins on each grantee
      arm, and the three lifecycle circuits on each device arm, with k,
      rows, and prover-key size measured over the whole thirty-circuit
      roster and proving times measured for every circuit the
      conformance run exercised; the layout, arity, and change-append
      questions settled with no fallback taken; Testing item 1 green.
      Residue, carried by other criteria and not by this one: one of
      the six grant twins has no on-node timing, the jubjub
      contract-recipient twin, because the run never called it; the two
      unshielded twins were timed in a later run on a non-native color,
      since what the reference localnet refuses is the funding leg, a
      small contract call carrying a NIGHT unshielded input that
      balancing pairs with a NIGHT change output
      ([unshielded evidence](../../../contract/GRANTS-E2.md#20260916-the-unshielded-grant-twins-on-node-grants-e4)).
      The later P-256 grant twins compile under Compact 0.35, and the
      23 October 2026 run exercises `wa-json134` unshielded grant spends
      on node (Implementation and E11). E1 does not certify the later
      caller-bound schema or the P-256 profile.
- [ ] E2: Testing item 2 green on a node, each case ending with the
      invariants it exercises. Held: twelve rejection rows and eight
      grantee-key rows against on-chain state, each ending with no
      transaction and the recorded six-field snapshot unchanged; a ninth
      key row refused by the client encoder before a transaction exists;
      the within-the-margin expiry row, built and submitted then refused
      at admission with that same snapshot unchanged; and the device-key
      control that measures GR-2. These historical snapshots cover
      `round`, `auth_nonce`, `inbox_count`, `device_count`,
      `grant_generation` and grant-map size, not complete state. Open, so
      the box does not close: the faults Testing
      item 2 lists as green off-node only (the remaining issue rules,
      recipient-kind mismatch, a wrong envelope, cumulative overflow and
      foreign witness color) and the remaining cases named there. The later
      unshielded run also holds the out-of-scope operation, wrong
      `spent_prev`, and tampered-signature rows against a funded chain
      state; these are build-time refusals, not node-admission refusals.
      The 23 October 2026 recovery node run adds stale-grant-epoch and
      prior-incarnation refusals, same-key slot isolation on both arms,
      and refusal to overwrite an expired but unrevoked record, with
      the tracked-state comparisons described in Implementation.
      Stale generation is structurally unreachable while revoke-all
      clears the register and needs an explicit
      conformance argument. Revoke plus issue over an expired but
      unrevoked id remains distinct from the already exercised live id.
- [x] E3: the on-chain unit of `kernel.blockTimeLessThan` and the
      never-expires arm pinned on a ledger-9 network and recorded in the
      evidence record, with past, future, zero, and wrong-unit cases
      (evidence record: ledger 9, node 2.1.0, whole seconds since the
      UNIX epoch; enforcement at client build and at node admission;
      admission-time block time about one block interval plus a
      network-defined tolerance ahead of the wall clock).
- [ ] E4: Testing item 4 with two origins and two platform passkeys, a
      read-only grant, a two-element batch, and a zero-DUST refusal.
- [ ] E5: the vectors of Testing item 5 published, the Rust side linking
      no compiled contract module.
- [ ] E6: the deploy budget and wave plan of Testing item 6 for the
      submitted schema and supported arms. Historical evidence holds
      the thirty-circuit v2 roster in three waves and the caller-bound
      thirty-six-circuit roster in seven waves. The profiled P-256
      deployment records a fifty-two-circuit inventory and ten waves
      ([profile guide](../../../contract/WEBAUTHN.md)). These are
      distinct source and toolchain snapshots, not interchangeable
      sizing claims. The pre-grants compatibility control remains open.
- [ ] E7: the read handover of Testing item 7, including the
      two-delegate re-seal case.
- [ ] E8, E9, E10: Testing items 8, 9, and 10 (item 9 green on node;
      item 10 includes the recorded rotation between grant signing and
      the attempted call, refused at build with no inbox append in the
      snapshot. Complete state and UTXO snapshots were not recorded;
      item 8 not run).
- [x] E11: an r1 grantee end to end under an explicitly supported
      WebAuthn profile (Testing item 12), satisfied for the bounded
      `wa-json134` unshielded path. The 23 October 2026 full-account suite
      records three grantee spends and sixteen build-time refusals;
      a separate three-operation fixture adds identity-key and
      authenticator-extension (ED/suffix) refusals against indexed state
      using the original constructor, circuits and verifier keys.
      Implementation distinguishes their execution layers and scope.
      General browser support, P-256 shielded grant spends and a successful
      forwarded P-256 call are not established by these runs.
      `schnorr_bip340` has its own independent arm and implementation status.
- [ ] A public reference implementation passing E2 as its conformance
      suite with the E5 vectors published.
- [ ] A second independent implementation of the grantee side (a wallet
      provider or an Open Wallet Standard plugin) producing bit-identical
      challenges from the byte recipes alone.
- [ ] Public-testnet deployment of a `spec_version = 3` account with at
      least one grant issued, exercised, and revoked by a third-party
      dApp; a dApp not written by the authors completing E4 from the
      text alone.
- [ ] The scope mapping table of section 5.2 accepted by the Open Wallet
      Standard upstream, or a documented divergence.

### Implementation Plan

1. Review this self-contained Draft and its executable reference in
   Passport first. A subsequent canonical submission follows MIP-0001,
   presenting section 12's AUTH-1/2 conformance delta, the MIP-0021
   domain-naming decision, and the evidence gaps explicitly. Link any
   assigned number from MPS-0018; a parent-MIP cross-reference may be
   added during that review.
2. Complete the grant rejection, recovery, caller and r1 matrices
   against the current schema. Re-run the device suite alongside them.
   Publish vectors for the current scope digest and every supported
   grantee profile; retain historical evidence with its original scope.
3. Build and exercise the reference authoriser and dApp ceremony,
   read handover, reconnect and re-seal paths (E4, E7 and E8). Validate
   the byte recipes with an independent implementation.
4. Commission independent cryptographer review and address its findings
   before requesting adoption. In particular, resolve validation-layer
   coverage, commitment hiding, replay and WebAuthn policy questions.
5. Settle domain-separator compatibility and the proposed wire spellings
   with the relevant maintainers. Confirm the custody viewing-capability
   interaction and the Open Wallet Standard mapping or document divergence.
6. Complete the remaining Path to Active criteria and then request the
   lifecycle transition required by MIP-0001. Draft publication itself
   does not assert completion of those criteria.

## Backwards Compatibility Assessment

This MIP introduces a new contract standard. It requires no ledger,
consensus, or node change and no hard fork; every mechanism used exists
in the current stable network protocol.

**Schema change: grant support is a redeploy.** `grants`,
`grant_generation`, and the two structs are new ledger cells and types.
A contract's ledger schema is fixed at deploy; only its circuits evolve
by maintenance update. A grant-capable account therefore exposes
`spec_version = 3`, and **an existing v1 or v2 account cannot gain this
schema by maintenance update**; migration is a new account and is
out of scope. Clients MUST read `spec_version` before requesting a grant
and MUST report `account_not_capable` otherwise. This schema change does
not repair duplicate device enrolment or device-wide retirement; those
remain separate from grant conformance.

**Custody MIP.** A grant is an "authorisation-policy object behind the
same seam" (its section 2). INV-1: grants are inside the seam. INV-2: no
coin description is stored in the clear and the running total is a
salted commitment. INV-3 and INV-4: the change rule and inbox backfill
fall on the grantee and are made atomic by the `change_entry` argument.
INV-5: a wrong qualified description fails at proving, unchanged for a
grantee. INV-6: one-hop remains the client default. INV-7: every grant
call bumps `round`. INV-8: the unshielded mirror is written only by the
unchanged custody chips. S5 is discharged in section 6.2. GrantViewSeal
is a new container with its own numbering; a companion note to R9 is
raised on the write channel the viewing secret opens.

**Authorisation MIP.** Section 12 preserves the device path and defines
an explicit AUTH-1/2 adaptation for the additional grant path. Extended
accounts MUST pass the device suite as well as the grant suite; this is
not a claim that every current reference path has already passed.
AUTH-9 remains a device-handle requirement. Wallet keys under MIP-0003
retain their existing meaning. MIP-0030 supplies the ECDSA and WebAuthn
rules; this MIP defines the grant-specific domains, credential binding
and wire spellings.

## Security Considerations

The dominant residual risk is section 8: `read` is total. The controls
are rotate-before-share with the re-seal obligation, sealed delivery to
a bound `read_pk`, the declarative flag for audit, rotation on revoke,
the disclosure sentence as the first consent item, and the custody MIP's
R9 successor as the named dependency for spend without full disclosure.

**Observability (normative).** A passive observer of current register
state learns the retained grant records, including retained tombstones,
and each record's admitted operations, read flag, caps, expiry, call
count and status. This is not the number of all connections ever made:
revoke-all may clear the map, and one connection can hold multiple slots.
An observer retaining transaction history can retain past records and
link calls by `grant_id` even after clearing. A caller restriction is
visible, and exercising it reveals the authenticated immediate caller.

The grant record does not publish the grantee key, origin, dApp host,
held coin color, recipient pin or spent amount in clear. This hiding
claim depends on secret per-connection key material and fresh salts;
known candidate keys and salt disclosure weaken it as described above.
Custody operations retain their own disclosures: unshielded transfers
publish color, amount and recipient; contract recipients and exercised
caller relationships are visible. GR-15 does not conceal those facts.
The caps are public policy values, not held coin values. AUTH-9's device
protection remains unchanged under section 12.

The authoriser knows `scope_salt` for every grant it issued and, if it
retains it, can open the corresponding commitments from public state
indefinitely; section 4.2 therefore says it SHOULD NOT retain the salt.

| | Attack | Mitigation |
|---|---|---|
| S1 | Grant injection (login CSRF) | possession proof over `request_digest`, hashed from the `request` bytes as received (9.2); browser-attested origin on every navigation; session-bound `state`. Residual (9.2): the proof shows a credential on the origin, not the dApp's front-end; consent and the roster are the remaining barriers |
| S2 | Open redirector | `redirect_uri` `https`, same-origin with the proven `client_id`, exact match; none in non-browser bindings (9.7) |
| S3 | Binding downgrade by `client_id` form | the binding is a property of the transport (9.2) |
| S4 | Mix-up between authorisers | `iss` string equality; the grant is read from chain, never from redirect parameters (9.7) |
| S5 | Request replay | `aud`, `iat`, `exp` under `request_digest`; one-time `nonce`; `account` MUST when known (9.1) |
| S6 | Key substitution by a compromised authoriser | the dApp recomputes `grant_id` and the openings and treats a mismatch as compromise (9.7); in-circuit derivation rejected (R2) |
| S7 | Grantee signature replay | `nonce` read in-circuit and advanced; `grant_id` and `issued_at` bound (6.3) |
| S8 | Blind-signing oracle through connector `signData` | envelope-1 grantees `read`-only, refused by the authoriser and asserted in-circuit by every `k1` grant twin (3.2, 6.2 step 1) |
| S9 | Scope creep or upward adjustment | only device-gated circuits write scope; attenuation only; modify is revoke plus issue (7.2) |
| S10 | Object-scope bypass through the witness coin | `coin.color` and `coin.value` asserted before the chip (6.2 step 5) |
| S11 | Change orphaning, including by a rotation racing a pending call | `max_coin_value`; the change entry appended in the same transaction; `enc_pk == enc_key` (6.2 step 5) |
| S12 | Revocation escape through duplicate credential records | contract-recomputed identity; bounded `slot`; tombstone read at use; roster reconciliation (7.5) |
| S13 | Device compromise not contained by `remove_device` | `revoke_all_grants` composed with a compromise removal (7.3) |
| S14 | Epoch or generation confusion, future-state pre-planting | `epoch`, `gen`, `issued_at` contract-written and asserted at use (GR-8) |
| S15 | Weak, off-curve or identity keys | the per-arm table of 3.3, including SEC 1 validation on r1; off-curve negative vectors |
| S16 | Origin spoofing through a free parameter | browser origin is browser-attested; optional contract caller enforcement uses the ledger's `kernel.caller()`, never a self-asserted address (6.2.1); the two identities are distinct |
| S17 | Consent deception by dApp-supplied strings or units | consent from the exact plaintext scope and the proven `client_id`; no dApp strings; atomic units; hex color (9.5) |
| S18 | Consent as a Document Object Model (DOM) event on a page holding signing material | consent precedes the ceremony; the assertion challenge is the issue challenge; the derived key is used once; `frame-ancestors 'none'`, top-window check, `Cross-Origin-Opener-Policy` (9.4) |
| S19 | Leakage through URLs, logs, referrers, history | fragment transport on both legs; `no-referrer`; no third-party resources; `replaceState`; `view` sealed; `scope_salt` confers no authority but is the key to the record's INV-2 protection, and history, session restore, profile sync, and extensions remain exposure surfaces |
| S20 | Low-order, non-canonical, or malicious `read_pk` | rejected; abort on an all-zero shared secret; the associated data binds `eph_pk`, `grant_id`, `account` (8) |
| S21 | Silent reconnect to an attacker-chosen key | re-seal only to `read_pk_hash`; full read consent; roster (9.8) |
| S22 | Owner liveness griefing by a racing grantee | grant calls never touch `auth_nonce` (GR-5) |
| S23 | Register or inbox growth | only devices create records; one 192-byte entry per paid spend; counters `Uint<32>` or wider |
| S24 | Cumulative-cap wrap | widened comparison before the narrowing cast (6.2 step 5) |
| S25 | Inbox forgery with the delegated secret | readers verify each entry's coin against chain data (8 item 4) |
| S26 | Cross-account linkage of a grantee key or `read_pk` | the key is never stored; `kernel.self()` in the identity; per-connection keys; `read_pk` per account; publicly known keys are linkable, stated |
| S27 | Dictionary test "is this account connected to origin X" | `origin_hash` only inside a preimage with a per-connection key; the r1 host under a salted commitment; succeeds only for a publicly known key |
| S28 | Sibling-origin assertion against the authoriser credential | RP identifiers equal their full hosts; Related Origin Requests excluded (9.4, 11.3) |
| S29 | Address disclosure by sign-in | no identity-only grant; every grant discloses the address, stated (10) |
| S30 | Cross-network replay after a state-preserving fork | no cell defeats it; `kernel.self()` plus never reusing deploy content; expiry and revocation on the surviving chain (R15) |
| S31 | Kernel argument disclosure per call | `expires_at` is public anyway |
| S32 | Maintenance authority above the seam | grant circuits in the wave plan; authority retired after the last wave (6.7) |
| S33 | Malleated ECDSA twins | both `s` forms accepted; inert because `nonce` advances (MIP-0030 section 2) |
| S34 | Pending private key stored before consent | non-extractable storage SHOULD; unavoidable in kind |
| S35 | Concurrent coin selection | a fee-wasting race, or the loser refused by the node as a double spend; never a mis-spend, and never a proving failure, since proving consults the commitment tree and not the nullifier set (INV-5) |
| S36 | Fee-payment linkability | a dApp paying DUST from an address linked to its identity links itself to the account; outside the contract |
| S37 | Authoriser unavailability | a recoverable error carrying no state; any device-key holder can act as authoriser (11.3) |
| S38 | Toolchain hazards | the vacuous-verifier control of the authorisation MIP's S10; pinned toolchain versions |

## Implementation

**Evidence scope.** The measurements below belong to the source and
stack snapshots each evidence record identifies. Compilation, circuit
simulation, node execution with a software authenticator, and a live
browser ceremony establish different claims. None alone establishes
complete conformance to every requirement of this MIP.

**Caller extension (2026-10-01).** The recorded run uses Compact 0.35.0,
runtime 0.20.0 and `spec_version = 3`. Its JubJub and k256 device arms
bind scope:v2 and all six grant-spend circuits enforce the caller pin.
The [caller implementation guide](../../../contract/GRANTS-CALLER.md)
records a JubJub unshielded spend via a contract, the `A -> B -> account`
case, direct/wrong-caller/replay refusals, and a proved forged-context
submission refused by the node. The k256 and shielded caller predicates
are covered offline. P-256 later shares the same scope digest and gate;
that run does not exercise it. The following E1/E2 rows retain their
historical scope:v1 measurements and are not measurements of the caller
extension or the P-256 arm.

**P-256 profile.** The reference implements
[`wa-json134`](../../../contract/WEBAUTHN.md), with a 134-byte canonical
client-data object, a 21-byte origin, 37-byte authenticator data, and
UP+UV required. Other JSON shapes, origin lengths, and authenticator
extensions are outside this implementation profile. The
[compiled inventory](../../../contract/P256-MEASUREMENTS.md) contains
all three P-256 grant-spend circuits. The
[offline suite](../../../contract/src/tests/p256-offline.ts) exercises
a P-256 owner and grantee, including RP binding, scope, signature,
replay, and revocation checks. The earlier
[recorded node suite](../../../contract/evidence/p256-webauthn/conformance-and-proving.json)
uses an OpenSSL software authenticator for account calls, including
P-256 device-authorised grant issuance and revocation; it does not
exercise a P-256 grantee spend.

The 23 October 2026 [grant suite](../../../contract/src/tests/p256-grants.ts)
and its [node evidence](../../../contract/evidence/grants-completion-20261023/p256-grants-conformance.json)
establish the `wa-json134` unshielded end-to-end milestone on the full
52-operation account with an OpenSSL software authenticator. Eleven recorded funding/lifecycle
transactions include three proved and included grantee spends of 5 units
each, reducing the funded non-native balance from 100 to 85. Sixteen
negative cases exercise origin/RP/profile/signature binding, scope,
replay, revocation, incarnation isolation, read-only exclusion and root
caller rejection. They fail during circuit build against indexed state,
with zero prover or submission calls and identical serialized
`ContractState` before and after; they are not submitted invalid proofs.
The profile case uses a valid ES256 signature over unsupported client
JSON. Source and artifact hashes are recorded and checked unchanged
throughout the run.

The separate [refusal fixture](../../../contract/src/tests/p256-grant-refusals.ts)
and its [recorded results](../../../contract/evidence/grants-completion-20261023/p256-grant-refusals.json)
complete the bounded Testing item 12 negative cases. This unfunded fixture
deploys only three original account operations, retaining the original
constructor and verifier keys; it is not a full conforming account.
Three included setup transactions deploy it, activate its owner and issue
a real P-256 grant. An identity grantee fails at the Compact point-coordinate
accessor during client execution, before grant lookup. A signed 37-byte
authenticator header with ED set fails the circuit's profile flag check;
a genuine signed 44-byte assertion carrying ED and a CBOR extension-map
suffix fails the runtime's `Bytes<37>` argument encoding. A valid ordinary
assertion passes authentication and reaches the expected unfunded custody
guard. All four calls leave serialized `ContractState` unchanged and make
zero prover or submission calls. The 18 source/artifact hashes remain
unchanged through the run, and all 13 entries shared with the full-account
run match. These are execution/encoding refusals against node-backed state,
not rejected proofs or a complete adversarial ECDSA proof audit.

The separate
[live browser result](../../../contract/evidence/p256-webauthn/browser-flow.md)
is an account encryption-key rotation, not the connection ceremony or
a grant spend. E4, general browser support, P-256 shielded grant spends
and successful forwarded P-256 calls remain outside these results.

**Recovery.** The reference implements `recover_finalise`, including
an epoch increment and complete retirement of old device entries. The
[historical node run](../../../contract/evidence/rec-1-9-recovery-conformance.json)
exercises recovery and successor control. The expanded
[node suite](../../../contract/src/tests/recovery-conformance.ts), its
[23 October 2026 results](../../../contract/evidence/grants-completion-20261023/rec-1-9-recovery-conformance.json)
and the [validation report](../../../contract/GRANTS-COMPLETION.md)
also establish the grant sequence. A pre-existing JubJub grant and a
k256 grant issued during the pending window fail after each of two
recovery finalisations. Each finalisation leaves exactly its authorised
successor device. That successor revokes and re-issues both grant ids;
old-incarnation signatures fail even with fresh openings and nonce zero,
while fresh signatures spend 5 units on each arm without changing device
authority. The run also establishes same-key two-slot revocation
isolation on both arms and refusal to overwrite an expired but unrevoked id.

The negative cases fail during client build against funded on-chain state,
not at node admission. Before/after comparisons cover `round`, `auth_nonce`,
epoch, device count and entries, grant records and generation, unshielded
balances, inbox count, pending-recovery status and `enc_key`; they are
tracked-state snapshots, not complete serialized `ContractState` or UTXO
comparisons. The expanded [simulator](../../../contract/src/tests/recovery-sim.ts)
also passes these cases. Its cumulative-overflow and foreign-witness-color
cases on both shielded twins of both arms remain offline-only evidence.

| Evidence held | What it establishes |
|---|---|
| the account-custody prototype's v0 grants on a devnet node (withdraw only, one color, cumulative cap, tombstone revoke, epoch-scoped, bearer grantee) | in-circuit cap, color, tombstone, and epoch enforcement at the seam |
| the wallet-key gate experiment on a ledger-9 network (k=15 and k=16; 31,046 and 32,900 rows; 0.5 to 0.8 s) | the k256 envelope arm compiled and measured; the connector prefix as envelope `1` |
| the earlier P-256 in-circuit experiment against a real platform assertion (k=16, 36,466 rows, 1.1 to 1.2 s) | historical custom-stack envelope evidence; not the later Compact 0.35 account or grant proving cost |
| the reference implementation's wave deploy and block-limit numbers | the deploy budget rule of section 6.7 |
| E3: the kernel block-time comparison on a ledger-9 node (node 2.1.0): 21 unit and enforcement cases and a 13-case admission-time sweep over a probe contract carrying the seam's comparison shape, each case recording the host clock, the node head, the client phase reached, and the node outcome | the unit is whole seconds since the UNIX epoch on the client and the node alike; the comparison is a transcript read (one public input, no constraints) enforced at client build against the wall clock and at node admission against the node's block context, never by the proof; the never-expires arm records no read; the admission-time block time ran 8.2 to 10.2 s ahead of the wall clock on a 6 s-block localnet; the client-side and node-side refusal texts of section 6.4 |
| the cross-contract-calls experiment | composition in one transaction |
| E1 stage one: the grant seam compiled on the reference contract at `spec_version = 2` (the two cells and structs, the pure derivations, the k256 seam chips, the unshielded and shielded k256 grant twins, and the three lifecycle circuits on the k256 device arm), measured at k=16 to 17 and 58,771 to 91,862 rows, the two grant twins re-measured three rows higher in stage two once the envelope assert landed, 2,745-byte k256 and 2,313-byte jubjub verifier keys, and executed off-node in a circuit simulator (lifecycle, the unshielded twin, its rejection matrix) | the circuit items settled: a struct as a `Map` value, `Boolean` as a one-byte hash element, tuple arities of thirteen and seventeen hashed as the raw concatenation, the kernel block-time comparison inside an assert, the `if`-guarded lifecycle bodies, and the `Map` reset primitive; the byte recipes of sections 4 and 6.3 reproduced two ways from the text alone (a TypeScript recipe and a Rust recipe linking no compiled module) against the compiled circuits, 27 vectors and two pinned signatures; the deploy budget of section 6.7 |
| E1 stage two: the roster completed on the same contract (the jubjub grantee seam and its three twins, the remaining k256 twin, the three lifecycle circuits on the jubjub device arm, the client and Rust signer surfaces for both grantee arms, and an off-node suite of 121 checks over both arms), with the whole thirty-circuit roster measured from 311 rows at k=9 to 97,568 at k=17 | the six grant twins and six lifecycle circuits of section 6.1 exist on both arms with no exported signature changed; the cost table of section 6.7 and the seam's per-arm row cost; the `v1` challenge and lifecycle recipes of section 6.3 and their measured preimage widths, agreed three ways over 34 vectors and three pinned signatures; the `v1` key guard of section 3.3 and the runtime's refusal of every other point outside the prime-order subgroup |
| E2: the conformance run on a ledger-9 localnet (node 2.1.0, indexer 4.4.0-rc.2, proof server 9.0.0-rc.6), twelve evidence groups from one account and one chain: the three-wave deploy and a cross-arm enrolment, issuance across both device arms, grant spends on both grantee arms, a twelve-row rejection matrix, ten grantee-key rows, the expiry rows with their transaction transcripts read back, composition of two calls on one contract, the one-coin race, the change prediction, and every proof of the run timed at the proof provider, with the per-update ceiling bisected to one verifier key by a separate probe over throwaway accounts | the seam behaves on node as specified on both grantee arms; the change-precomputation rule and the `mt_index` retry rate of section 6.5; the refusal points and wrapper texts of section 6.4; the key-validation findings of section 3.3, including GR-2 as an authoriser obligation the contract cannot enforce; the composition mechanics of section 7.4; the wave plan, the per-update ceiling, and the proving times of section 6.7; the expiry horizon of section 5.1; Testing items 1, 3, and 9 green and items 2, 6, 10, and 11 in part |

The [later unshielded run](../../../contract/GRANTS-E2.md#20260916-the-unshielded-grant-twins-on-node-grants-e4)
adds five successful non-native-color spends across the JubJub and k256
grantee arms, proving samples for both twins, and fourteen build-time
rejection rows against funded chain state. The historical NIGHT
funding refusal is narrower than a general prohibition on unshielded
calls; it does not invalidate these successful grant spends.

Still outstanding for this MIP: the complete connection ceremony and
its two-origin attack suite (E4); GrantViewSeal read handover and
re-sealing (E7); the specified agent/self connection forms (E8); the remaining
cross-implementation constructions of Testing item 5; the pre-grants
compatibility control; the remaining rejection cases; and independent
cryptographer review. The key-rotation leg of Testing item 10 has
historical build-refusal evidence with the snapshot limits stated there.
General account
viewing-key sharing experiments do not by themselves establish the
GrantViewSeal protocol required here. The historical E2 run also leaves
the JubJub contract-recipient twin without an on-node measurement, and
the non-native unshielded run establishes no NIGHT grant spend.

The source implements three grant twins and three device-authorised
lifecycle circuits per deployed arm at `spec_version = 3`. The
[scope:v2 vectors](../../../contract/src/tests/vectors/grants-v3.json)
extend the earlier scope:v1 vectors to the caller commitment; the Rust
signer reproduces the `k1` and `v1` constructions without a compiled
contract module. A static consent page and a dApp implementing the
return leg, sign-in and read handover remain reference-client work.
This MIP's combined-conformance rules are stated in section 12; they do
not depend on an unsubmitted companion amendment. This MIP contains no
implementation code, and its reference implementation and full
conformance suite remain acceptance criteria.

## Testing

**Caller restriction (GR-19).** Exercise correct and wrong immediate
contract callers, absent and user-address callers, the `A -> B -> account`
case, and a forged off-chain caller context with no matching ledger
claim. Verify owner-signature binding of the pin, retained grantee
signature/freshness checks, and caller-independent transcripts for the
unrestricted branch. Wrong-caller checks cover both arms and all six
spend gates; on-node composition and the forged-context control are
reported separately in the caller implementation guide. The recorded
caller matrix predates P-256 integration; the same checks MUST cover
any additional deployed grantee arm before claiming its conformance.

Conformance is demonstrated by a suite exercising, against a real node,
the following. Each item names the invariants it exercises.

1. **Grant happy path.** Issue from a device; a grant call within scope
   executes; `nonce` advances, `spent_commit` re-commits, `round`
   advances, `auth_nonce` is unchanged (GR-1, GR-5, GR-13; INV-7).
   Expiry rows: a record with `expires_at = 0` and one with
   `expires_at = head + 3600` each execute on node with state advancing,
   and the `0` record produces no time read in the transcript (GR-7).
   Status: green on node for the recorded JubJub/k256 baseline.
   Issuance from both device arms; a within-scope spend on both
   grantee arms, with `nonce`,
   `spent_commit`, and `round` advancing and `auth_nonce` unchanged;
   and both expiry rows on the grant twins themselves, the accepted
   transactions read back through the indexer to show that the `0`
   record's call records no block-time read and the forward-dated one
   exactly one. The unshielded twins are held on node too, over a
   non-native color, on both grantee arms: five spends debiting the
   mirror, advancing `nonce` and `spent_commit`, and honouring a
   recipient pin (`evidence/grants-e4-spend-*.json`). Over NIGHT they
   are exercised off-node only, in the circuit simulator, because the
   reference localnet refuses the funding leg, a small contract call
   carrying a NIGHT unshielded input that balancing pairs with a NIGHT
   change output; that is a property of the network the run used rather
   than a failure of the non-native-color result. The P-256 grantee
   path is tracked separately in item 12.
2. **Rejection matrix.** The same call aborts with no state change under
   each single fault: out-of-scope operation; over `per_call_cap`; over
   `cap`; a cumulative wrap attempt; a wrong `spent_prev` opening; a
   witness coin of another color; a
   witness coin above `max_coin_value`; wrong recipient under a pin;
   recipient-kind mismatch; a stale `enc_pk`; revoked; `expires_at` in
   the past (the grantee's build aborts with `failed assert: grant
   expired`, nothing is submitted); `expires_at` within one block
   interval plus tolerance of the wall clock (the build succeeds, the
   node refuses at admission with `Custom error: 104` on a transcript
   read mismatch, state unchanged); a client clock skewed behind the
   network by more than the margin (the same admission refusal,
   showing the check does not rest on the grantee's clock) and skewed
   ahead (the client refuses a call the chain would accept, a
   client-quality control rather than a protocol property); stale
   epoch; stale generation; identical resubmission after success; a
   prior-incarnation signature against a re-issue; a cross-scheme key; a
   wrong envelope (k1); an envelope-1 grantee against any withdraw twin
   refused in-circuit (k1); the identity, small-order, off-curve, and
   invalid-curve keys on every deployed arm; a grantee calling any
   device-gated or lifecycle circuit; re-issue of a live id rejected;
   re-issue of an expired but unrevoked id rejected, and revoke plus
   issue over it in one transaction accepted; issue over an absent id
   succeeds; revoke of an absent id aborts; a reused `scope_salt` flagged
   by the harness as non-conforming; revoke one of two slots held by one
   key, the other still authorises and only that one; `device_count`
   untouched; the vacuous-verifier control (GR-2, GR-3, GR-4, GR-5, GR-6,
   GR-7, GR-9, GR-12, GR-14; the authorisation MIP's S10). One row is a
   control rather than a fault: a device key issued as a grantee is NOT
   refused by the seam and spends under it, which is the measured
   content of GR-2 (section 3.3). The replay row admits any of the
   step-5 needles and a suite MUST NOT name one (section 6.2), and on
   the `v1` arm a mismatched authorisation admits either the range
   error of the challenge cast or `invalid grant signature`
   (section 6.4).

   Status: partial. Historical rejection rows with before/after
   snapshots left the recorded fields unchanged: six fields in E2
   (`round`, `auth_nonce`, `inbox_count`, `device_count`,
   `grant_generation`, grant-map size), with the relevant unshielded
   balance added in the later unshielded run. The cross-arm client-encoder
   refusal has no such snapshot. These are not complete-state comparisons.
   Every rejection except the admission-margin row ended before submission;
   that row was built, proved and submitted, then refused at admission.
   Green against on-chain state: an out-of-scope operation flag on the
   unshielded twin; over `per_call_cap`; over `cap`; a wrong `spent_prev`
   opening; a witness coin
   above `max_coin_value`; wrong recipient under a pin; a
   stale `enc_pk`; revoked; `expires_at` in the past; `expires_at`
   inside the admission margin, built by the client and refused by the
   node; identical resubmission after success; a cross-scheme key in
   both directions; an envelope-1 grantee against a withdraw twin; a
   grantee calling a device-gated circuit; the identity on both
   deployed arms; an off-curve pair and an invalid-curve twin on `k1`;
   an order-2 point and an off-curve pair on `v1` (secp256k1 has prime
   order, so the identity is its only small-order point);
   `device_count` untouched across every row; and the device-key
   control above. The
   [23 October 2026 recovery run](../../../contract/evidence/grants-completion-20261023/rec-1-9-recovery-conformance.json)
   additionally holds same-key two-slot revocation isolation on both arms,
   refusal to overwrite an expired but unrevoked id, stale grants after
   two recovery epochs, and old-incarnation signature refusal using fresh
   openings and nonce zero. These negative calls abort during client build
   against funded chain state with the tracked fields in Implementation
   unchanged; successor-issued grants spend successfully on both arms.
   Green off-node only, in the circuit simulator: recipient-kind mismatch,
   a wrong envelope, and the remaining issue rules (re-issue of a live id
   refused, issue over an absent id succeeding, revoke of an absent id
   aborting, revoke of a tombstone aborting). The expanded recovery simulator also
   passes, on both JubJub and k256, a legitimate spend of `2^128 - 2`
   followed by an attempted spend of 2 with the correct `spent_prev`,
   refused at the cumulative cap before narrowing; a foreign-color witness
   supplied under the admitted color's lookup key on both shielded twins
   (four cases), refused as `scope object mismatch`.
   Not run anywhere: the two clock-skew rows on a grant twin (E3 carries
   them on a probe contract), the reused-`scope_salt` harness flag, and
   the vacuous-verifier control. Revoke
   plus issue over one id in one transaction is green, but over a live
   id, where this item asks it of an expired but unrevoked one.
   Stale generation cannot be reached through the specified lifecycle
   while `revoke_all_grants` clears the register; conformance evidence
   MUST state that structural argument rather than report an unrun
   stale-record test as passing. The key results above concern the
   JubJub/k256 baseline; P-256 profile results are distinguished in
   Implementation and item 12.
3. **Owner liveness.** A pending owner signature still verifies after a
   grant call; a permissionless deposit between grantee signing and
   submission does not invalidate the grantee's call (GR-5; AUTH-8).
   Status: green on node, both legs, with the second landing the
   signature made before the intervening deposit rather than a re-signed
   call, because the qualified coin's index was resolved by prove-only
   trials before the grantee signed (section 6.5).
4. **Redirect attack suite.** Open redirect; injection with a `state`
   mismatch; `iss`, `aud` mismatch; expired request; replayed `nonce`;
   proof failure; `clientDataJSON.origin` mismatch; an `rdns:`
   `client_id` by navigation refused; a `client_id` outside the
   productions of section 4.4 refused; a `request` value re-serialised
   in transit (whitespace or key order changed) whose proof therefore
   fails; a repeated `request` or `proof` parameter refused; an object
   with an unknown, duplicated, or wrongly typed member refused; a
   `proof` value altered with `request` unchanged refused; an r1 proof
   under the identity key refused; an envelope-1 withdraw refused; a
   non-canonical or low-order `read_pk` refused; a `grant_id` or opening mismatch on
   return treated as compromise; a narrowed request whose returned scope
   opens `object_commit`; a sibling-origin assertion against the
   authoriser credential; fragment leakage under a logging reverse proxy
   and in browser history before and after `replaceState` (GR-16,
   GR-17).
5. **Cross-implementation vectors.** Every derivation of section 4,
   including a read-only `object_commit`, `rp_commit` for an r1 and a
   non-r1 grant, and one recipient projection per kind; every challenge
   of section 6.3 with the qualified coin written out element by
   element; `envelope_digest`, `origin_hash`, `request_digest` over a
   published `request` parameter string with its decoded object and
   detached `Proof` beside it, `signin_digest`, the `read_pk`
   derivation, and GrantViewSeal,
   bit-identical between the compiled contract, the TypeScript client,
   and a Rust implementation linking no compiled module; `pk`
   normalisation from SEC 1 and the `v1` wire form; one off-chain
   signature vector per arm and a high-S vector per ECDSA arm that MUST
   verify; negative vectors for weak, identity, and low-order keys
   (GR-3, GR-4, GR-14). Status: green offline for every derivation of
   section 4; the `k1` and `v1` challenges of section 6.3 (unshielded,
   shielded, and shielded-to-contract, with the qualified coin written
   out) and the six lifecycle challenges; `envelope_digest`;
   `origin_hash`; and the preimage-width table of section 6.3, agreed
   three ways (the compiled circuits, a TypeScript recipe, and a Rust
   implementation linking no compiled module) over 34 published vectors
   with three pinned signatures, the high-S `k1` twin included. A `v1`
   challenge commits to its own nonce point and grinding nonce, so it
   cannot exist before it is signed and a suite verifies such a
   signature rather than pinning it. Pending are the r1 challenges and
   signature vectors agreed across the three implementations,
   `request_digest`, `signin_digest`, the `read_pk` derivation,
   GrantViewSeal, and the complete negative key-vector corpus. The later
   [scope:v2 vector file](../../../contract/src/tests/vectors/grants-v3.json)
   contains 36 Compact/TypeScript/Rust derivation vectors for the caller
   extension. P-256/OpenSSL verification fixtures are separate evidence,
   not a substitute for the full three-implementation corpus.
6. **Deploy budget.** The full submitted `spec_version = 3` roster deployed in
   waves within the per-block parameters; authority retirement after the
   last wave; a `spec_version = 1` account shown unable to gain grants
   (GR-3, GR-13; Backwards Compatibility). Status: partial. Green on
   node for the three-wave deploy of the thirty-circuit roster at the
   measured per-update ceiling of section 6.7, each update at the
   authority counter read from chain, and for the retirement landing in
   the same update as the last batch. This is historical v2 evidence.
   The caller guide separately records seven waves for its v3
   thirty-six-circuit snapshot, and the WebAuthn profile guide records
   ten for its fifty-two-circuit snapshot. A conformance report MUST
   identify the source revision, arm roster, toolchain and measured
   verifier inventory it deploys. Missing: the pre-grants compatibility
   control, which requires building the historical v1 source and
   exercising the attempted grant installation; source availability in
   Git history is not evidence that this experiment has run.
7. **Read handover.** Read-only grant; seal, decrypt, verify the secret
   against `enc_key`, inbox walk with commitment verification; rotate-
   before-share hides spent history; a second read grant issued and the
   first re-sealed and still reading; rotate on revoke blinds the revoked
   delegate and the remaining one is re-sealed; re-seal to the bound key
   succeeds and to another key is refused; a forged inbox entry is
   quarantined (GR-18).
8. **Agent and self grantees.** A jubjub grant to the reference signer in
   the non-browser binding with its structured response, a spend within
   scope, the scope mirrored as a PE-1 policy; a `self:` grant issued
   from the owner's client with no authoriser page (GR-1, GR-17).
9. **Composition.** Revoke plus issue in one transaction; batch issuance
   in one transaction; two grant calls under one grant with consecutive
   nonces; reordering refused (GR-5, GR-11, GR-13). Status: green on
   node. Each of the three rides one transaction, the third appending
   both change entries in it, and the reordered pair is refused at
   admission on a transcript read mismatch rather than by a signature
   failure (section 7.4). One run saw the reordered pair included with
   only one of its two calls applied, which is the same verdict by
   another route; a record advancing by two under the reordered graft
   would be the counter-example.
10. **Change and concurrency.** A grantee shielded spend with the change
    entry appended in the same transaction under one-hop; a
    `rotate_enc_key` between signing and submission aborts the call with
    no orphaned change; owner and grantee selecting the same coin (the
    loser refused by the node as a double spend, no mis-spend); a coin
    above `max_coin_value` aborts (GR-7; INV-3, INV-4, INV-5, INV-6).
    Status: partial. Green on node for the change entry appended in the
    same transaction under one hop, for a coin above `max_coin_value`
    aborting, and for the same-coin race: the owner's call landed, the
    grantee's was refused by the node as a double spend with the grant
    record untouched, and the same grantee call rebuilt against the
    post-spend state still proved, so the refusal is the ledger's and
    not the prover's. The same-transaction reading rests on the contract
    source, where the append is inside the circuit, and on the chain
    showing no other contract action between the call and the appended
    entry; the suite's own assertion is the `inbox_count` delta across
    the call, which is weaker. The historical
    [stale-`enc_pk` row](../../../contract/evidence/grants-e2-rejections-grants-conformance-rejections.json)
    also signs a JubJub shielded grant call, includes the owner's
    `rotate_enc_key_with_k256`, then attempts the original call. It is
    refused at build with `stale encryption key`; no transaction is
    submitted, and all six recorded fields remain unchanged, including
    `inbox_count` at 5 before and after. This establishes the rotation
    refusal and no inbox append in that run. It does not include a
    complete-state or UTXO snapshot and does not by itself exhaust the
    no-orphaned-change obligation.
11. **Kill totality.** `revoke_all_grants` and a recovery epoch bump each
    inert every record; re-issue under the new generation or epoch
    succeeds (GR-9). Status: satisfied for the recorded JubJub/k256
    paths. Historical E2 holds `revoke_all_grants`: the generation bumped,
    the register cleared, a pre-kill grant refused as `unknown grant`
    with its recorded snapshot unchanged, and a
    re-issue under the new generation spending. The
    [23 October 2026 recovery node run](../../../contract/evidence/grants-completion-20261023/rec-1-9-recovery-conformance.json)
    holds a pre-existing JubJub grant and a pending-window k256 grant
    invalid across two recovery epochs, each leaving exactly the
    authorised successor device. Successor re-issuance admits fresh
    unshielded spends on both arms; prior-incarnation signatures with
    fresh openings and nonce zero fail. Refusals occur during client
    build with the tracked snapshots unchanged, as detailed in Implementation.
12. **r1 grantee.** End-to-end issuance, spend and revocation under an
    explicitly supported WebAuthn profile, with profile-invalid and
    identity-key assertions refused and `rp_commit` opened at the seam
    (GR-14, GR-15). For the reference `wa-json134` profile, an
    extension-bearing assertion is a negative case; another permitted
    profile MUST state its own envelope acceptance rules. Status: satisfied
    for the bounded `wa-json134` unshielded path.
    The [23 October 2026 node run](../../../contract/evidence/grants-completion-20261023/p256-grants-conformance.json)
    holds `wa-json134` issuance, three unshielded grantee spends,
    revocation/re-issuance, RP binding, and sixteen build-time negative
    cases, including a valid ES256 signature over unsupported client JSON.
    The separate [three-operation fixture](../../../contract/evidence/grants-completion-20261023/p256-grant-refusals.json)
    uses the original constructor, circuits and verifier keys to hold
    identity-key, ED-flag and genuine signed-extension-suffix refusals
    against indexed state, plus an ordinary-authentication control.
    Implementation distinguishes its runtime point, circuit policy and
    fixed-size encoding boundaries from the full-account spend evidence.
    The software authenticator establishes neither the connection ceremony
    nor general browser support, and these runs do not cover P-256 shielded
    grant spends or successful forwarded P-256 calls.

## References

**Midnight documents**

- [MPS-0018](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md):
  the parent problem statement.
- [MIP-0012](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0012-native-asset-custody.md):
  Contract Custody of Midnight-Native Assets (the seam, INV-1 to INV-8,
  the viewing capability, R9, S5).
- [MIP-0013](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0013-account-authorisation.md):
  Multi-key Account Authorisation for Custody Contracts (the device
  seam and AUTH-1 to AUTH-10; combined conformance is specified in section
  12 of this proposal); its
  [discussion](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/244)
  (the VRF objection, R12).
- [MIP-0030](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0030-ecdsa-account-authorisation.md):
  ECDSA Authorisation for Custody Accounts (Draft); signing boundary,
  ECDSA validation, WebAuthn profiles and policy downgrade protection.
- [MIP-0021](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0021-domain-separation.md):
  Domain Separation Convention; the naming compatibility question is
  stated in section 14.
- [MIP-0003](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0003-ecdsa-support.md):
  ECDSA support (the connector `signData` `scheme` discriminator).
- [MIP-0008](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0008-caip-2-network-identifiers.md):
  CAIP-2 network identifiers (the wire `chain` member).
- [MIP-0015](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0015-wallet-derived-deterministic-secrets.md):
  Wallet-derived deterministic secrets (consent and error vocabulary).
- [MPS-0003](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0003-caip-support.md):
  CAIP support.
- [MPS-0015](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0015-agent-identity.md):
  Agent identity (the `mais:` grantee identity); the Midnight Agent
  Identity Standard
  ([PR #110](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/110)).
- [MPS-0027](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0027-domain-separation.md):
  Domain Separation for Midnight Hash Constructions (the problem
  statement addressed by MIP-0021; it does not supply a tag registry).
- [MPS-0029](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0029-compact-caller-identity.md):
  Caller identity in Compact circuits.
- Upstream proposals and discussions: Private Mandate Tokens
  ([PR #251](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/251),
  R13); a ZK-attested scoped-grant primitive offered as prior art
  ([#223](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/223));
  `persistentHash` byte framing
  ([#260](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/260)).
- The Midnight dApp connector API specification (scheme strings, the
  `midnight_signed_message:32:` prefix, `Rejected` and
  `PermissionRejected`, `rdns`); the Open Wallet Standard (policy engine
  PE-1 to PE-7).

**External standards**

- RFC 2119; RFC 3986 (unreserved characters); RFC 4648 (base64url,
  section 5); RFC 6749 (OAuth 2.0 parameter names, section 4.1.2.1
  errors); RFC 9700 (OAuth 2.0 Security Best Current Practice, sections
  2.1, 4.1.3, 4.2, 4.10); RFC 9207 (`iss`); RFC 9101 (OAuth 2.0
  JWT-Secured Authorization Request, the request-object precedent and
  the claim names, R21); RFC 7515 (JSON Web Signature, the
  sign-the-transmitted-bytes and detached-signature precedents, R21);
  RFC 6454 (web origin); RFC 7748 (X25519, clamping, contributory
  behaviour); RFC 5869 (HKDF).
- W3C Web Authentication Level 3 (client data, authenticator data, RP ID
  scoping, the PRF extension, Related Origin Requests).
- SEC 1 (point encodings and validation); the Jubjub curve specification
  (Zcash protocol specification, section 5.4.9.3).
- CAIP-2, CAIP-10, CAIP-25, and CAIP-217 (the scope objects a CAIP-25
  session carries, section 11.1).
- EIP-4361 Sign-In with Ethereum (the sign-in message shape,
  section 10); UCAN (prior art for an explicitly stated never-expiring
  capability, section 5.1); NEAR access keys and wallet login (the
  redirect flow whose defects section 9 closes).

**Evidence**

The reference source and evidence links below are pinned to Passport
commit `3fa1b32585032305f3dbef15476dd5fe7fca8a4b`. The reports retain the
compiler, ledger and schema versions of their respective runs; a pinned
current snapshot does not turn a historical v2 run into v3 conformance.

- [Reference contract](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/contracts/account.compact)
  and [P-256 arm](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/contracts/account-p256.compact).
- [E1 measurements](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/GRANTS-E1.md)
  and [recipe vectors](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/src/tests/vectors/grants-e1.json).
- [E2 conformance and unshielded follow-up](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/GRANTS-E2.md),
  with its per-group JSON evidence under
  [contract/evidence](https://github.com/midnightntwrk/passport/tree/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/evidence).
- [E3 block-time measurements](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/GRANTS-E3.md).
- [Caller restriction and v3 evidence](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/GRANTS-CALLER.md).
- [P-256 measurements and evidence boundaries](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/P256-MEASUREMENTS.md).
- [Recorded recovery conformance run](https://github.com/midnightntwrk/passport/blob/3fa1b32585032305f3dbef15476dd5fe7fca8a4b/contract/evidence/rec-1-9-recovery-conformance.json).

## Acknowledgements

The Input Output Group Advanced Research and Creativity department and
the Midnight Foundation. The connector specification and MIP-0015
maintainers, for the vocabulary this MIP reuses. The scoped-grant
prior-art offer in discussion #223 is recorded in References; evaluation
and attribution of any adopted contribution remain follow-up work.

## Copyright Waiver

This document is licensed under the Apache License, Version 2.0, and its
contribution is made under the Midnight Foundation Contributor License
Agreement.

Portions of this document were drafted with the assistance of a large
language model. The named authors reviewed all content and are solely
accountable for it.
