---
MIP: 0003
Title: ECDSA signature support
Authors:
  - Andrzej Kopeć (kapke)
Status: Draft
Category: Core
Created: 2026-05-14
Requires: N/A
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

## Abstract

To improve Midnight compatibility with enterprise key-management offerings, Midnight needs to
support ECDSA signatures for authorizing transactions. This MIP introduces ECDSA over secp256k1
(with SHA-256, as specified in [SEC1](https://www.secg.org/sec1-v2.pdf) and
[RFC 6979](https://datatracker.ietf.org/doc/html/rfc6979)) as an alternative signature scheme in
every context in which Midnight transactions carry signatures, while keeping Schnorr over secp256k1
([BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)) as the primary scheme.
The change is opt-in: existing wallets, contracts, and integrations that do not need ECDSA only need
to handle the new transaction format. To prevent accidental cross-scheme key reuse, ECDSA keys are
derived under a dedicated role and addresses are bound to the scheme through a domain separator.

## Motivation

Midnight uses Schnorr signatures over the secp256k1 curve as its default signature scheme. Schnorr
offers technical advantages over ECDSA (provable security under the discrete-log assumption, native
aggregation; see
[ADR-0017](https://github.com/midnightntwrk/midnight-architecture/blob/main/adrs/0017-signature-scheme.md)),
but it lags ECDSA in one important dimension: ecosystem adoption. A wide range of key-management and
signing solutions — including HSMs and MPC implementations — support ECDSA out of the box. For
custodians and other institutional users, the ability to reuse this existing infrastructure
materially lowers the barrier to adopting Midnight.

This MIP introduces ECDSA as an alternative signature scheme in every context in which Midnight
transactions carry signatures:

- unshielded token operations (UTXO spends, Dust registrations, reward claims)
- contract maintenance actions (deployment and maintenance updates)

## Specification

Supporting ECDSA alongside Schnorr requires changes in four areas:

- key management and derivation
- address generation
- transaction format and signature verification
- the DApp Connector API

ECDSA is instantiated over secp256k1 with SHA-256, deterministic nonces per RFC 6979, and the
standard fixed-width encodings: 33-byte compressed SEC1 verifying keys, 64-byte `(r ‖ s)` signatures
with 32-byte components, and 32-byte signing keys.

### Key management

A new role value (`4`) is introduced in the key derivation hierarchy. This allows a single wallet to
derive both Schnorr and ECDSA keys under the same account while keeping key material for the two
schemes strictly separated:

```
m / 44' / 2400' / account' / role / index
```

| Role name                             | Value | Description                                                                                        |
| ------------------------------------- | ----- | -------------------------------------------------------------------------------------------------- |
| Unshielded (and Night) External chain | 0     | Per BIP-44, the primary external chain. Schnorr-only. Used for unshielded tokens, including NIGHT. |
| Unshielded (and Night) Internal chain | 1     | Internal (change) chain, retained for BIP-44 compatibility. Schnorr-only.                          |
| Dust                                  | 2     | Keys for Dust, the token used to pay fees on Midnight.                                             |
| Shielded                              | 3     | Keys for shielded tokens, managed by the Zswap-based sub-protocol.                                 |
| Unshielded (and Night) ECDSA          | 4     | ECDSA-only role for unshielded tokens, including NIGHT.                                            |

The bytes derived at the leaf of the path are interpreted as a secp256k1 scalar (modulo the curve
order, rejecting zero) and used directly as the ECDSA signing key. The corresponding verifying key
is the compressed SEC1 encoding of a secp256k1 point.

### Address generation

The user-facing address format for unshielded tokens is unchanged: Bech32m encoding with the
`mn_addr` prefix. Address derivation for Schnorr keys is also unchanged: `SHA-256(verifying_key)`,
where `verifying_key` is the 32-byte BIP-340 x-only public key. ECDSA addresses are derived by
prefixing the compressed SEC1 verifying key with a domain separator:
`SHA-256("mn:ecdsa:" ‖ verifying_key)`, where `verifying_key` is the 33-byte compressed SEC1
encoding. This keeps the address format identical at the wire level (32 bytes) but binds each
address to a specific authorization scheme.

### Transaction format and signature verification

Signatures and verifying keys appear in the following transaction structures: claim-rewards
transactions, unshielded offers, UTXO spends, UTXO outputs (via the recorded owner address),
contract maintenance authorities (set on contract deploy and updatable via maintenance updates), and
contract maintenance updates:

```rust
struct ClaimRewardsTransaction {
    pub owner: SignatureVerifyingKey,
    pub signature: Signature,
    //...
}

struct UnshieldedOffer {
    pub signatures: storage::storage::Array<Signature>,
    //...
}

struct UtxoOutput {
    pub owner: UserAddress,
    //...
}

struct UtxoSpend {
    pub owner: SignatureVerifyingKey,
    //...
}

struct ContractMaintenanceAuthority {
    pub committee: Vec<SignatureVerifyingKey>,
    //...
}

struct MaintenanceUpdate {
    pub signatures: storage::storage::Array<Signature>,
    //...
}
```

The following structural changes are required:

- `Signature`, `SignatureVerifyingKey`, and `SigningKey` become tagged enums over the supported
  schemes.
- The transaction serialization tag is adjusted accordingly. Subsidiary types whose representations
  change are versioned accordingly.
- The ECDSA signature is encoded the same way as Schnorr: concatenation of the big-endian byte
  encodings of scalars `r` and `s`, each padded to 32 bytes, yielding a 64-byte signature.

```rust
enum Signature {
  Schnorr(base_crypto::schnorr::Signature),
  ECDSA(base_crypto::ecdsa::Signature),
}

enum SignatureVerifyingKey {
  Schnorr(base_crypto::schnorr::VerifyingKey),
  ECDSA(base_crypto::ecdsa::VerifyingKey),
}

enum SigningKey {
  Schnorr(base_crypto::schnorr::SigningKey),
  ECDSA(base_crypto::ecdsa::SigningKey),
}
```

The ledger validation rules are extended as follows:

- For `ClaimRewardsTransaction`, `UnshieldedOffer`, and `MaintenanceUpdate`, each signature must use
  the same scheme as the verifying key it is checked against; cross-scheme combinations are
  rejected.
- For `UtxoSpend`, an address is derived from the provided verifying key using the
  address-derivation rule corresponding to that key's scheme (see
  [Address generation](#address-generation)) and compared for equality against the recorded
  `UserAddress`.
- To prevent ECDSA signature malleability, the value `s` must be less than the curve order `n`
  divided by two (as in
  [BIP-0062](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki#low-s-values-in-signatures)).

### DApp Connector API

The DApp Connector API requires a single change: the return type of `signData` is extended with a
`scheme` discriminator identifying the signature scheme used. The field is initially optional, with
the default value `schnorr_bip340`; a future major version bump of the API will make it mandatory.

```typescript
export type Signature = {
  scheme?: "ecdsa_secp256k1_sha256" | "schnorr_bip340";
  data: string;
  signature: string; // 64 bytes for either scheme
  verifyingKey: string; // 32 bytes for Schnorr (BIP-340 x-only), 33 bytes for ECDSA (SEC1 compressed)
};
```

This requires DApps relying on the `signData` API to be aware of both signature schemes and to
handle the `scheme` discriminator.

## Rationale

**Strict key separation between schemes.** A single secret key must never be used to produce both
Schnorr and ECDSA signatures. Nonce reuse — across schemes, or across distinct signatures within
either scheme — can fully expose the signing key. Allocating a dedicated role index for ECDSA
enforces this separation at the derivation level, so that wallets cannot accidentally share key
material between the two schemes. Determinism of ECDSA signing under RFC 6979 removes within-scheme
nonce-reuse risk for correctly implemented signers, but does not protect against cross-scheme reuse
— only key separation does.

**Compatibility tradeoff.** The chosen derivation path differs from the defaults used by software
designed primarily for Ethereum or Bitcoin, which typically derives ECDSA keys at
`m/44'/2400'/account'/0/0` — a path reserved for Schnorr in Midnight. Two alternatives were
considered and rejected:

- _Introduce a Midnight-specific `purpose` level._ This would more cleanly signal scheme separation
  at the top of the derivation hierarchy. However, the underlying requirement — supporting multiple
  cryptographic schemes under a single account — is itself incompatible with wallet software that
  assumes a single scheme per BIP-44 account. Adding a `purpose` level would not resolve this and
  would only introduce an additional point of divergence from BIP-44.
- _Allow wallets to select the scheme for keys derived at existing paths._ If two wallets restored
  from the same seed disagreed on the scheme, both Schnorr and ECDSA signatures could be produced
  from the same key, compromising the secret key and any funds it controls.

**Address-level binding.** The domain separator in the ECDSA address-derivation rule
(`SHA-256("mn:ecdsa:" ‖ verifying_key)`) ensures that even if a signing party were to accidentally
produce both Schnorr and ECDSA signatures over an identical secret scalar, the two would resolve to
different on-chain addresses. This provides a second line of defense beyond key-derivation
separation.

**Opt-in adoption.** This MIP enables ECDSA support while keeping Schnorr the primary signature
scheme; it does not mandate ECDSA. The minimal surface change in the DApp Connector reflects this
design intent: existing applications and wallets that do not need ECDSA require only changes related
to the transaction format, and integrators with existing ECDSA infrastructure can adopt it
incrementally.

## Path to Active

### Acceptance Criteria

The MIP transitions to Active once the following are met:

1. **Ledger.** ECDSA signature support is merged into a released `midnight-ledger` version and
   deployed in the corresponding `midnight-node` release; the new transaction format is the
   canonical wire format for that release.
2. **Wallet SDK.** A released version of the wallet SDK supports the new role-`4` derivation path,
   ECDSA address derivation with the `"mn:ecdsa:"` domain separator, and signing for both schemes.
3. **DApp Connector.** A released version of `@midnight-ntwrk/dapp-connector-api` includes the
   optional `scheme` discriminator on `signData` responses.
4. **Test vectors.** Test vectors covering ECDSA key derivation and address derivation are published
   in `midnight-architecture` and consumed by Wallet SDK.
5. **Network deployment.** A Midnight network upgrade activating the new transaction format has been
   completed, and the upgraded ledger version validates ECDSA-authorized transactions on the live
   network.

### Implementation Plan

The work is sequenced in three stages so that dependent components can land independently:

1. **Ledger** Land [midnight-ledger#498](https://github.com/midnightntwrk/midnight-ledger/pull/498):
   enum types, validation rules, address derivation, integration tests, and the transaction tag
   update. Bump the WASM bindings accordingly.
2. **Wallet SDK and DApp Connector.** Extend the wallet SDK to derive at role `4`, sign and verify
   ECDSA, and produce ECDSA `UserAddress`es. Extend the DApp Connector API to surface the `scheme`
   discriminator.
3. **Integration and deployment.** Implement compatibility with Ledger 9 across the node, indexer,
   Midnight.js, and other client applications.

## Backwards Compatibility Assessment

This change requires a hard fork: the transaction serialization tag is updated (with several
subsidiary tags bumped; see [Specification](#specification)), and the ledger validation rules are
extended.

Application-level impact is expected to be limited:

- The DApp Connector change is additive — existing callers continue to receive responses that omit
  the `scheme` field and treat it as the default `schnorr_bip340`.
- Most applications only need to follow the transaction-format transition.
- Signing-related APIs in the wallet SDK and ledger require updates to accept the ECDSA scheme.
- Indexers and explorers need only consume the new serialized format; no semantic change is required
  for display of addresses.

There is no need for migration of existing on-chain addresses or balances: pre-fork UTXOs remain
bound to their Schnorr keys, and ECDSA addresses are simply additional addresses that may now be
funded.

## Security Considerations

**Cross-scheme nonce reuse.** ECDSA and Schnorr both derive nonces from the signing key and message;
producing both signature types under a single secret scalar can leak the secret. This is mitigated
by:

1. Separate BIP-44 roles for Schnorr (`0`/`1`) and ECDSA (`4`), so derivation never yields the same
   scalar under different schemes.
2. Distinct address derivation (`SHA-256("mn:ecdsa:" ‖ vk)` vs. `SHA-256(vk)`), so even an
   accidental cross-scheme signing would address different UTXOs.

**Within-scheme nonce reuse.** ECDSA signing uses deterministic nonces per RFC 6979, eliminating
reuse risk in correctly implemented signers. Implementations sourcing nonces from external HSMs or
MPC stacks should ensure their nonce-generation is either RFC 6979-compliant or backed by a strong
RNG and per-message uniqueness guarantee.

**Signature malleability.** ECDSA over secp256k1 admits both `(r, s)` and `(r, -s mod n)` as valid
signatures for a given message and key. It is addressed by enforcing the `s` component to always be
in the lower range.

**Encoding rigidity.** Verifying keys and signatures use fixed-width SEC1 / `(r ‖ s)` encodings
rather than DER, which closes off a class of parser-ambiguity malleability that has historically
affected DER-based ECDSA implementations (e.g. BIP-66).

**Cross-scheme verification rejection.** The ledger explicitly rejects any verifying-key / signature
pair whose variants do not match (e.g. an ECDSA signature presented against a Schnorr key, or vice
versa), preventing downgrade-style attacks via type confusion.

## Implementation

Specific changes are needed in the following areas:

- Ledger: enum types, validation rules, transaction-format version bump, specification update
- Wallet SDK: role-`4` key derivation, ECDSA address derivation, signing APIs and helpers,
  specification update.
- DApp Connector API: extension of the `Signature` response type with the optional `scheme`
  discriminator; specification update.
- Wallet implementations: surfacing the `scheme` field; key-management for ECDSA accounts
  (optional).

Downstream components — node, indexer, explorer, Midnight.js, example DApps — do not require
ECDSA-specific changes, but must pick up the new transaction format.

## Testing

All existing tests should pass with little or no modification. New tests are required to cover the
flows involving signatures and their verification:

- ECDSA key derivation from a seed (against test vectors)
- ECDSA address derivation (against test vectors)
- Dust registration
- Unshielded token transfers (UTXO spend authorization)
- Contract maintenance actions (deployment with ECDSA committee members, ECDSA-signed updates,
  mixed-scheme committees)
- Reward / bridge pool claims
- Signing data via DApp Connector API, including the `scheme` discriminator

Particular attention should be paid to mismatch cases that must be rejected:

- UTXO spend from an ECDSA-derived address authorized by a Schnorr key and Schnorr signature
- UTXO spend from a Schnorr-derived address authorized by an ECDSA key and ECDSA signature
- Claim-rewards transaction whose `owner` is ECDSA but whose `signature` is Schnorr (and vice versa)
- Maintenance update where a committee member's verifying-key variant disagrees with the supplied
  signature variant

Cross-implementation conformance — Rust and TypeScript stacks producing identical signatures over
the same test vectors — should be exercised via the integration test suite.

## References

- [BIP-340 — Schnorr Signatures for secp256k1](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
- [SEC 1 v2 — Elliptic Curve Cryptography](https://www.secg.org/sec1-v2.pdf)
- [RFC 6979 — Deterministic Usage of DSA and ECDSA](https://datatracker.ietf.org/doc/html/rfc6979)
- [ADR-0017 — Signature scheme](https://github.com/midnightntwrk/midnight-architecture/blob/main/adrs/0017-signature-scheme.md)

## Acknowledgements

Iñigo Querejeta Azurmendi (@iquerejeta) for reviews and highlighting possible problems

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement
[Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
