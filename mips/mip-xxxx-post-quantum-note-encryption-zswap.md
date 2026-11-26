---
MIP: xxxx
Title: Post-Quantum Note Encryption for Zswap
Authors:
  - David Nevado (davidnevadoc)
Status: Draft
Category: Core
Created: 2026-11-26
Requires: none
Replaces: none
MPS: MPS-0050
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

Zswap delivers each shielded coin by encrypting its contents to the recipient.
The key that protects that delivery is established with a Diffie-Hellman
exchange over the Jubjub curve, which a cryptographically relevant quantum
computer (CRQC) breaks. Because every note ciphertext is recorded on chain, the
exposure is harvest-now-decrypt-later: ciphertexts collected today can be
decrypted once a CRQC exists.

This MIP adds a post-quantum key encapsulation mechanism (ML-KEM) to the
key-establishment step, combined with the existing Jubjub exchange so the note
stays protected while either primitive holds. The Poseidon symmetric layer that
encrypts the note contents is kept. An algorithm version tag frees the
ciphertext and address formats from a single primitive and lets both formats
coexist during a transition window. The change is confined to note delivery.
Coin ownership, value commitments, balancing and proof soundness are untouched,
and the hash-based ownership layer that keeps spend-linkage private is already
post-quantum.

The migration protects only outputs sent in the new format. Ciphertexts already
on chain stay exposed, since no upgrade can re-encrypt them. Its main cost is
size: a lattice ciphertext and public key are far larger than a curve point,
which grows transactions and addresses but, because encryption sits outside the
proof, leaves the proving circuit unchanged.

## Motivation

Every shielded output on Midnight carries a note ciphertext that tells the
recipient the coin's value, token type and nonce. The confidentiality of that
ciphertext rests on a Jubjub Diffie-Hellman exchange between the sender and the
recipient's published encryption key. Shor's algorithm recovers the recipient's
secret from that public key, after which every note ever sent to the recipient
decrypts. The full threat model, including what does and does not leak, is set
out in the originating MPS,
[MPS-0050](/mps/mps-0050-post-quantum-confidentiality-zswap.md)
(Post-Quantum Confidentiality for Zswap).

Note delivery is the only confidentiality exposure whose cost accrues while we
wait. A ciphertext recorded today can be harvested and stored now and decrypted
the day a CRQC exists, so the secrecy of past transactions depends on migrating
before that day, not on reacting to it.

The current scheme offers no way to migrate in place. The delivery primitive is
hardwired to Jubjub. The ledger versions its serialized types as a whole, the
ciphertext included, but a ciphertext carries no identifier of its own and an
output accepts a single ciphertext type. The address is the two keys
concatenated, with no algorithm identifier. A post-quantum primitive cannot be
introduced without changing those formats, and there is no hook that lets a
second primitive coexist with the first during a transition.

## Specification

Today a coin is delivered as follows. The sender draws an ephemeral Jubjub
scalar, runs a Diffie-Hellman exchange against the recipient's Jubjub encryption
key and derives a symmetric key from the shared secret with Poseidon. The note
(nonce, token type and value) is five field elements. It is encrypted with
Poseidon in counter mode together with a leading zero-valued check element, six
field elements in total. The ciphertext, an ephemeral Jubjub point plus those
six field elements, is published in the output. Encryption happens outside the
zero-knowledge circuit: the output proof binds only a hash of the ciphertext as
a public input, and the recipient confirms a decrypted note is genuine by
recomputing the coin commitment and matching it against the one on chain. The
circuit never encrypts and never checks that the ciphertext decrypts correctly.

**Key establishment gains a KEM.** Add a key encapsulation mechanism to the
Jubjub Diffie-Hellman exchange. To deliver a coin the sender encapsulates to the
recipient's KEM public key, producing an encapsulation and a shared secret; the
recipient decapsulates the encapsulation with their secret key to recover the
same shared secret. The proposed mechanism is ML-KEM (FIPS 203) at level
ML-KEM-768.

