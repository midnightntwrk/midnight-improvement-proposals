---
MIP: "xxxx" # to be assigned
Title: ML-DSA-65 signature support
Authors:
  - Quantova Inc (quantova-inc)
Status: Draft
Category: Core
Created: 2026-11-19
Requires: none
Replaces: none
MPS: MPS-0051
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

[MPS-0051](/mps/mps-0051-post-quantum-spending-authorization.md) records that every signature
the ledger accepts today, Schnorr over secp256k1 (BIP-340) and ECDSA over secp256k1
([MIP-0003](/mips/mip-0003-ecdsa-support.md)), can be forged by a cryptographically relevant
quantum computer once the verifying key is known. This MIP adds ML-DSA-65
([FIPS 204](https://csrc.nist.gov/pubs/fips/204/final)) as a third signature scheme in every
context where Midnight transactions carry signatures, using the same enum extension that MIP-0003
used for ECDSA. Addresses bind to the scheme through a domain separator that follows
[MIP-0021](/mips/mip-0021-domain-separation.md), every ledger signature carries a FIPS 204 context
string, and ML-DSA keys are derived from the wallet seed outside the secp256k1 BIP-32 tree so that
an exposed extended public key cannot lead to them. The change is opt in. Existing Schnorr and
ECDSA outputs stay valid, and a holder moves to post quantum ownership with an ordinary transfer
signed by the current key while that key is still sound. No freeze, sweep or new transaction kind
is involved.

## Motivation

An unshielded output is owned by `SHA-256` of a verifying key. The key stays hidden until the first
spend, then appears in the `UtxoSpend` and stays public for every other output held at the same
address. Reward claims, Dust registrations and contract maintenance committees put the verifying key
on chain directly. MPS-0051 sets out these exposure classes and
the measured cost of the post quantum candidates.

Moving funds to a post quantum key takes time. Wallets, hardware signers and custodians need a
scheme to target, and holders need a window in which their current key is still sound so that they
can authorize the move themselves. That window exists only if the post quantum scheme is in the
ledger before it is needed.

MIP-0003 already turned `Signature`, `SignatureVerifyingKey` and `SigningKey` into tagged enums and
added scheme specific address derivation. This MIP adds one variant to each and reuses the rest.

## Specification

ML-DSA-65 is used in its pure form (FIPS 204 Algorithms 2 and 3, not HashML-DSA) with these fixed
sizes.

| Item          | Size        | Encoding                                  |
| ------------- | ----------- | ----------------------------------------- |
| Signing seed  | 32 bytes    | the seed `ξ` of FIPS 204 Algorithm 6      |
| Verifying key | 1,952 bytes | `pkEncode` (FIPS 204 Algorithm 22)        |
| Signature     | 3,309 bytes | `sigEncode` (FIPS 204 Algorithm 26)       |

### Key derivation

ML-DSA signing seeds are not derived through the BIP-32 tree that holds Schnorr and ECDSA keys.
For account `a` and index `i`, both unsigned 32 bit integers, the wallet computes

```
ξ = HMAC-SHA512(key = "midnight:mldsa65-seed[v1]", data = seed ‖ ser32(a) ‖ ser32(i))[0..32]
```

where `seed` is the same wallet seed passed to `HDWallet.fromSeed` and `ser32` is the 4 byte big
endian encoding. The key pair is `ML-DSA.KeyGen_internal(ξ)` (FIPS 204 Algorithm 6). The wallet
stores only `ξ` and expands it when it signs. The separator follows MIP-0021 and is used as the HMAC
key bytes.

No new role is added to `m / 44' / 2400' / account' / role / index`. The reason is in
[Rationale](#rationale).

### Address generation

The address format is unchanged, a 32 byte `UserAddress` shown in Bech32m with the `mn_addr`
prefix. For an ML-DSA-65 verifying key `vk` of 1,952 bytes the address is

```
SHA-256("midnight:mldsa65-addr[v1]" ‖ vk)
```

Schnorr addresses keep `SHA-256(vk)` and ECDSA addresses keep `SHA-256("midnight:ecdsa:" ‖ vk)`, so
the three schemes can never resolve to the same address.

### Transaction format and signature verification

The structures listed in MIP-0003 carry the new variants without other changes. These are
`ClaimRewardsTransaction`, `UnshieldedOffer`, `UtxoSpend`, `UtxoOutput` through the recorded owner
address, `DustRegistration`, `ContractMaintenanceAuthority` and `MaintenanceUpdate`.

```rust
enum Signature {
    Schnorr(base_crypto::schnorr::Signature),
    ECDSA(base_crypto::ecdsa::Signature),
    MlDsa65(base_crypto::mldsa65::Signature),
}

enum SignatureVerifyingKey {
    Schnorr(base_crypto::schnorr::VerifyingKey),
    ECDSA(base_crypto::ecdsa::VerifyingKey),
    MlDsa65(base_crypto::mldsa65::VerifyingKey),
}

enum SigningKey {
    Schnorr(base_crypto::schnorr::SigningKey),
    ECDSA(base_crypto::ecdsa::SigningKey),
    MlDsa65(base_crypto::mldsa65::SigningKey),
}

enum ContractMaintenanceVerifyingKey {
    Schnorr(base_crypto::schnorr::VerifyingKey),
    ECDSA(base_crypto::ecdsa::VerifyingKey),
    MlDsa65(base_crypto::mldsa65::VerifyingKey),
}
```

The serialization tags of these types and of the structures that contain them are bumped, as
MIP-0003 did for ledger 9.

The validation rules are extended as follows.

1. A signature must use the same scheme as the verifying key it is checked against. Any other pairing
   is rejected, as MIP-0003 already requires for Schnorr and ECDSA.
2. Deserialization rejects an ML-DSA-65 verifying key that is not exactly 1,952 bytes and a
   signature that is not exactly 3,309 bytes.
3. An ML-DSA-65 signature is accepted only if `ML-DSA.Verify(vk, M, σ, ctx)` (FIPS 204 Algorithm 3)
   returns true, where `M` is the existing `data_to_sign` output for that structure and `ctx` is the
   23 byte string `midnight:mldsa65-tx[v1]`. A signature whose hint encoding fails `HintBitUnpack` is
   rejected by that algorithm and is not repaired.
4. For `UtxoSpend`, the address is derived from the presented key with the rule for its scheme and
   compared with the recorded `UserAddress`, as for ECDSA.

Signers should use the hedged signing mode of FIPS 204. Verification does not depend on the mode.

### Cost model

Today every signature check is charged the single `signature_verify_constant` of the runtime cost
model, whatever the scheme. ML-DSA-65 gets its own constant, set by the cost model generator from
measured decode and verify time. On the transaction shape of MPS-0051, decode and verify of one
ML-DSA-65 input takes 58.6 µs against 33.4 µs for BIP-340, and the key and signature add 5,165 bytes
per spent input. Those bytes count under `blockUsage` as they do today. One input's key and signature
take about 0.5 percent of the 1 MiB `transaction_byte_limit`.

### DApp Connector API

The `scheme` discriminator that MIP-0003 added to the `signData` result gains one value.

```typescript
export type Signature = {
  scheme?: "ecdsa_secp256k1_sha256" | "schnorr_bip340" | "ml_dsa_65";
  data: string;
  signature: string; // 64 bytes for Schnorr and ECDSA, 3,309 bytes for ML-DSA-65
  verifyingKey: string; // 32, 33 or 1,952 bytes depending on the scheme
};
```

For `ml_dsa_65` the wallet signs the full message `M` that
[MIP-0031](/mips/mip-0031-message-signing/mip-0031-message-signing.md) builds, prefix included, with
pure ML-DSA and the context string `midnight:mldsa65-msg[v1]`. It does not sign `SHA-256(M)` as the
two classical schemes do. ML-DSA already hashes the message with SHAKE256, and signing a SHA-256
digest would cap a signature over a message the DApp chose at SHA-256 collision resistance, which is
NIST category 2. The context string means a message signature can never verify as a ledger
signature and the reverse. Encoded as MIP-0031 requires, `signature` is 6,618 hexadecimal characters
and `verifyingKey` is 3,904.

### Moving to post quantum ownership

No new transaction kind is needed.

- An unshielded holder spends existing outputs to an ML-DSA-65 address with an ordinary transfer
  signed by the current Schnorr or ECDSA key.
- A contract replaces its maintenance committee members with ML-DSA-65 keys through an ordinary
  `MaintenanceUpdate`. Mixed committees are valid during the change.
- Rewards are credited to a `UserAddress`. A recipient whose rewards are credited to an ML-DSA-65
  address claims them with the matching ML-DSA-65 `owner` key.
- A Dust registration is made again under an ML-DSA-65 night key under the existing registration
  rules.

Wallets that support this MIP should send change to an ML-DSA-65 address and offer the holder a
single action that moves the remaining classical outputs.

## Rationale

**ML-DSA-65.** Unshielded ownership can last for years, so the scheme should keep a margin against
future lattice cryptanalysis. ML-DSA-65 is NIST security category 3. ML-DSA-44 is category 2 and
saves 1,529 bytes per input. ML-DSA-87 is category 5 and costs 1,958 bytes more. Decode and verify
for ML-DSA-65 is 1.75 times BIP-340 per input, which the cost model can carry. SLH-DSA verifies 15
to 54 times slower than BIP-340 with signatures of 7.9 KB to 17.1 KB, so it is a better fit for low
frequency authority such as governance and recovery, which is midnight-node#2263 and outside this
MIP. Because the scheme is a tagged variant, a later MIP can add ML-DSA-44 or ML-DSA-87 the same way.

**Seeds outside the BIP-32 tree.** BIP-32 lets anyone holding an extended public key, the pair of a
public key and chain code, derive the public keys below it. A quantum adversary who sees one can
also recover its private key and from there every child key, hardened or not, because hardened
derivation only hides children from parties that lack the parent private key. Wallets share
account level extended public keys for watch only use. If ML-DSA seeds sat under such an account,
one exposed extended public key would give an adversary every post quantum key under it. Deriving
`ξ` with HMAC from the wallet seed keeps the backup a single seed while removing every elliptic curve
step from the path.

**Context strings.** The ledger already separates signed structures through their serialization
tags. The FIPS 204 context adds a second boundary at the primitive itself, between ledger
signatures, `signData` signatures and any other protocol where the same seed might be used. It
costs nothing in size because the context is not transmitted.

**No hybrid scheme.** A hybrid of BIP-340 and ML-DSA-65 would add a second key and signature to
every input and a second verification. Its benefit is protection against a flaw in ML-DSA or its
implementation before a quantum computer exists, while the classical half gives nothing once one
does. The tagged enum leaves room for a hybrid variant if the network asks for one.

**No cached expanded keys in consensus.** A node may cache the expanded matrix of an ML-DSA-65 key
under its address for repeat spends. That is a local optimization with no effect on validity, so
this MIP does not specify it.

**Dormant exposed outputs.** Outputs whose keys are already public and whose holders never move
are left as they are. A freeze, sweep or deadline would remove funds from holders who did nothing
wrong, and MPS-0051 rules it out.

## Path to Active

### Acceptance Criteria

1. **Ledger.** ML-DSA-65 support is merged into a released `midnight-ledger` version and deployed in
   the matching `midnight-node` release.
2. **Wallet SDK.** A released wallet SDK derives ML-DSA-65 seeds and addresses as specified, signs
   with the ledger context string and sends change to ML-DSA-65 addresses when enabled.
3. **DApp Connector.** A released `@midnight-ntwrk/dapp-connector-api` accepts `ml_dsa_65` in the
   `scheme` discriminator.
4. **Test vectors.** Vectors for seed derivation, address derivation and signing are published and
   consumed by the Rust and TypeScript stacks, and both pass the NIST ACVP ML-DSA-65 keyGen, sigGen
   and sigVer vectors.
5. **Network deployment.** A network upgrade activates the new format and the live network validates
   ML-DSA-65 authorized transactions.

### Implementation Plan

1. **Ledger.** Add `base_crypto::mldsa65`, the enum variants, the validation rules, the address rule,
   the cost model entry and the tag bumps, with integration tests and WASM bindings.
2. **Wallet SDK and DApp Connector.** Add seed derivation, signing, address derivation and the
   `scheme` value.
3. **Integration and deployment.** Pick up the new format in the node, indexer, Midnight.js and
   other clients, then activate through a network upgrade.

## Backwards Compatibility Assessment

This is a hard fork. Serialization tags change and the validation rules are extended, as with
MIP-0003. Existing Schnorr and ECDSA keys, addresses and outputs keep working unchanged. Clients
that do not use ML-DSA-65 only need to read the new format. Indexers and explorers see larger
verifying keys and signatures in spends and should not assume 32 or 33 byte keys or 64 byte
signatures. Hardware wallets that cannot hold an ML-DSA-65 key keep signing with Schnorr or ECDSA.

## Security Considerations

**Security level.** ML-DSA-65 targets NIST category 3 against classical and quantum adversaries.
Ownership of an ML-DSA-65 address does not depend on any elliptic curve assumption, before or after
the first spend.

**Cross scheme confusion.** Variant mismatches are rejected, and the address separators give each
scheme its own address space, so a key of one scheme can never authorize an output of another.

**Cross context reuse.** Ledger signatures and `signData` signatures use different FIPS 204
context strings, so neither can be replayed as the other even when the signed bytes match.

**Malleability.** ML-DSA is designed to be strongly unforgeable. Fixed lengths and strict decoding
leave no second encoding of a valid signature or key.

**Randomness.** Key generation is deterministic from `ξ`, so seed quality is the only requirement.
Hedged signing adds fresh randomness but does not depend on it for security, so a weak signing RNG
does not leak the key.

**Implementation.** The ledger compiles to WASM, so the Rust implementation must be pure Rust, for
example RustCrypto `ml-dsa` or `libcrux-ml-dsa`. Whichever is chosen must pass the ACVP vectors in
CI and agree byte for byte with the TypeScript implementation used by the wallet.

**Exposure before migration.** This MIP does not protect a Schnorr or ECDSA key that is already
public. It gives holders the means to move before a quantum adversary exists, which is the only
point at which their current signature still proves they are the owner.

## Implementation

- Ledger. Enum variants, validation rules, address rule, cost model, tag bumps, specification update.
- Wallet SDK. Seed derivation, signing with the ledger context, address derivation, change handling.
- DApp Connector API. The `ml_dsa_65` scheme value and the documented sizes.
- Node, indexer, explorer, Midnight.js. Pick up the new transaction format. No ML-DSA specific logic
  beyond decoding.

## Testing

All existing tests should pass. New tests cover

- seed derivation from fixed wallet seeds, account and index values against published vectors
- address derivation against published vectors
- ACVP ML-DSA-65 keyGen, sigGen and sigVer vectors in the Rust and TypeScript stacks
- unshielded spends, reward claims, Dust registrations and maintenance updates authorized with
  ML-DSA-65, including mixed committees

and these cases, which must be rejected

- an ML-DSA-65 signature checked against a Schnorr or ECDSA key, and the reverse
- an ML-DSA-65 key presented for an output whose address was derived for another scheme
- a verifying key or signature one byte short or one byte long
- a valid ML-DSA-65 signature made with any context other than `midnight:mldsa65-tx[v1]`, including
  `midnight:mldsa65-msg[v1]` and the empty context
- a signature with a malformed hint encoding

## References

- [MPS-0051, Post Quantum Spending Authorization for Unshielded Outputs](/mps/mps-0051-post-quantum-spending-authorization.md)
- [MIP-0003, ECDSA signature support](/mips/mip-0003-ecdsa-support.md)
- [MIP-0021, Domain-Separation Convention](/mips/mip-0021-domain-separation.md)
- [MIP-0031, Midnight Message Signing](/mips/mip-0031-message-signing/mip-0031-message-signing.md)
- [FIPS 204, Module-Lattice-Based Digital Signature Standard](https://csrc.nist.gov/pubs/fips/204/final)
- [NIST ACVP ML-DSA test vectors](https://github.com/usnistgov/ACVP-Server/tree/master/gen-val/json-files)
- [BIP-32, Hierarchical Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
- [NIST IR 8547, Transition to Post-Quantum Cryptography Standards](https://csrc.nist.gov/pubs/ir/8547/ipd)
- [Size and cost measurements on midnight-ledger#798](https://github.com/midnightntwrk/midnight-ledger/issues/798)
  and the harness at [quantova-inc/midnight-pq-signature-costs](https://github.com/quantova-inc/midnight-pq-signature-costs)

## Acknowledgements

None yet.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement
[Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
