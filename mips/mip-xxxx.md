---
MIP: X
Title: Sign-In with Midnight
Authors:
  - Gabriel Viganotti (bochaco)
Status: Draft
Category: Standards
Created: 2026-11-22
Requires: "MIP-0031: Midnight Message Signing; MIP-0008: CAIP-2 Network Identifiers for Midnight"
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

DApp backends authenticate users by asking the wallet to sign a challenge with `signData`, whose bytes [MIP-0031](./mip-0031-message-signing/mip-0031-message-signing.md) specifies. MIP-0031 binds no context into the message. That leaves an attack open: a hostile site relays a genuine DApp's challenge to the user and hands the signature back to that DApp.

This MIP defines Sign-In with Midnight (SIWM), a fixed text format for the `signData` payload. It follows EIP-4361 and the layout of [MIP-0033](./mip-0033-scoped-grants.md) §10. The message carries the requesting origin, the address, a CAIP-2 chain identifier ([MIP-0008](./mip-0008-caip-2-network-identifiers.md)), a nonce and a validity window. The MIP specifies a strict grammar, a wallet procedure that refuses to sign unless the message origin equals the browser-attested caller origin, and backend verification rules. Test vectors are included. No ledger, node or connector change is required.

## Motivation

Authentication is the most common use of `signData`, yet MIP-0031 §1 only says a DApp "SHOULD place both a nonce and its own identity in `P`". Every DApp invents its own challenge format, so wallets cannot recognise sign-in requests and sign opaque strings.

MIP-0031's Security Considerations names the resulting attack and leaves it open:

1. The user connects their wallet on `https://evil.example`.
2. `evil.example` obtains a sign-in challenge from `https://app.example.com`'s backend.
3. `evil.example` passes that challenge to `signData`, and the wallet signs it.
4. `evil.example` posts the signature to `app.example.com` and gets a session for the user.

The backend cannot detect this, even if it builds the challenge itself and checks its hash, because the relayed signature is over the genuine challenge. Only the wallet sees both the origin the message claims and the page that called it, and it can compare them only if the message has a known format. EIP-4361 solved this on Ethereum. Midnight has no counterpart.

MIP-0033 §10 uses a similar message, but only for custody-account grantee keys, without wallet involvement. MIP-0033 §1 leaves general sign-in to "a separate, not yet filed proposal". This MIP covers the ordinary, address-revealing case. It is not the unlinkable sign-in for which MIP-0033 reserves `midnight:account:signin`.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals.

### 1. Scope and terminology

This MIP profiles MIP-0031's payload `P`. MIP-0031's message construction, digest, schemes, wire encoding and verification are unchanged, so every SIWM signature is an ordinary MIP-0031 signature.

- **Marker**: the 49-byte ASCII string ` wants you to sign in with your Midnight account:` (with its leading space).
- **Message origin**: the RFC 6454 origin named by the message's first line (§2).
- **Caller origin**: the RFC 6454 origin of the browsing context that called `signData`, as the browser reports it to the wallet.
- **Relying party (RP)**: the backend that issues and verifies the challenge.

Out of scope: unlinkable (zero-knowledge) sign-in; custody-account grant sign-in, which remains MIP-0033 §10; shielded keys, since MIP-0031 signs with unshielded keys only; non-browser transports (QR pairing, deep links, remote signers), on which §4.2 makes wallets refuse; session management after sign-in; and new connector methods.

### 2. Message format

`P` MUST be well-formed UTF-8 with no byte-order mark and at most 8192 octets. Lines are separated by a single LF. `P` MUST NOT contain CR and MUST NOT end with LF. Fields appear in exactly this order, with no others:

```abnf
sign-in-message = origin-line LF
                  account LF
                  LF
                  [ statement LF LF ]
                  %s"URI: " uri LF
                  %s"Version: 1" LF
                  %s"Chain ID: " chain-id LF
                  %s"Nonce: " nonce LF
                  %s"Issued At: " timestamp LF
                  %s"Expiration Time: " timestamp
                  [ LF %s"Not Before: " timestamp ]
                  [ LF %s"Request ID: " request-id ]
                  [ LF %s"Resources:" 1*16( LF "- " uri ) ]

origin-line     = [ %s"http://" ] authority marker
marker          = %s" wants you to sign in with your Midnight account:"
authority       = host [ ":" port ]
host            = dns-name / %s"localhost" / %s"127.0.0.1" / %s"[::1]"
dns-name        = label *( "." label )          ; at most 253 octets
label           = ( lc-alnum [ *61( lc-alnum / "-" ) lc-alnum ] )
lc-alnum        = %x61-7A / DIGIT
port            = %x31-39 0*4DIGIT              ; 1..65535

account         = <lowercase Bech32m unshielded address>
statement       = 1*<UTF-8 code point>          ; 1..512 octets
uri             = <RFC 3986 absolute URI>       ; ASCII, at most 2048 octets
chain-id        = %s"midnight:" 1*32( ALPHA / DIGIT / "-" / "_" )   ; MIP-0008
nonce           = 22*64( ALPHA / DIGIT )
timestamp       = 4DIGIT "-" 2DIGIT "-" 2DIGIT %s"T" 2DIGIT ":" 2DIGIT ":" 2DIGIT %s"Z"
request-id      = 1*128( ALPHA / DIGIT / "-" / "." / "_" / "~" )
```