**Hybrid key establishment.** The sender runs both the existing Jubjub exchange
and the ML-KEM encapsulation, and the symmetric key is derived from both shared
secrets together with the ephemeral Jubjub point. Confidentiality holds as long
as either primitive is unbroken, which hedges against a future weakness in
ML-KEM while it is still comparatively young. This is current practice for
harvest-now-decrypt-later confidentiality. Against pure ML-KEM, the cost is the
32-byte Jubjub point per ciphertext, the Jubjub key per address and one scalar
multiplication.

Pure ML-KEM, where the shared secret comes from ML-KEM alone, is the
alternative. It is simpler and smaller, but confidentiality then rests entirely
on ML-KEM. It remains an option if review finds the hedge not worth its cost.

**The symmetric layer is preserved.** The note stays encrypted as six field
elements under Poseidon in counter mode. Only the source of the symmetric key
changes: it is derived from both shared secrets through a key-derivation step
rather than from the Jubjub shared point alone. Keeping the payload field-native
leaves the note representation and the coin commitment unchanged, and keeps the
proof binding a single natively computed hash.

**No authentication tag.** As today, the payload carries no authentication
tag. The zero-valued check element detects a wrong key but does not protect
against modification. Integrity comes from the rest of the system: the output
proof binds the ciphertext, so it cannot be altered once proven, and the
recipient accepts a note only if it reproduces the coin commitment on chain.

**Wire format and version tag.** The output ciphertext gains the KEM
encapsulation alongside the ephemeral Jubjub point and the encrypted note, and
both the ciphertext and the address gain an explicit algorithm identifier.
The current format carries no identifier, so the tag is added rather than
repurposed. It lets legacy and hybrid ciphertexts coexist on chain and be told
apart, and lets a
later primitive be added the same way. An ML-KEM-768 encapsulation is 1088 bytes
and its public key is 1184 bytes, against a 32-byte Jubjub point today.
ML-KEM-1024 would raise both to 1568 bytes for a higher security level, NIST
category 5 (comparable to AES-256) against category 3 (comparable to AES-192)
for ML-KEM-768. The shared secret is 32 bytes at either level. ML-KEM-768 is
proposed.

**Proof binding.** The output proof must keep binding the published ciphertext
so it cannot be swapped after proving. Today this is a single public-input field
element, the ciphertext hash, computed natively by both the prover and the
verifier; the circuit itself does not hash or inspect the ciphertext. Preserving
that arrangement means the larger ciphertext, encapsulation included, is hashed
natively to one field element and the circuit is unchanged. The KEM ciphertext
is not verified inside the circuit.

**Address and key derivation.** A user's address bundles two keys, a hash-based
coin public key and an encryption public key. The address gains a third key, an
ML-KEM public key derived from the wallet seed under its own domain. The Jubjub
encryption key stays, derived as today, and serves the classical half of the
hybrid exchange. The coin public key is hash-based, already post-quantum and
left as is. The address grows by the size of the ML-KEM public key.

**Migration through a dual-support window.** The upgrade opens a window during
which the ledger accepts both legacy Jubjub ciphertexts and the new hybrid
ciphertexts, told apart by the version tag. Wallets scan for both. A sender
uses the new format when the recipient's address carries an ML-KEM key, and the
legacy format otherwise. Once the ecosystem has migrated, a later upgrade stops
accepting legacy ciphertexts in new outputs. Outputs already on chain stay
valid and decryptable in either case.

**Note (migration):** the length of the window, how wallets, indexers and
other services handle the coexistence and the conditions for closing the
window need input from engineers familiar with the rest of the Midnight stack.

**Wallet scan.** A wallet still trial-decrypts every candidate output: it runs
the Jubjub exchange, decapsulates the encapsulation, derives the symmetric key,
decrypts the note and recomputes the coin
commitment to decide whether the coin is its own. A KEM decapsulation never
fails outright, so ownership is decided by the commitment match exactly as
today.

**Domain separation.** The change adds three hash use sites: the derivation of
the ML-KEM key pair from the wallet seed, the derivation of the symmetric key
from the two shared secrets and the native hash that binds the new ciphertext to
the output proof. Each takes its own domain separator in the form set by
MIP-0021: the `midnight:` namespace, segments naming the construction and a
version suffix. The proposed separators are `midnight:zswap-esk-mlkem[v1]` for
the key pair, `midnight:zswap-note-key[v1]` for the symmetric key and
`midnight:zswap-ciphertext[v2]` for the binding hash. The legacy derivation and
binding keep their current separators.

