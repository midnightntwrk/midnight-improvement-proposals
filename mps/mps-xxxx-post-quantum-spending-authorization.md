---
MPS: "xxxx" # to be assigned
Title: Post-Quantum Spending Authorization for Unshielded Outputs
Authors:
- Quantova Inc (quantova-inc)
Status: Proposed
Category: Core
Created: 16-Oct-2026
Requires: none
Replaces: none
MIP: none
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

Unshielded outputs on Midnight are owned by a hash of a BIP340 verifying key and
spent with a BIP340 Schnorr signature over secp256k1. A cryptographically
relevant quantum computer (CRQC) running Shor's algorithm recovers the signing
key from any verifying key it can see, and can then spend every output still
held under that key. A key stays hidden behind its address hash until its first
spend, so exposure grows with every spend and every reused address. Reward
claims carry the verifying key directly and are exposed from the start.

MPS #362 covers the confidentiality of shielded notes. This MPS covers the
complementary integrity problem, the authorization of unshielded spends and
reward claims. It is solution agnostic. It records the current signing paths,
the exposure classes, the measured cost of the standardised post quantum
candidates on the current transaction shape, and the conditions a migration
has to meet.

## Vision

Every unshielded output can be owned and spent under a post quantum signature.
Holders can move to it well before a CRQC is credible, through an authenticated
step they control, with no freeze, sweep or confiscation. The per input cost of
each candidate is measured and published, and the signed context of every
scheme is separated so that a signature made for one scheme or purpose can
never be accepted as another.

## Problem

### Current signing paths

- `base-crypto/src/schnorr.rs` implements BIP340 Schnorr over secp256k1 with
  `k256`.
- An unshielded output stores `owner: UserAddress`. `UserAddress` is
  `persistent_hash`, which is SHA-256, over the serialized `VerifyingKey`.
- A spend carries the full `VerifyingKey` in `UtxoSpend` and one signature per
  input in `UnshieldedOffer.signatures`, over the intent and the segment.
- `ClaimRewardsTransaction` carries `owner: VerifyingKey` and a signature
  directly.

### Exposure classes

1. Outputs at an address whose key has never signed. The key is hidden behind
   SHA-256. A quantum search against a 256 bit hash leaves about 128 bits of
   preimage security, so these outputs are not exposed by the signature scheme
   until their owner spends.
2. Outputs at an address whose key has already signed a spend. The key is
   public, and every output still held at that address is exposed.
3. Reward claims. The key is public from the first claim.

Only the first class can be protected by acting in time. Outputs in the second
and third classes need a migration that completes while BIP340 is still sound.

### Measured cost on the current transaction shape

Each spent input adds its verifying key and its signature. A node decodes the
key from every spend, because the key arrives in the transaction. Measured as
decode plus verify per input on one core, Apple M4, Rust 1.98, with the
harness in the references.

| Scheme | Public key (B) | Signature (B) | Added per input vs BIP340 (B) | Decode + verify | vs BIP340 |
|---|---:|---:|---:|---:|---:|
| BIP340 (today) | 32 | 64 | 0 | 33.4 µs | 1.0× |
| ML-DSA-44 | 1,312 | 2,420 | 3,636 | 35.8 µs | 1.07× |
| ML-DSA-65 | 1,952 | 3,309 | 5,165 | 58.6 µs | 1.75× |
| ML-DSA-87 | 2,592 | 4,627 | 7,123 | 95.9 µs | 2.9× |
| SLH-DSA-SHA2-128s | 32 | 7,856 | 7,792 | 502 µs | 15× |
| SLH-DSA-SHA2-128f | 32 | 17,088 | 17,024 | 1.37 ms | 41× |
| SLH-DSA-SHAKE-128f | 32 | 17,088 | 17,024 | 1.79 ms | 54× |

The larger cost is bytes. A spend grows by 3.6 KB to 7.1 KB per input under
ML-DSA, which falls on block space, fees, indexer storage and wallet and
hardware signer limits. Verification time stays within three times the
current path for every ML-DSA parameter set.

### Signed context

The signed context identifies the intent and the segment but not the signature
scheme, because only one scheme exists today. Adding a second scheme without a
scheme and purpose tag in the signed bytes allows a signature made for one
purpose to be presented for another. MPS-0027 already sets out domain
separation for Midnight hash constructions.