A parser MUST reject `P` unless every rule below holds, and MUST NOT repair input (trim, lowercase, normalise or reorder).

| Field | Rule |
|---|---|
| origin line | One spelling per origin, so the wallet and RP compare strings. The scheme is `https`, implied and never written. `http://` is allowed only for `localhost`, `127.0.0.1` and `[::1]`. Hosts are lowercase, with internationalised names in punycode. A port is present exactly when it is not the scheme default. |
| `account` | The address as `getUnshieldedAddress` returns it: lowercase Bech32m with prefix `mn_addr` on mainnet and `mn_addr_<network>` elsewhere (MIP-0003). The network segment MUST match `Chain ID`. |
| `statement` | Optional text from the RP. It MUST NOT begin with `URI: ` and MUST NOT contain C0/C1 controls, U+FEFF, or the bidirectional controls U+200E, U+200F, U+202A–U+202E and U+2066–U+2069. |
| `URI` | The message origin itself, or the origin immediately followed by `/`, `?` or `#`. |
| `Version` | Exactly `1`. |
| `Chain ID` | A MIP-0008 identifier such as `midnight:mainnet`. |
| `Nonce` | Generated by the RP from at least 128 bits of CSPRNG output. |
| `Expiration Time` | Required. Later than `Issued At` by at most 86400 seconds. RPs SHOULD use 600 seconds or less. |
| `Not Before` | If present, `>= Issued At` and `< Expiration Time`. |
| `Resources` | Informational URIs shown to the user. An RP MUST NOT treat them as authorisation scopes. |

Example (non-normative):

```
app.example.com wants you to sign in with your Midnight account:
mn_addr_preprod18anzyad48gzkclaerljwtwv0qrmw7mylj7entrs4mten77nsxl9qmyk8u9

Sign in to Example App. This request costs nothing and moves no funds.

URI: https://app.example.com/login
Version: 1
Chain ID: midnight:preprod
Nonce: k7Qz3VmR9pXw2LbN8tYc4HsD
Issued At: 2026-11-22T12:00:00Z
Expiration Time: 2026-11-22T12:10:00Z
```

### 3. Sign-in flow

1. The DApp reads the address with `getUnshieldedAddress` and sends it to the RP.
2. The RP builds the message. It takes the origin line and `URI` from its own configuration (never from request headers) and `Chain ID` from the network it serves, then draws the nonce and sets the times. It stores the exact bytes, keyed by nonce and bound to the client session, and returns the message.
3. The DApp calls `signData(message, { encoding: 'text', keyType: 'unshielded' })`.
4. The DApp posts the `Signature` and nonce to the RP, which verifies per §5.