## Rationale

The KEM-DEM shape matches the existing structure. Zswap already delivers notes
as key establishment followed by symmetric encryption, so adding a KEM to the
key-establishment half is the smallest change that closes the exposure.

Preserving the Poseidon payload keeps the note field-native, which keeps the
coin commitment and the six-element payload unchanged, keeps the proof binding a
single native hash and avoids touching the output circuit.

Binding a native hash rather than proving encryption in-circuit follows what
Zswap already does. Non-malleability needs only a commitment to the ciphertext,
not a proof that it decrypts correctly, and lattice verification inside the
proof system would be prohibitively expensive.

Hybrid key establishment is proposed over pure ML-KEM because the exposure is
harvest-now-decrypt-later: a ciphertext published today must stay confidential
for decades, and a weakness later found in ML-KEM could not be repaired for
notes already on chain. The hedge also covers implementation flaws in the
ML-KEM dependency, not only a break of the underlying problem. Keeping the
Jubjub exchange costs 32 bytes per ciphertext and one scalar multiplication,
small next to the ML-KEM encapsulation itself.

A version tag is the minimum agility needed to introduce a second primitive and
to let it coexist with the first. A dual-support window, rather than a single
cutover, lets wallets and services migrate without a flag day, at the cost of
keeping both paths for a while.

The scope is delivery only because Zswap's guarantees rest on separate
assumptions with separate quantum fates. Confidentiality of delivery falls to a
CRQC, coin ownership does not and integrity is out of scope here.

## Path to Active

### Acceptance Criteria

- The KEM primitive, the versioned ciphertext and address formats and the
  native proof binding are implemented in the ledger and merged.
- The wallet SDK and WASM bindings generate and decode the new keys and
  ciphertexts and scan for both formats.
- Test vectors cover seed-to-key derivation, encapsulation and decapsulation,
  note round-trip and rejection of the wrong version or a malformed ciphertext.
- Size and performance benchmarks for transactions, blocks, proving and scanning
  are recorded and judged acceptable against network limits.
- The construction receives external cryptographic review.
- The change is deployed and exercised on a test network.

### Implementation Plan

1. Introduce the versioned ciphertext and address formats and the hybrid
   ML-KEM delivery behind the version tag.
2. Wallets and the SDK gain ML-KEM keys, issue the new addresses and scan for
   both formats.
3. A network upgrade opens the dual-support window: the ledger accepts both
   formats and senders use the new one for recipients with an ML-KEM key.
4. A later upgrade closes the window and stops accepting legacy ciphertexts in
   new outputs.

## Backwards Compatibility Assessment

This is a breaking change that requires a network upgrade. The new output and
ciphertext serialization and the new address format are not backward compatible
with current nodes and wallets. Under the native binding proposed here, the
output circuit and its verifying keys are unchanged, because the proof binds
only a natively computed hash of the ciphertext.

The version tag lets the two formats coexist during the dual-support window.
Outputs created before the upgrade keep their Jubjub ciphertexts and are still
decrypted on the legacy path, so no on-chain data is migrated or re-encrypted.
Historical ciphertexts therefore stay exposed to harvest-now-decrypt-later; this
sunk exposure is inherent and is discussed in the originating MPS.

The parts of Zswap outside note delivery are unaffected. The coin public key,
coin commitments, nullifiers, value commitments, balancing and proof soundness
are unchanged, so ownership and integrity guarantees carry over as they are.

## Performance and Network Impact

The proving circuit does not change. Note encryption in Zswap runs outside the
zero-knowledge circuit; the output proof binds only a hash of the ciphertext as
a public input, and the circuit neither encrypts nor inspects the ciphertext.
Adding a KEM to the Jubjub exchange keeps encryption outside the circuit,
so the output circuit, its proving time and its verifying keys are unaffected by
the larger ciphertext. This holds as long as the KEM is not verified in-circuit;
proving lattice operations inside the proof system would be a far more expensive
proposition, which is why this MIP keeps the native binding.