### Implementations

A migration depends on implementations that agree byte for byte. Two
independent Rust implementations of ML-DSA-65, RustCrypto `ml-dsa` 0.1.1 and
Quantova `qtv-crypto`, produce identical public keys from the same seed and
identical deterministic signatures, verify each other's signatures, and reject
altered messages, across 64 seeds and two contexts.

## Use Cases

- A long term holder whose address has never signed wants to move to a post
  quantum owner before a CRQC is credible, with a step their wallet and
  hardware signer can perform.
- A custodian that reuses deposit addresses holds outputs whose keys are
  already public. It needs a migration that completes in time and is
  authenticated by the current key.
- A validator or delegator claiming rewards exposes its key on the first claim.
  It needs a post quantum owner for future claims.
- A wallet or hardware signer must handle public keys of 1.3 KB to 2.6 KB and
  signatures of 2.4 KB to 4.6 KB under ML-DSA, or larger under SLH-DSA.

## Goals

1. A versioned owner type that can commit to a post quantum verifying key,
   while existing BIP340 addresses remain valid.
2. An authenticated migration that a holder performs with the current key while
   BIP340 is still sound, with no freeze, sweep or confiscation.
3. A scheme and purpose tag in every signed context, consistent with MPS-0027.
4. A published per input cost for each candidate on the real transaction
   shape, including size, decode and verify.
5. At least two independent implementations of the selected scheme that agree
   with the FIPS 204 or FIPS 205 vectors and with each other.

## Expected Outcomes

Holders, custodians and reward recipients have a defined path to post quantum
ownership. Wallet, hardware and indexer teams know the sizes and costs to plan
for. The network can activate the new owner type without disturbing existing
outputs, and can set its timeline against public guidance such as NIST IR 8547.

## Open Questions

- Which parameter set balances size and security for unshielded spends,
  ML-DSA-44, ML-DSA-65 or ML-DSA-87, and is SLH-DSA reserved for low frequency
  authorization such as governance and recovery?
- Should a transition period accept a hybrid of BIP340 and ML-DSA, and for how
  long?
- Should nodes cache expanded public keys under the address hash for repeat
  spends, and with what bound?
- What policy applies to dormant outputs whose keys are already exposed, given
  that freezes and sweeps are out of scope?
- How do the DUST key paths relate to this migration?

## Recommended MIPs

- Versioned owner and signature kind. Defines a tagged owner commitment and a
  post quantum signature kind beside BIP340, and the activation rule. Addresses
  goals 1 and 4.
- Authenticated ownership migration. Defines the conversion transaction signed
  by the current key, its replay and reorg rules, and wallet behaviour.
  Addresses goal 2.
- Signature context separation. Adds a scheme and purpose tag to every signed
  context, building on MPS-0027. Addresses goal 3.

## References

- NIST FIPS 204, Module-Lattice-Based Digital Signature Standard. https://csrc.nist.gov/pubs/fips/204/final
- NIST FIPS 205, Stateless Hash-Based Digital Signature Standard. https://csrc.nist.gov/pubs/fips/205/final
- NIST IR 8547 initial public draft, Transition to Post-Quantum Cryptography Standards. https://csrc.nist.gov/pubs/ir/8547/ipd
- MPS #362, Post-Quantum Confidentiality for Zswap. https://github.com/midnightntwrk/midnight-improvement-proposals/pull/362
- MPS-0027, Domain Separation for Midnight Hash Constructions.
- midnight-ledger issue 798, post quantum spending authorization, with the measurements above. https://github.com/midnightntwrk/midnight-ledger/issues/798
- Measurement and interoperability harness. https://github.com/quantova-inc/midnight-pq-signature-costs
- RustCrypto `ml-dsa`. https://github.com/RustCrypto/signatures
- Quantova `qtv-crypto`. https://github.com/Quantova/Q-Crypto

## Acknowledgements

The tracking issues opened by the Midnight team for the post quantum
programme, in particular midnight-ledger issues 796 to 798 and midnight-node
issues 2261 to 2263.

## Copyright

This MPS is licensed under CC-BY-4.0.
