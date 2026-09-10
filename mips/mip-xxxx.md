---
MIP: X
Title: Midnight Message Signing
Authors:
  - Andrzej Kopeć (kapke)
Status: Draft
Category: Standards
Created: 2026-09-08
Requires: "MIP-0003: ECDSA signature support"
Replaces: none
MPS: none
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

A DApp can ask the user's wallet to sign arbitrary data, by calling `signData` on the [DApp Connector API](https://github.com/midnightntwrk/midnight-dapp-connector-api/). The wallet signs with the user's unshielded key and returns the signature. This MIP specifies that procedure end to end: which bytes the wallet signs, and the algorithm anyone follows to check the result afterwards. Checking a signature needs no wallet, no connector and no Midnight-specific software.

The mechanism is already deployed; what is missing is a findable, complete, normative description. This MIP lifts the procedure out of the [DApp Connector API specification](https://github.com/midnightntwrk/midnight-dapp-connector-api/blob/main/SPECIFICATION.md) and settles the points that specification leaves silent, then pins the result with machine-checkable test vectors. Conformance is a break with what ships: no wallet today returns a signature this document accepts. Both schemes the connector names, `schnorr_bip340` and `ecdsa_secp256k1_sha256`, are specified normatively.

## Motivation

The procedure's only normative statement is a single sentence in the Signing section of the DApp Connector API specification, backed by doc comments on the API's TypeScript types. Neither is where an implementer looks for a signing algorithm. No other document describes the message construction: MIP-0003 extends the same `Signature` response type with the `scheme` discriminator but says nothing about what is signed, and the Wallet specification is silent.

Multiple known implementations already disagree, and several reached the same misreading independently, in codebases that share no code. The specification, not the code, is the thing to correct. An implementation that never finds the Signing sentence omits the prefix altogether, losing the domain separation it provides. Elsewhere the text does not settle what the length counts, what `data` returns, how malformed input is handled, which value the signature binds, or how the wire fields are encoded. This MIP is meant to be both findable and complete.

## Specification

The key words MUST, MUST NOT, SHOULD and MAY are to be interpreted as described in BCP 14 (RFC 2119 and RFC 8174) when, and only when, they appear in all capitals, as shown here.

This MIP profiles the DApp Connector API's existing types. It changes one of them: `scheme` is optional on the connector's `Signature` and defaults there to `schnorr_bip340`; here it is required. [MIP-0003](./mip-0003-ecdsa-support.md#dapp-connector-api) foresaw that change when it introduced the field. Until the connector API's next major version ships — the one [§8](#8-versioning) calls for — this document governs where the two disagree. A few rules presuppose the connector: the error shape of §2, and the JavaScript-specific typing rules of §2 and §7.

```ts
signData(data: string, options: SignDataOptions): Promise<Signature>;

type SignDataOptions = { encoding: 'hex' | 'base64' | 'text'; keyType: 'unshielded' };

type Signature = {
  scheme: 'schnorr_bip340' | 'ecdsa_secp256k1_sha256';
  data: string;
  signature: string;
  verifyingKey: string;
};
```

### 1. Message construction

The signed message `M` is the byte concatenation

```
M = utf8("midnight_signed_message:")  ‖  dec(L)  ‖  utf8(":")  ‖  P
```

where `P` is the decoded payload (§2). This document calls `midnight_signed_message:` the *prefix literal*, and everything in `M` ahead of `P` the *prefix*.

`M` is a function of `P` alone: no `encoding`, no `keyType`, no network identifier, timestamp or nonce enters it. Two calls whose payloads decode to the same `P` therefore produce the same `M`, and their signatures are interchangeable. A DApp relying on a signature for authentication SHOULD place both a nonce and its own identity in `P`: without the nonce an old signature replays, and without the identity one DApp's challenge serves at any other.

- The prefix literal is the 24-byte ASCII string `midnight_signed_message:`, in hexadecimal `6d69646e696768745f7369676e65645f6d6573736167653a`.
- `L` MUST count bytes of the **decoded** payload, never characters of the input string.
- `dec(L)` MUST be the shortest ASCII decimal representation of `L`: digits U+0030–U+0039 only, no leading zeros, `0` when `L` is zero, no sign, no grouping separators.
- This MIP imposes no upper bound on `L`. A *producer* — an implementation of `signData` — MAY refuse an oversized payload with `InvalidRequest`; a producer that accepts one MUST use its full byte length.

### 2. Payload derivation

`P` is derived from `data` according to `options.encoding`.

- `options` MUST be an object and `data` MUST be a string.
- `options.encoding` MUST be present and exactly one of `'hex'`, `'base64'` and `'text'`; a missing or unrecognised `encoding` MUST be rejected rather than defaulted or inferred.
- `options.keyType` MUST be present and MUST be `'unshielded'`; any other value, and its absence, MUST be rejected.
- Validation MUST complete before any signing occurs and before any approval prompt the producer shows.
- On failure the producer MUST throw an `APIError` per the connector specification's Errors section, with `code: 'InvalidRequest'`, and MUST NOT return a `Signature`.
- A producer MUST NOT fall back to another encoding when the declared one fails to parse: there is no sniffing and no retry. A rejected input MUST NOT be repaired and then accepted: no truncating, padding, stripping or substituting a replacement character.
- The empty string MUST be accepted under all three encodings, yielding `L = 0`.

<table>
  <thead>
    <tr>
      <th scope="col"><code>encoding</code></th>
      <th scope="col"><code>P</code></th>
      <th scope="col">Accepted</th>
      <th scope="col">Rejected</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td valign="top"><code>hex</code></td>
      <td valign="top">the decoded bytes</td>
      <td valign="top">
        <ul>
          <li>hexadecimal digits in either case — <code>^[0-9a-fA-F]*$</code></li>
          <li>even length</li>
        </ul>
      </td>
      <td valign="top">
        <ul>
          <li>odd length</li>
          <li>a <code>0x</code> or <code>0X</code> prefix</li>
          <li>whitespace and any other non-hexadecimal character</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td valign="top"><code>base64</code></td>
      <td valign="top">the decoded bytes</td>
      <td valign="top">canonical base64 under RFC 4648</td>
      <td valign="top">everything else — unpadded, whitespace, the URL-safe alphabet, non-zero trailing bits — much of which a permissive decoder accepts</td>
    </tr>
    <tr>
      <td valign="top"><code>text</code></td>
      <td valign="top">the UTF-8 encoding of the string as given</td>
      <td valign="top">any string encodable as well-formed UTF-8</td>
      <td valign="top">a string containing an unpaired surrogate, which RFC 3629 forbids encoding</td>
    </tr>
  </tbody>
</table>

The `reason` accompanying an error is human-readable and unspecified; a DApp MUST NOT branch on it.

### 3. Digest and signature schemes

Let `h = SHA-256(M)` — a single untagged application over the whole of `M`, prefix included. The signature MUST bind `h` under both schemes, and MUST NOT bind `SHA-256(P)`, a tagged hash of `M`, or `SHA-256(h)`.

- `schnorr_bip340` — sign and verify with `h`, the 32 bytes, as BIP-340's message input.
- `ecdsa_secp256k1_sha256` — sign and verify `h` as the ECDSA digest, i.e. ECDSA-with-SHA-256 over `M`.

The key a wallet signs with MUST be the key whose address it reports through the connector's unshielded-address API, `getUnshieldedAddress`. Which address derivation applies follows from the `scheme` the returned `Signature` declares; both derivations are given in [MIP-0003](./mip-0003-ecdsa-support.md#address-generation). `getUnshieldedAddress` takes no scheme argument and is answered before `signData`, so a wallet holding an unshielded key under each scheme MUST pick one of those keys for the lifetime of a `ConnectedAPI` — the object the connector's `connect` resolves to — and report that key's address.

Signatures under either scheme MAY be randomised, so two calls signing the same `M` may return different signatures. Nonce derivation belongs to the scheme, not to this procedure, and this document fixes none for either. A verifier therefore:

- MUST NOT treat two differing signatures over the same `M` as an error;
- MUST NOT require any particular nonce derivation, which a signature does not expose for checking.

### 4. The `scheme` discriminator

Producers MUST emit `scheme` on every `Signature`, spelled exactly `'schnorr_bip340'` or `'ecdsa_secp256k1_sha256'`, and MUST NOT emit any other discriminator field. A verifier resolves it as step 1 of [§7](#7-verification) prescribes. Verifiers MUST ignore fields they do not recognise and MUST NOT read any field other than `scheme` as a discriminator.

The scheme is not negotiated: it follows from the key material the producer holds. The connector specification's Signing section already requires a DApp to handle both.

### 5. Wire encoding of `Signature`

- `data`, `signature` and `verifyingKey` MUST each be bare lowercase hexadecimal (`^[0-9a-f]*$`) of even length, with no `0x` prefix, no separators, no whitespace and no version or tag bytes. Producers MUST emit lowercase; verifiers MUST reject any value not in that canonical form.
- `data` MUST be the hexadecimal encoding of the whole of `M`, prefix included.
- `signature` MUST be exactly 128 hexadecimal characters under both schemes. Its two 32-byte halves are `r ‖ s`, both big-endian, for `ecdsa_secp256k1_sha256`, and BIP-340's `bytes(R) ‖ bytes(s)` for `schnorr_bip340`.
- `verifyingKey` MUST be exactly 64 hexadecimal characters for `schnorr_bip340`, a 32-byte BIP-340 x-only key, and exactly 66 for `ecdsa_secp256k1_sha256`, a 33-byte SEC1 compressed point leading with `02` or `03`. An uncompressed SEC1 key MUST be rejected.
- For `ecdsa_secp256k1_sha256` a producer MUST emit `s <= (n-1)/2`, where `n` is the secp256k1 group order; a signing routine that yields a high `s` MUST have it replaced by `n - s` before the `Signature` is returned. `schnorr_bip340` carries no such bound.

### 6. Worked examples (non-normative)

The declared encoding, not the string, determines the bytes:

| `encoding` | `data`, `L` | `M` (hex) |
|---|---|---|
| `text` | `decade`, 6 | `6d69646e696768745f7369676e65645f6d6573736167653a363a646563616465` |
| `hex` | `decade`, 3 | `6d69646e696768745f7369676e65645f6d6573736167653a333adecade` |
| `base64` | `3sre`, 3 | `6d69646e696768745f7369676e65645f6d6573736167653a333adecade` |

The second and third rows are the same message, as §1 requires, so their signatures are interchangeable; `h = SHA-256(M)` for both is `3e1a914a91ed897e6e20423b3761a6026a9cdd039867f74e509d4f8af885f0e4`. The vector `schnorr-hex-decade` carries the `Signature` for the second row, and verifies under [§7](#7-verification) against the three bytes `decade`.

### 7. Verification

Verification takes the `Signature` object of the typings above together with `P_exp`, the payload bytes the verifier asked to be signed. A DApp derives `P_exp` by applying §2 to the exact `data` and `encoding` it passed to `signData`. The `Signature` is untrusted on arrival, so `P_exp` MUST NOT be recovered from it or informed by any of its fields. A verifier acting for another party MUST receive `P_exp` from the party that made the request, over a channel independent of the `Signature`. That party may send the `data` and `encoding` instead, and the verifier derives `P_exp` per §2.

Verification establishes that the holder of `verifyingKey` signed `M`, not who that holder is: a wallet that ignores §3's key-binding rule can sign under a key it generated for the purpose and still pass every step below. A verifier that attributes any meaning to the signature MUST therefore also check `verifyingKey` against a key or address it obtained independently of the `Signature`, deriving that address per [MIP-0003](./mip-0003-ecdsa-support.md#address-generation) for the resolved scheme.

Throughout the steps below, a verifier:

- MUST reach a verdict on every input. An untrusted `Signature` crosses an in-page boundary, so its property accessors may themselves throw; such a throw is a rejection, and MUST NOT escape as though verification had not concluded.
- MUST read each field of the `Signature` once, and carry the value it first read through every later step.
- SHOULD bound the length of `data` it will decode, rejecting an oversized value before step 3. That bound is a local resource guard rather than a conformance rule: §1 fixes no upper bound on `L`, so two verifiers may draw it in different places and a bounded verifier may reject an otherwise valid signature.
- MUST NOT, on a step 9 failure, retry with another message construction — not the bare payload, not `SHA-256(P)`, not `M` unhashed.

1. Reject unless the `Signature` is a non-null object that is not an array. Then read `scheme` and reject unless it is exactly one of the two literals; an absent or `null` `scheme` is rejected exactly as an unrecognised one.
2. Reject any of `data`, `signature` and `verifyingKey` that is absent, is not a string, or does not match the §5 wire form for the resolved scheme.
3. Let `M := hexDecode(data)`.
4. Reject unless `M` begins with the 24-byte prefix literal.
5. Read the maximal run of ASCII digits at offset 24. Reject unless it is non-empty, has no leading zero unless it is the single digit `0`, and is immediately followed by the byte `0x3a` (`:`). Let `L` be its value.
6. Let `P := M[k..]`, where `k` is the offset just past that colon; reject unless `len(P) == L`.
7. Reject unless `P` equals `P_exp` byte for byte.
8. Let `h := SHA-256(M)`.
9. Verify per scheme: `schnorr_bip340` — BIP-340 verify with message `h` and the x-only key; `ecdsa_secp256k1_sha256` — ECDSA verify with `h` as the digest and the compressed key, additionally rejecting `s > (n-1)/2`.
10. Accept only if every step passed.

### 8. Versioning

Requiring `scheme` is a breaking change to the connector's `Signature` type, so a new major version of the DApp Connector API will be released to signal conformance with this document. Versioning of this procedure belongs to that API. A later MIP that supplements the procedure — adding a third signature scheme, for instance — may in turn require a new version of that API.

## Rationale

Many choices here are settled in advance by the two schemes this document must support and by the precedent of EIP-191.

CIP-8, the Cardano analog, wraps the payload in COSE_Sign1 and CBOR. Midnight's mechanism already ships as a plain byte concatenation. Adopting a serialization format now would invalidate every deployed signature and add a parser to every verifier, buying no property this document needs.

The prefix is there for the domain separation Security Considerations sets out; the decoded byte length makes `M` self-describing, so a message truncated or extended after construction fails step 6 of [§7](#7-verification) instead of verifying over a shorter payload.

`data` carries the whole of `M` so that `M` can be reconstructed from the `Signature` object alone; echoing the caller's input would force every verifier to know the original `encoding` and rebuild `M` from it.

## Path to Active

For a Standards MIP, Active means that deployed wallets and DApps run the procedure specified here and that the connector specification points to this document for it. `signData` is an off-chain wallet operation, so no network upgrade is required; the path runs through the connector specification's next major revision, corrections in shipping producers, and independent conformance against the vectors.

### Acceptance Criteria

- A verifier written by an implementer who is not an author of this MIP accepts every valid vector and rejects every `kind: "verification"` vector.
- The DApp Connector API specification's Signing section cites this MIP as the normative description of the procedure, and a released major version of `@midnightntwrk/dapp-connector-api` carries `scheme` as a required field of `Signature`.
- At least two `signData` producers, independent of each other and of this MIP's authors, meet the producer rules under Testing.

### Implementation Plan

Adoption is documentary and per-implementation, and no step below depends on the others:

- Cite this MIP from the connector specification, make `scheme` required in its `Signature` type, delete the default-to-`schnorr_bip340` rule from the Signing section and from the `Signature` doc comment, and publish that as a new major version.
- Change producers to return `hex(M)` in `data` and to emit `scheme`.
- Tighten payload validation wherever malformed input is currently repaired.
- Apply the prefix wherever it is currently omitted.
- Correct the description of both signature halves as "the scalars `r` and `s`", wrong for `schnorr_bip340`, in all three places it appears: `SPECIFICATION.md`, the `Signature` doc comment in `src/api.ts`, and [MIP-0003's transaction-format section](./mip-0003-ecdsa-support.md#transaction-format-and-signature-verification).

## Backwards Compatibility Assessment

`signData` is an off-chain wallet operation: this MIP requires no hard fork, no ledger change and no consensus change. What changes is what counts as a conformant `Signature`, and no deployed producer emits one: none returns `hex(M)` in `data`, and `scheme` has so far shipped only in a prerelease of the API. A verifier following [§7](#7-verification) therefore rejects every signature a shipping wallet returns today. The type itself changes too: `scheme` is required here and optional in the connector's `Signature`, which is why §8 calls for a major version. This MIP deliberately defines no transition period and no dual acceptance: a verifier permitted to treat a structurally invalid `data` as a raw echo reopens the ambiguity being closed.

## Security Considerations

**The prefix is a security control, not a style choice.** It exists so that a value signed through `signData` cannot also be a well-formed Midnight transaction. Under `ecdsa_secp256k1_sha256` that holds as long as no transaction's signing message begins with the 24-byte prefix literal. Under `schnorr_bip340` the signature binds `h` rather than `M`, so it holds as long as no transaction's BIP-340 message equals `SHA-256` of a prefixed message. This MIP claims no proof of domain separation; an implementation that omits the prefix removes the control.

**The producer controls the bytes that get signed.** A malicious or buggy producer can return a valid signature over a message the DApp never asked for, and silently repairing malformed input has the same effect. Step 7 of [§7](#7-verification) catches this, and it is not optional. A verifier that holds no `P_exp` cannot verify under this MIP at all, and MUST NOT report the weaker fact — that some message was signed — as verification. Because a signature cannot be withdrawn once made, [§2](#2-payload-derivation) forbids repair before anything is signed rather than leaving it to a later check.

**ECDSA signatures are malleable.** Without the low-`s` bound, a third party can derive a second valid signature over the same `M` from an existing one, which is why [§5](#5-wire-encoding-of-signature) requires the bound of producers and [§7](#7-verification) step 9 of verifiers. No Midnight verifier enforces it today, so a verifier cannot assume the signatures reaching it already meet the bound.

**The signed message binds no context.** `M` is a function of `P` alone ([§1](#1-message-construction)): it carries no nonce, no challenge, no timestamp, no network identifier and no DApp identity. A DApp that needs a signature to bind any of those puts them in `P` itself; [§1](#1-message-construction) recommends a nonce and the DApp's own identity for authentication. That identity closes cross-DApp reuse — a signature made for one DApp presented at another — but not relay: a hostile site can take the challenge a genuine DApp issued, identity and all, present it to the user as its own, and hand the resulting signature back to that DApp. This document takes no position beyond that.

## Implementation

The components that change are the DApp Connector API specification and wallets implementing `signData` — chiefly their payload validation and the `data` they return.

## Testing

The vectors are at [`./mip-xxxx/vectors.json`](./mip-xxxx/vectors.json). They cover the bytes; §2's rules about the shape of `data` and `options` cannot be written down in the file, and a harness has to check those by other means. No valid vector's `input` carries `keyType`, by convention rather than omission: a producer harness replaying one MUST supply `'unshielded'`.

Two of the file's fields carry rules rather than data. `schemaVersion` is an integer, and a harness reading the file MUST reject a value it does not recognise. Every vector with `valid: false` carries a `kind` naming its class: `payload` for a §2 rejection a producer must make, `verification` for a §7 rejection a verifier must make. A harness MUST switch on `kind` rather than infer the class from which fields are present.

A conformant producer MUST:

- reproduce, from each valid vector's `input`, the four values that vector pins: the payload, its length, the message and the digest;
- reproduce a valid vector's `signature.signature` byte for byte when either determinism condition holds:
  - its `signatureDeterminism` is `rfc6979` and the producer does derive its nonces that way, or
  - its `signatureDeterminism` is `bip340-aux-fixed` and the producer accepts external auxiliary randomness (`auxRandHex`);
- return, for every valid vector, a `Signature` that verifies under §7 against the vector's `payloadHex`;
- reject every `kind: "payload"` vector's `input` with `InvalidRequest`.

For a producer meeting neither determinism condition, that verification is the only check on the signature it returns.

A conformant verifier MUST:

- accept every valid vector's `signature`, with that vector's `payloadHex` as `P_exp`;
- reject every `kind: "verification"` vector's `signature` against its `payloadHex`.

A verifier never sees an `input`, an `encoding` or an error code.

`expectedFailureStep` names the §7 step at which a verifier executing the algorithm in order first fails, and each such vector fails at exactly that step and passes every step before it. That field is diagnostic: rejection is the pass criterion, and a verifier that returns only accept or reject MAY ignore it.

## References

- [DApp Connector API specification](https://github.com/midnightntwrk/midnight-dapp-connector-api/blob/main/SPECIFICATION.md) — the Signing section this MIP lifts and completes, and [the API's types](https://github.com/midnightntwrk/midnight-dapp-connector-api/blob/main/src/api.ts)
- [MIP-0003: ECDSA signature support](./mip-0003-ecdsa-support.md) — the ECDSA scheme, its key and address derivation, and its ledger-level low-`s` rule
- [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki) — Schnorr signatures for secp256k1
- [RFC 4648](https://www.rfc-editor.org/rfc/rfc4648) — base64 encoding (Section 4) and canonical encoding (Section 3.5); [RFC 3629](https://www.rfc-editor.org/rfc/rfc3629) — UTF-8; [RFC 6979](https://www.rfc-editor.org/rfc/rfc6979) — deterministic ECDSA nonces, and its Abstract's guarantee that they are verifiable by unmodified verifiers
- [FIPS 180-4](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf) Section 6.2 — SHA-256
- [SEC 1 v2.0](https://www.secg.org/sec1-v2.pdf) Section 2.3.3 — compressed elliptic-curve point encoding; [SEC 2 v2.0](https://www.secg.org/sec2-v2.pdf) Section 2.4.1 — secp256k1's parameters, including the group order `n`
- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) — the requirement keywords
- [BIP-146](https://github.com/bitcoin/bips/blob/master/bip-0146.mediawiki) — signature encoding malleability; the low-`s` bound, proposed as consensus and left unactivated
- [EIP-2](https://eips.ethereum.org/EIPS/eip-2) — the low-`s` bound made binding for Ethereum transaction signatures, and explicitly not for signature recovery
- [EIP-191](https://eips.ethereum.org/EIPS/eip-191) — Ethereum's signed-data prefix, the closest analogue to §1's message construction
- [CIP-8](https://cips.cardano.org/cip/CIP-8) — the Cardano message-signing standard this document parallels

## Acknowledgements

Special thanks to the Wallet Working Group for reporting the issues and highlighting the need for this MIP.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