The dominant cost is size. Measurements in midnight-ledger issue 797 put today's
note ciphertext at about 230 bytes serialized: a 32-byte Jubjub point and six
field elements. Adding a 1088-byte ML-KEM-768 encapsulation brings it to about
1318 bytes, so a shielded output grows by roughly one kilobyte. An address grows
from 64 bytes to 1248 with the 1184-byte ML-KEM-768 public key. Pure ML-KEM
would save 32 bytes on each: about 1286 bytes per ciphertext and 1216 per
address.

That size growth is where the network feels the change.

- **Block space and chain growth.** Every shielded output is about a kilobyte
  larger, so blocks hold fewer transactions at the same byte budget and the
  chain grows faster. Block-size and per-transaction limits may need revisiting.
- **Bandwidth and storage.** Full nodes propagate and store larger transactions
  and a larger ledger, and indexers and light-wallet backends carry more per
  output.
- **Address length.** A 1248-byte address encodes to about 2000 Bech32m
  characters, which affects how addresses are displayed, copied and shared, for
  example as QR codes.
- **Fees.** If fees are size-sensitive, shielded transfers become more
  expensive, and the fee schedule should reflect the larger outputs.
- **Verification.** The verifier recomputes the ciphertext hash natively over a
  larger input. Hashing about a kilobyte to a field element is cheap next to the
  proof verification it accompanies, so per-output verification cost rises only
  marginally.
- **Wallet scanning.** A wallet decapsulates once per candidate output, on top
  of the Jubjub scalar multiplication and the Poseidon step it already runs.
  Measurements in midnight-ledger issue 797 (Apple M4, one thread) put
  ML-KEM-768 decapsulation at about 33 microseconds, against about 259 for a
  full trial decryption today, so scanning cost rises by roughly an eighth. The
  larger data a wallet must fetch and store is the real change.

The migration is therefore viable without touching the proving system, and the
engineering question is size rather than circuit complexity. The size increase
is significant in aggregate, and its effect on block limits, fees and storage
should be measured on a test network before the switch.

## Security Considerations

**Exposure closed for new outputs.** Notes sent in the new format are no longer
harvest-now-decrypt-later targets. Legacy-format notes sent during the
dual-support window stay exposed, so the window should be as short as the
ecosystem allows.

**Residual historical exposure.** Notes delivered before the upgrade stay
decryptable by a future CRQC. This cannot be remedied after the fact and should
be communicated plainly.

**KEM assumption.** Against a CRQC, confidentiality rests on ML-KEM's
module-lattice hardness. The hybrid exchange retains the classical hedge, so a
break of either primitive alone does not expose the note. This requires the
key derivation to depend on both shared secrets.

**Delivery integrity.** As today, a wrong key is caught by the zero-check
element and genuine ownership by the coin-commitment recomputation. Without a
tag the symmetric layer is malleable, so the construction does not meet the
textbook chosen-ciphertext definition on its own. The gap is closed by the
system around it: the proof binding stops third parties from altering a
ciphertext, a wallet reveals nothing about what a decryption produced and the
only party able to send a malformed note is the sender, who holds the key
anyway and can at most make the coin unrecoverable. A KEM's implicit rejection
returns a pseudorandom secret for a wrong key, so a foreign ciphertext still
fails the commitment match rather than silently decrypting.

**Binding.** The native ciphertext hash used for non-malleability must stay
collision and second-preimage resistant over the larger input.

**Denial of service through size.** Larger ciphertexts and addresses raise the
cost of flooding the network with shielded outputs. Size limits and fees must
price the new outputs so oversized or malformed ciphertexts cannot grief nodes.

**Key domain separation.** The ML-KEM key must be derived under its own seed
domain, distinct from the coin key and any other derived key, so seed material
is never reused across primitives.

## Implementation

- **Transient crypto.** The delivery primitive gains an ML-KEM implementation
  and the key-derivation step from the two shared secrets to the Poseidon key.
  The Jubjub exchange is retained as the classical half of the hybrid.
- **Zswap ledger.** The coin-ciphertext structure and its serialization gain the
  encapsulation field and the version tag; the native ciphertext-hash binding is
  extended to the new format; output construction and verification carry the new
  ciphertext; key derivation, the wallet-facing key types and the scan path
  handle both formats. The output circuit does not change under the native
  binding. The versioned ciphertext follows the pattern the ledger already uses
  for proofs, where a leading discriminant lets two proof versions be verified
  side by side within one ledger version.