An RP SHOULD build the message itself, so that it verifies against the bytes it issued (MIP-0031 §7's `P_exp`). An RP that accepts client-built messages MUST still apply every check in §5.

Note that the RP's checks prove the user signed *this RP's challenge*, not *which page they were on*. Relay protection rests on the wallet procedure in §4.

### 4. Wallet procedure

#### 4.1 Detection

A wallet MUST apply this section to every `signData` call whose decoded payload `P` contains the marker, whatever the declared `encoding`. Running detection on the decoded bytes means hex or base64 encoding cannot bypass it. Such a payload MUST either pass §4.2–§4.4 and be approved by the user, or be refused. It MUST NOT be signed as ordinary data.

Payloads without the marker are ordinary MIP-0031 data. A near-miss such as `Midnight Account:` is not detected, but no conforming RP accepts it either, because every payload a conforming RP accepts contains the marker and is therefore subject to the wallet checks.

#### 4.2 Caller origin

The wallet MUST take the caller origin from the browser (in an extension, the sender origin the messaging runtime reports), never from DApp-supplied data, and MUST check it on every call. The wallet MUST refuse if the origin is opaque (`null`), if the calling frame's origin differs from the top-level page's origin, or if no browser-attested origin is available.

#### 4.3 Checks

After MIP-0031 payload validation and before any prompt, the wallet MUST check, in order:

1. `P` parses under §2.
2. The message origin equals the caller origin exactly (scheme, host and port). Subdomains, parent domains and "related" origins do not match.
3. `Chain ID` matches the network of the current connection.
4. `account` equals the address `getUnshieldedAddress` reports for the current connection.
5. `Expiration Time` is in the future and `Issued At` is at most 300 seconds ahead of the wallet's clock.

On failure the wallet MUST throw an `APIError` with code `InvalidRequest`, and MUST NOT offer an override. A user declining on the approval screen yields `Rejected`.

#### 4.4 Approval screen

The wallet MUST show a dedicated sign-in screen, not the raw text as its primary view. The screen shows the verified origin (scheme and any port included), the address and network, the statement (marked as the site's words), the time to expiry, and any resources. The raw message SHOULD be available on request.

### 5. Relying-party verification

Given the `Signature` `S`, the issued bytes `P_exp`, and the RP's configured origins, chain ID, nonce store and clock, the RP MUST accept only if all of the following pass:

1. `S` verifies against `P_exp` per MIP-0031 §7.
2. `P_exp` parses under §2.
3. The message origin is one of the RP's configured origins. It MUST NOT be derived from the request's `Host`, `Origin` or `Referer` headers.
4. `Chain ID` equals the RP's chain ID.
5. The 32-byte address in `account` equals the address MIP-0003 derives from `S.verifyingKey` for `S.scheme`: `SHA-256(verifyingKey)` for `schnorr_bip340`, and `SHA-256("midnight:ecdsa:" ‖ verifyingKey)` for `ecdsa_secp256k1_sha256`.
6. The nonce was issued to this client session and not yet used. The RP MUST consume it atomically, whether or not the other checks pass.
7. With clock skew σ of at most 60 seconds (SHOULD): `Issued At ≤ now + σ`, `now < Expiration Time`, and `Not Before ≤ now + σ` if present.
8. Any `Request ID` or `Resources` the RP set match what it issued.

### 6. Versioning

The `Version` field versions the format, and this MIP defines `1`. Parsers MUST reject other values. Any change to fields or parse rules is a new version, defined by a MIP that amends or supersedes this one, and MUST keep the marker so that detection still applies. Clarifications that change no parse or check outcome are amendments. New MIP-0031 schemes apply automatically if they define an address derivation for §5 step 5.

## Rationale

- **A payload profile.** Defining only `P` keeps MIP-0031 unchanged and works on today's `signData`. The alternative, a wallet-attested origin inside the signed bytes (like WebAuthn's `clientDataJSON`), would let RPs check the origin themselves. But it means wallets signing bytes the DApp did not supply, which conflicts with MIP-0031 §2 and §7, or else a new connector method every wallet must ship first. It is left to a future MIP.
- **Detection rather than a `signIn` method.** A new method would need a connector release and every wallet's adoption before any DApp could use it. Until then DApps would fall back to unchecked `signData`. Detection protects users as each wallet adopts it, and a later `signIn` method could produce the same message. This is content detection, not the encoding sniffing MIP-0031 forbids.
- **Refusal, not warning.** EIP-4361 allows wallets to warn on a mismatch. Phishing pages tell users to click through warnings, and no legitimate flow needs a mismatch.
- **One spelling per origin.** Canonical origins make the wallet and RP comparison a string match, avoiding normalisation bugs.
- **Required, capped expiry.** Sign-in challenges are used within seconds. A cap bounds the harm from a leaked, unused signature.
- **Shared layout.** Matching EIP-4361 and MIP-0033 §10 gives users one familiar shape. The Midnight marker and closed field set keep the formats from being confused with each other.
- **Text over structured signing (EIP-712 style).** Text works today and reads correctly even on older wallets.

## Path to Active

Active means deployed wallets implement §4, RP libraries implement §5, and the DApp Connector API specification references this MIP. No network upgrade is needed.

### Acceptance Criteria

1. Two independent wallets pass every `wallet` vector in a released build.
2. Two independent verifier libraries, in different languages, pass every `parse` and `verify` vector.
3. At least one of those wallets and one of those libraries come from outside the authors.
4. The connector specification's Signing section references this MIP.
5. A production DApp on `midnight:mainnet` uses SIWM with a conforming wallet.

### Implementation Plan

1. **Verifier libraries** (TypeScript, then Rust). DApps can adopt as soon as these ship, and relay protection follows wallet adoption.
2. **Wallets**: detection, origin checks and approval screen. These break nothing, since payloads without the marker are unaffected.
3. **Connector specification**: a reference from its Signing section.
4. **Reference DApp and backend** implementing §3.

A non-normative reference implementation, used to generate the vectors, will be linked here when published.

## Backwards Compatibility Assessment

No hard fork and no ledger, node, indexer or proof-server change.

- **Older wallets** sign SIWM messages without the §4 checks. Their signatures still verify, but those users stay exposed to relays, and an RP cannot tell which kind of wallet signed. Protection grows with wallet adoption.
- **Existing free-form challenges** are unaffected unless they happen to contain the marker.
- **MIP-0031 conformance is a prerequisite.** Until wallets return MIP-0031-conformant signatures, §5 step 1 rejects them.
- **MIP-0033** is unaffected. Its messages are signed by grantee keys, and this parser rejects them.

## Security Considerations

**Closed:** with a conforming wallet, a relay fails at §4.3 step 2. A hostile site can only get signatures naming its own origin, and those fail §5 step 3 at the genuine RP.

**Not closed:** non-conforming wallets (no RP-side check can substitute for the wallet's); look-alike domains, where the wallet shows the true origin but cannot make users read it; script injection on the genuine origin; and compromised wallets or browsers.

**Replay** is prevented by single-use, session-bound nonces, by `Chain ID` with the address's network segment, and by the expiry cap. MIP-0031's prefix keeps sign-in messages distinct from transactions.

**Malleability:** MIP-0031 requires low-`s` ECDSA and canonical encodings, and the grammar gives each message exactly one byte form.

**Display:** the statement is site-controlled, so the dangerous code points are banned and the wallet shows the statement separately from the verified origin.

**Privacy:** signing in reveals one unshielded address, with its public history, and lets colluding sites link users. Wallets SHOULD offer per-site accounts and SHOULD say that the address will be shared.

## Implementation

The changes fall on wallets implementing `signData` (§4; browser extensions already receive the sender origin), new RP verifier libraries (SHA-256, BIP-340, secp256k1 ECDSA and Bech32m only), and a reference in the DApp Connector API specification. Dependencies are MIP-0031, MIP-0003 and MIP-0008. No on-chain state is used.

## Testing

`mip-xxxx/vectors.json` is normative. It uses MIP-0031's published test keys, and every signature is reproducible: Schnorr signatures fix the BIP-340 auxiliary randomness to `SHA-256(vector name)`, and ECDSA uses RFC 6979 with low `s`. The generator also reproduces MIP-0031's `schnorr-hex-decade` vector exactly.

| Set | Count | Covers |
|---|---|---|
| `parse` | 31 (9 accept, 22 reject) | every §2 rule, including CRLF, the userinfo trick in `URI`, a default port, a bidirectional override, a MIP-0033 `Grant ID` line and a near-miss marker |
| `wallet` | 17 | consent, refusal (relay, including through hex encoding; port, scheme or subdomain mismatch; framed and opaque callers; chain, account, expiry) and pass-through |
| `verify` | 13 (2 accept, 11 reject) | both schemes, key/address binding, replay, unknown nonce, expiry, foreign origin, chain, tampering, `Not Before`, near-miss marker |

Implementations SHOULD also test that the origin comes from the browser runtime rather than the page, and that nonce consumption is atomic under concurrent submissions.

## References

- [MIP-0031: Midnight Message Signing](./mip-0031-message-signing/mip-0031-message-signing.md)
- [MIP-0033: Scoped Grants and dApp Connection for Custody Accounts](./mip-0033-scoped-grants.md), §1 and §10
- [MIP-0003: ECDSA signature support](./mip-0003-ecdsa-support.md), address generation
- [MIP-0008: CAIP-2 Network Identifiers for Midnight](./mip-0008-caip-2-network-identifiers.md)
- [DApp Connector API specification](https://github.com/midnightntwrk/midnight-dapp-connector-api/blob/main/SPECIFICATION.md)
- [EIP-4361: Sign-In with Ethereum](https://eips.ethereum.org/EIPS/eip-4361); [CAIP-122: Sign in With X](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-122.md)
- RFC 6454, RFC 3986, RFC 3339, RFC 3492, BIP-340, BIP-350, RFC 6979

## Acknowledgements

The authors of MIP-0031 and MIP-0033, whose security analysis and sign-in layout this MIP builds on.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
