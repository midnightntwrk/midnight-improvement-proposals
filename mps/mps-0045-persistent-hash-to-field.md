---
MPS: "0045"
Title: Persistent Full-Range Hashing into the Native Field
Authors:
  - Iskander Andrews (0xisk)
Status: Proposed
Category: Libraries and Tooling
Created: 14-Oct-2026
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

Compact has no persistent hash that returns a full native `Field`.
`persistentHash` returns 32 bytes. The standard library bridge to a `Field`,
`degradeToTransient`, keeps the low 31 bytes and drops the 32nd, so its
output is below `2^248` in a 255-bit field: not full-range, not uniform. The
only hash that returns a `Field`, `transientHash`, is documented as not
guaranteed to survive upgrades, so it must not derive state. Casts do not
help either: `Bytes<32> as Field` is range-checked and fails at or above the
modulus, while the same cast to every foreign field type reduces.

Our case is OpenZeppelin's
[Confidential Note Fungible Token](https://github.com/OpenZeppelin/compact-contracts/issues/722),
a Compact module. Below, "the contract" is the program that imports it.

- A balance is a set of
  [notes](https://github.com/OpenZeppelin/compact-contracts/blob/e612622df401b4c8e8f2ce1853b2a0f24809f65b/contracts/src/token/ConfidentialNoteFungibleToken.compact#L160-L163),
  `(value: Uint<128>, nonce: Field)`, committed into a Merkle tree.
- A note is spent by publishing its
  [nullifier](https://github.com/OpenZeppelin/compact-contracts/blob/e612622df401b4c8e8f2ce1853b2a0f24809f65b/contracts/src/token/ConfidentialNoteFungibleToken.compact#L662-L667),
  `persistentHash(domain, nonce)`. Whoever knows the nonce can spend the note
  first.
- [`_mint`](https://github.com/OpenZeppelin/compact-contracts/blob/e612622df401b4c8e8f2ce1853b2a0f24809f65b/contracts/src/token/ConfidentialNoteFungibleToken.compact#L367-L372)
  derives the nonce through
  [`freshNonce`](https://github.com/OpenZeppelin/compact-contracts/blob/e612622df401b4c8e8f2ce1853b2a0f24809f65b/contracts/src/token/ConfidentialNoteFungibleToken.compact#L390-L396),
  `degradeToTransient(persistentHash(...))`, so it is 248 bits.
- [`_mintNote`](https://github.com/OpenZeppelin/compact-contracts/blob/e612622df401b4c8e8f2ce1853b2a0f24809f65b/contracts/src/token/ConfidentialNoteFungibleToken.compact#L445)
  takes a note the contract built, so its nonce can be any `Field`. Every
  commit goes through it: `_mint`, `_transfer` and `_burn` all call it with
  the `Note` values the contract passes in.
- The
  [Delivery extension](https://github.com/OpenZeppelin/compact-contracts/issues/727)
  publishes each output note's `(value, nonce)` encrypted for the recipient
  as `nonce + mask`. The mask is the
  [`EcdhMask` KDF](https://github.com/OpenZeppelin/compact-contracts/blob/f270f73a14394f34e36f591c5d7886289d2aeffd/contracts/src/crypto/EcdhMask.compact#L119-L124),
  again `degradeToTransient(persistentHash(...))`, so below `2^248`.

Since the mask is below `2^248`, `nonce + mask` never wraps, and the
ciphertext places the nonce in a window of size `2^248`. For a `_mint` nonce
the window is as wide as the nonce, so only the edge leaks: about 0.7 bits
on average, and a `(nonce, mask)` pair that makes the nonce searchable has
probability around `2^-337`. For a contract-chosen nonce above `2^248` there is no
probability in it, the top bits are simply visible in the ciphertext.

We had two ways out:

- A full-field pad,
  [compact-contracts#913](https://github.com/OpenZeppelin/compact-contracts/pull/913),
  that reimplements RFC 9380 `hash_to_field` on SHA-256 with hand-written
  field arithmetic in Compact, at several times the rows.
- Typing the plaintext as `Uint<248>` under the existing pad,
  [compact-contracts#1150](https://github.com/OpenZeppelin/compact-contracts/pull/1150).
  The leak on the token's own nonces is negligible, and the range-checked
  cast refuses a wider nonce instead of leaking it.

The token took the second.

The ledger has a Poseidon `hash_to_field`, ZKIR has a reducing `FromBytes32`
and a wide-input reduction. None is reachable from Compact. Nobody is blocked
today, so this MPS is low priority. It writes the gap down so the next
library needing a uniform `Field` does not rediscover it.

## Vision

An author who needs a `Field` from a hash picks a standard library circuit by
persistence and output range and gets exactly that. The docs of every
bytes-to-field conversion say what is kept, what is dropped, and whether the
output is uniform. A mask, a key, or a nonce is one library call, with no
field arithmetic written in Compact.

## Problem

- **`degradeToTransient` truncates.** The ledger
  [returns the 31-byte chunk](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/transient-crypto/src/hash.rs#L70-L73)
  of the input's field representation and the compiler
  [lowers the call to a `copy`](https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/compiler/zkir-v3-passes/reduce-to-zkir.ss#L154-L156).
  The
  [docs](https://docs.midnight.network/compact/standard-library/exports#degradetotransient)
  never say 8 bits are dropped, and the
  [native declaration](https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/compiler/midnight-natives.ss#L44-L47)
  calls its disclosure "a modulus of" the input, though nothing is reduced.
- **No persistent `Field` hash.** `transientHash` is the only hash returning
  a `Field` and the only circuit-cheap one. Contracts that derive state fall
  back to SHA-256 and pay in rows
  ([compact#183](https://github.com/LFDT-Minokawa/compact/issues/183)).
- **Casts are inconsistent.** `Bytes<32> as Field` range-checks, in the
  runtime and in circuit. `Bytes<32> as Secp256k1Base`, the other foreign
  fields, and `Bytes<64> as Curve25519Scalar` reduce. A random 32-byte digest
  fails the native cast more than half the time.
- **The primitive exists below Compact.** ZKIR v3 has a reducing
  [`FromBytes32`](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/zkir-v3/src/ir.rs#L502-L520)
  over `Native`
  ([implementation](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/zkir-v3/src/ir_instructions/from_bytes32.rs)),
  the ledger has a Poseidon
  [`hash_to_field`](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/transient-crypto/src/hash.rs#L54-L61)
  and a 64-byte `from_uniform_bytes`. The compiler emits none of them for the
  native field.
- **Reducing 32 bytes is not enough.** A 256-bit integer modulo a 255-bit
  prime has a visible bias. Uniform output needs a wider input, as in
  RFC 9380.

## Use Cases

- **A `Field` plaintext under a one-time pad.** The Delivery case above. The
  nonce type is `Field` and
  [`_mintNote`](https://github.com/OpenZeppelin/compact-contracts/blob/e612622df401b4c8e8f2ce1853b2a0f24809f65b/contracts/src/token/ConfidentialNoteFungibleToken.compact#L445)
  accepts any value of it. A pad below `2^248` cannot hide the top 7 bits of
  a wider nonce. Hiding what the type allows needs a pad uniform over the
  whole `Field`
  ([compact-contracts#735](https://github.com/OpenZeppelin/compact-contracts/issues/735)).
  Today the type is narrowed instead.
- **Persistent key derivation.** A `Field` secret or nonce derived from a
  seed must come back the same after an upgrade. Today it is truncated or
  depends on `transientHash`.
- **Commitments in the field.** A `Field` Merkle leaf or commitment comes
  from `transientCommit`, with no persistence guarantee, or from a truncated
  SHA-256 digest.
- **The workaround.**
  [compact-contracts#913](https://github.com/OpenZeppelin/compact-contracts/pull/913)
  (closed, branch kept) reimplements RFC 9380 `hash_to_field` on SHA-256 with
  a hand-written
  [`Fq` module](https://github.com/OpenZeppelin/compact-contracts/blob/feat/ecdh-mask-field-kdf/contracts/src/crypto/curves/bls12-381/Fq.compact).
  Cost on that branch with the toolchain in the references (`k`, rows):

  | Circuit | Standard-library path | Full-field workaround |
  | --- | --- | --- |
  | ECDH mask derivation | 14, 8700 | 14, 14055 |
  | SHA-256 of 32 bytes, then to `Field` | 13, 2305 | 14, 9806 |
  | Wide-bytes reduction into the field alone | n/a | 11, 2006 |

## Goals

1. A standard library circuit from a hash to a `Field` that is uniform over
   the whole field and persists across upgrades.
2. A stable, named, circuit-cheap hash into `Field` that may derive state.
3. Docs for every bytes-to-field path: what is kept, what is dropped, uniform
   or not, persistent or not.

## Expected Outcomes

Libraries stop reimplementing hash-to-field in Compact. A mask or a
persistent `Field` key is one call at a fraction of today's rows. The docs
alone tell an author whether a conversion loses bits or has a bias.

## Open Questions

- Should the full-range map be a new circuit, a reducing cast from a wide
  `Bytes<n>` to `Field` like the foreign-field casts, or both? And which
  input width: RFC 9380 `L` for a 255-bit modulus, or something else?
- Can the ledger's Poseidon `hash_to_field` be the persistent circuit-cheap
  option, and under which guarantee?

## Recommended MIPs

- **Persistent hash to the native field.** A standard library circuit from
  any input to a uniform `Field` with a persistence guarantee: input width,
  expansion, domain separation, ZKIR lowering.
- **Stable named hashes.** `poseidon` and `sha256` as named circuits with
  stated guarantees, `transientHash` and `persistentHash` kept as aliases,
  rename done in the on-chain runtime first
  ([compact#183](https://github.com/LFDT-Minokawa/compact/issues/183)).
- **Conversion docs.** Document the truncation in `degradeToTransient` and
  the range check in `Bytes<32> as Field`.

## References

- Compact standard library exports, `degradeToTransient`, `transientHash`,
  `persistentHash`:
  https://docs.midnight.network/compact/standard-library/exports
- Compact compiler `compactc-v0.35.0`:
  native declarations
  https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/compiler/midnight-natives.ss,
  ZKIR lowering of `degradeToTransient` and byte casts
  https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/compiler/zkir-v3-passes/reduce-to-zkir.ss,
  cast rules
  https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/doc/compact-reference.mdx
- Midnight ledger, branch `ledger-8` at `c38ea00f`:
  `degrade_to_transient`, `hash_to_field`, `transient_hash`
  https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f/transient-crypto/src/hash.rs,
  byte-to-field representation
  https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f/transient-crypto/src/repr.rs,
  ZKIR v3 `FromBytes32`, `Bytes32IntoLowHigh`, `ReconstituteField`
  https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f/zkir-v3/src/ir.rs,
  https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f/zkir-v3/src/ir_instructions/from_bytes32.rs
- Midnight ZK: `midnight-circuits` Poseidon and SHA-256 chips,
  `midnight-zk-stdlib` `poseidon`, `sha2_256`, `assigned_from_le_bytes`:
  https://github.com/midnightntwrk/midnight-zk
- Compact issue, expose Poseidon from Compact:
  https://github.com/LFDT-Minokawa/compact/issues/183
- OpenZeppelin compact-contracts: Confidential Note Fungible Token
  https://github.com/OpenZeppelin/compact-contracts/issues/722, core
  https://github.com/OpenZeppelin/compact-contracts/pull/743, Delivery
  extension https://github.com/OpenZeppelin/compact-contracts/issues/727,
  `Field` pad for `EcdhMask`
  https://github.com/OpenZeppelin/compact-contracts/issues/735, full-field pad
  and row measurements (closed, branch kept)
  https://github.com/OpenZeppelin/compact-contracts/pull/913, the `Uint<248>`
  route taken instead
  https://github.com/OpenZeppelin/compact-contracts/pull/1150
- RFC 9380, Hashing to Elliptic Curves, `hash_to_field`:
  https://www.rfc-editor.org/rfc/rfc9380#section-5.2
- MPS-0027, Domain Separation for Midnight Hash Constructions; MPS-0008,
  Keccak-256 hashing; MPS-0011, native cryptographic primitives.

## Acknowledgements


## Copyright

This MPS is licensed under CC-BY-4.0.