- **WASM and SDK.** The wallet-facing encryption key types and the address
  assembly are updated, along with the key-derivation test vectors.
- **Dependencies.** A vetted ML-KEM (FIPS 203) implementation is added.

## Testing

- **Unit.** Seed-to-key derivation vectors for ML-KEM keys; encapsulation and
  decapsulation; note encryption and decryption round-trip; version-tag parsing;
  rejection of a wrong version and of malformed or oversized ciphertexts; the
  hybrid key derivation.
- **Integration.** End-to-end creation, scanning and spending of a shielded
  output across a network round trip; coexistence of legacy Jubjub and new KEM
  outputs in one wallet and one ledger state; proof binding against the new
  ciphertext.
- **Performance.** Benchmarks for transaction and block size, proving time,
  verification time and wallet scan time, and their effect on block-size and fee
  limits, recorded on a test network.
- **Conformance.** The ledger and the SDK agree on key derivation, wire format
  and scanning against shared test vectors.

## References

- Originating MPS:
  [MPS-0050](/mps/mps-0050-post-quantum-confidentiality-zswap.md)
  (Post-Quantum Confidentiality for Zswap).
- F. Engelmann, T. Kerber, M. Kohlweiss, M. Volkhov. *Zswap: zk-SNARK Based
  Non-Interactive Multi-Asset Swaps.* Proceedings on Privacy Enhancing
  Technologies 2022(4). IACR ePrint 2022/1002.
  <https://eprint.iacr.org/2022/1002>
- NIST FIPS 203: *Module-Lattice-Based Key-Encapsulation Mechanism Standard*
  (ML-KEM), August 2024. <https://doi.org/10.6028/NIST.FIPS.203>
- NIST SP 800-227: *Recommendations for Key-Encapsulation Mechanisms*,
  September 2025. Section 4.6 covers composite KEMs and approved key combiners.
  <https://doi.org/10.6028/NIST.SP.800-227>
- RFC 9954: *Hybrid Key Exchange in TLS 1.3*, July 2026. The hybrid
  construction deployed in TLS, combining a classical and a post-quantum key
  exchange. <https://www.rfc-editor.org/info/rfc9954>
- D. Connolly, P. Schwabe, B. Westerbaan. *X-Wing: general-purpose hybrid
  post-quantum KEM*, Internet-Draft draft-connolly-cfrg-xwing-kem, work in
  progress. A hybrid of X25519 and ML-KEM-768 with a specified combiner.
  <https://datatracker.ietf.org/doc/draft-connolly-cfrg-xwing-kem/>
- RFC 9180: *Hybrid Public Key Encryption* (HPKE), February 2022. Prior art for
  a KEM followed by a symmetric layer; here "hybrid" means KEM plus symmetric
  encryption, not classical plus post-quantum.
  <https://www.rfc-editor.org/info/rfc9180>
- Related MIPs: [MIP-0001](/mips/mip-0001-mip-process.md) (proposal process);
  [MIP-0003](/mips/mip-0003-ecdsa-support.md) (ECDSA support), a precedent for
  adding a scheme with its own key derivation and domain separation; and
  [MIP-0021](/mips/mip-0021-domain-separation.md) (domain-separation
  convention).
- Size and timing measurements: midnight-ledger issue 797.
  <https://github.com/midnightntwrk/midnight-ledger/issues/797>
- Ledger implementation (`midnightntwrk/midnight-ledger`, branch `ledger-9`),
  evidence trail: `transient-crypto/src/encryption.rs`,
  `zswap/src/structure.rs`, `zswap/src/lib.rs`, `zswap/src/keys.rs`,
  `zswap/src/construct.rs`, `zswap/src/verify.rs`, `zswap/src/local.rs`,
  `zswap/zswap.compact`, `ledger/src/structure.rs`.
- Wallet address format (`midnightntwrk/midnight-wallet`):
  `packages/address-format/src/index.ts`.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under
the Apache License, Version 2.0. Submission requires agreement to the Midnight
Foundation Contributor License Agreement, which includes the assignment of
copyright for your contributions to the Foundation.

Portions of this document were drafted with the assistance of a large language
model. The named author reviewed all content and is solely accountable for it.
