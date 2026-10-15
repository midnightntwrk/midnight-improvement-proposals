---
MPS: "xxxx"
Title: Persistent Full-Range Hashing into the Native Field
Authors:
  - Iskander Andrews <0xisk>
Status: Proposed
Category: Core | Libraries and Tooling
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

Compact has no persistent hash whose output covers the native `Field`. The
persistent primitive, `persistentHash`, returns 32 bytes. The standard
library's bridge from those bytes to a `Field` is `degradeToTransient`, which keeps the low 31 bytes
and drops the 32nd. Its output is below `2^248` in a field of 255 bits, so it
is neither full-range nor uniform. The only `Field`-valued hash,
`transientHash`, is documented as not guaranteed to persist across upgrades and
must not derive state. A `Bytes<32> as Field` cast is range-checked and fails
on values at or above the modulus, while the same cast to every foreign field
type reduces modulo that field.

A contract that needs a persistent, uniform `Field` from hashed data therefore
cannot get one from the standard library. The concrete case is OpenZeppelin's
[Confidential Note Fungible Token](https://github.com/OpenZeppelin/compact-contracts/issues/722).
Its [Delivery extension](https://github.com/OpenZeppelin/compact-contracts/issues/727)
sends each output note's `(value, nonce)` to the recipient on chain, encrypted
by adding a mask derived from an ECDH shared secret. The nonce has type
`Field`, and whoever learns it derives the note's nullifier and can race the
owner to spend it. The mask is `degradeToTransient(persistentHash(...))`, so
it is below `2^248` and `nonce + mask` never wraps: the ciphertext brackets the
nonce in a `2^248`-wide window. The core derives its own nonces at 248 bits,
which leaves no hiding margin at all, and a note built by a composer may carry
any `Field` value, whose top bits are then published outright. A full-field
pad, [compact-contracts#913](https://github.com/OpenZeppelin/compact-contracts/pull/913),
reimplements RFC 9380 `hash_to_field` on top of SHA-256 with hand-written
field arithmetic in Compact, at several times the circuit cost of the
truncating path. The token sidestepped it by typing the plaintext `Uint<248>`
under the existing pad
([compact-contracts#1150](https://github.com/OpenZeppelin/compact-contracts/pull/1150)),
which fits its own nonces and refuses wider ones.

The ledger and ZKIR layers already contain the missing pieces: a Poseidon
`hash_to_field`, a `FromBytes32` instruction that reduces into the native
field, and a wide-input reduction. None is reachable from Compact. No consumer
is blocked today, so this MPS is filed at low priority: it records the gap so
that the next library needing a uniform `Field` finds a written statement
instead of rediscovering it.

## Vision

A Compact author who needs a field element from hashed data picks a standard
library circuit by two properties, persistence and output range, and gets
exactly that. The documentation of every bytes-to-field conversion states what
it keeps, what it drops, and whether its output is uniform. A hiding mask, a
state-derived key, or a nonce is built from one library call, with no
hand-written field arithmetic and no digest reimplementation in Compact.

## Problem

- **`degradeToTransient` truncates.** The ledger
  [returns the 31-byte chunk](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/transient-crypto/src/hash.rs#L70-L73)
  of the field representation of the 32-byte input. The compiler
  [lowers the call to a plain `copy`](https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/compiler/zkir-v3-passes/reduce-to-zkir.ss#L154-L156).
  The output is below `2^248` in a 255-bit field. The
  [standard library documentation](https://docs.midnight.network/compact/standard-library/exports#degradetotransient)
  describes it as a conversion between hash outputs and does not say that 8
  bits are dropped. The
  [native declaration](https://github.com/LFDT-Minokawa/compact/blob/compactc-v0.35.0/compiler/midnight-natives.ss#L44-L47)
  labels its disclosure as "a modulus of" the input, though nothing is reduced.
- **No persistent `Field` hash.** `transientHash` is the only hash returning a
  `Field`. Its documentation states it is not guaranteed to persist between
  upgrades and should not derive state. It is also the only circuit-efficient
  hash Compact exposes, so contracts that derive state fall back to SHA-256
  and pay for it in rows
  ([compact#183](https://github.com/LFDT-Minokawa/compact/issues/183)).
- **Casts are asymmetric.** `Bytes<32> as Field` performs a run-time range
  check and is a dynamic error at or above the modulus, and the circuit
  lowering constrains the result. `Bytes<32> as Secp256k1Base`, the other
  foreign fields, and `Bytes<64> as Curve25519Scalar` instead reduce modulo
  their modulus. A random 32-byte digest fails the native cast with
  probability above one half.
- **The primitive exists below Compact.** ZKIR v3 has a
  [`FromBytes32` instruction](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/zkir-v3/src/ir.rs#L502-L520)
  over the `Native` type that reduces non-canonical input, with
  [off-circuit and in-circuit implementations](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/zkir-v3/src/ir_instructions/from_bytes32.rs).
  The ledger has a Poseidon-based
  [`hash_to_field`](https://github.com/midnightntwrk/midnight-ledger/blob/c38ea00f691906eac82a15be116dfb9a543d4e34/transient-crypto/src/hash.rs#L54-L61)
  and a 64-byte `from_uniform_bytes`. The compiler emits none of them for the
  native field.
- **Reducing 32 bytes is not enough.** A 256-bit integer reduced modulo a
  255-bit prime has a visible bias between residues. A uniform output needs a
  wider input, as RFC 9380 prescribes.
- **Transition on a proving-system change is unspecified.** The documentation
  says which hashes persist. It does not say what happens to a deployed
  contract that depends on a transient primitive when the proving system
  changes: a transition window or a hard cutover.

## Use Cases

- **Field plaintext under a one-time pad.** The
  [Confidential Note Fungible Token core](https://github.com/OpenZeppelin/compact-contracts/pull/743)
  stores value in notes whose
  [nonce is a `Field`](https://github.com/OpenZeppelin/compact-contracts/blob/2d1a71b960b200955942404ba85d400b116d7746/contracts/src/token/ConfidentialNoteFungibleToken.compact#L157-L160)
  and the sole nullifier preimage. The core derives it with
  `degradeToTransient`, so its own nonces are 248 bits, but `_mintNote` accepts
  caller-built notes with any `Field` nonce. The
  [Delivery extension](https://github.com/OpenZeppelin/compact-contracts/issues/727)
  publishes each output note's `(value, nonce)` encrypted to the recipient
  under ECDH, so wallets discover funds from chain data alone. The mask comes
  from the
  [`EcdhMask` KDF](https://github.com/OpenZeppelin/compact-contracts/blob/f270f73a14394f34e36f591c5d7886289d2aeffd/contracts/src/crypto/EcdhMask.compact#L119-L124),
  `degradeToTransient(persistentHash(...))`, so it is below `2^248` and the sum
  never wraps. The ciphertext then brackets the nonce in a `2^248`-wide window:
  no margin for a core nonce, and the top bits exposed for a wider one. The
  extension must hide the type, not one derivation of it
  ([compact-contracts#735](https://github.com/OpenZeppelin/compact-contracts/issues/735)).
- **Persistent key derivation.** A library derives a `Field`-typed secret or
  nonce from a seed and must get the same value after a toolchain upgrade.
  Today it either truncates or depends on `transientHash`.
- **Commitments in the field.** A Merkle leaf or commitment that must be a
  `Field` is built from `transientCommit`, which carries no persistence
  guarantee, or from a truncated SHA-256 digest.
- **The workaround.** The full-field answer,
  [compact-contracts#913](https://github.com/OpenZeppelin/compact-contracts/pull/913)
  (closed, branch kept), reimplements RFC 9380 `hash_to_field` on SHA-256
  with a hand-written
  [`Fq` module](https://github.com/OpenZeppelin/compact-contracts/blob/feat/ecdh-mask-field-kdf/contracts/src/crypto/curves/bls12-381/Fq.compact)
  in Compact. The token chose the narrower route,
  [compact-contracts#1150](https://github.com/OpenZeppelin/compact-contracts/pull/1150),
  which types the plaintext `Uint<248>` under the existing pad. Measured on
  the 913 branch with the toolchain in the references (`k`, rows):

  | Circuit | Standard-library path | Full-field workaround |
  | --- | --- | --- |
  | ECDH mask derivation | 14, 8700 | 14, 14055 |
  | SHA-256 of 32 bytes, then to `Field` | 13, 2305 | 14, 9806 |
  | Wide-bytes reduction into the field alone | n/a | 11, 2006 |

## Goals

1. A standard library circuit from hashed data to a `Field` whose output is
   statistically uniform over the whole field and is guaranteed to persist
   across upgrades.
2. A stable, named, circuit-efficient hash into `Field` that contracts may use
   to derive state.
3. Documentation of every bytes-to-field path stating what is kept, what is
   dropped, whether the result is uniform, and whether it persists.
4. A single cast rule for `Bytes` to field types, or an explicit statement of
   why the native field differs.
5. A published statement of how contracts that depend on transient primitives
   transition across a proving-system change.

## Expected Outcomes

Libraries stop reimplementing hash-to-field in Compact. A hiding mask or a
persistent field-valued key costs one library call and a fraction of the
current rows. Authors can read, from the documentation alone, whether a
conversion loses bits or bias. The pieces the ledger and ZKIR already carry
become usable from contracts.

## Open Questions

- Should the full-range map be a new circuit, a reducing cast from a wide
  `Bytes<n>` to `Field` like the foreign-field casts, or both?
- What input width gives acceptable uniformity for the native field, and
  should it follow RFC 9380 `L` for the 255-bit modulus?
- Should `degradeToTransient` keep its name and semantics, with its truncation
  documented, or be deprecated once a full-range map exists?
- The ZKIR instructions that split a `Bytes<32>` into the limbs
  `degradeToTransient` copies from are documented as a temporary bridge. When
  they go, does `degradeToTransient` keep its output? It feeds the standard
  library Merkle leaf hash, Dust commitments, and `evolveNonce`, so a change
  there is consensus-affecting.
- Can the ledger's Poseidon `hash_to_field` be the persistent circuit-efficient
  option, and under which persistence guarantee?
- Where should the persistence guarantee across proving-system changes be
  specified: the standard library documentation, the ledger specification, or
  a MIP of its own?
- Is a transition window possible, or does a proving-system change imply a
  hard cutover for contracts that depend on transient primitives?

## Recommended MIPs

- **Persistent hash to the native field.** Specify a standard library circuit
  from arbitrary input to a uniform `Field` with a persistence guarantee,
  including input width, expansion, and domain separation, and its lowering to
  ZKIR.
- **Stable named hashes.** Expose `poseidon` and `sha256` as explicitly named
  circuits with stated guarantees, with `transientHash` and `persistentHash`
  kept as aliases, and the rename applied first in the on-chain runtime
  ([compact#183](https://github.com/LFDT-Minokawa/compact/issues/183)).
- **Conversion semantics and documentation.** Document the truncation of
  `degradeToTransient` and the range-check of `Bytes<32> as Field`, and settle
  the cast asymmetry with the foreign field types.

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
