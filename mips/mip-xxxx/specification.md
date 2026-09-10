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

# Shielded Note V2: Supporting Specification

## 1. Scope and Authority

This document supports [mips/mip-xxxx.md](../mip-xxxx.md) and specifies the v2 reference
construction, cryptographic relations, and validation behavior. It is self-contained at the
relation level; it does not describe deployed v2 functionality.
VRF-derived nullifiers, payment binding, and isolation of the spending key are required.

The main MIP defines the required properties and implementation latitude. This attachment
defines one detailed realization, including `AuthScope`, `PoseidonV2`, and compatibility
circuits. Specifying those choices here does not
silently remove alternatives that the main MIP leaves open. An alternative realization must
preserve the MIP's properties and define a single interoperable consensus specification.

The following distinction applies throughout:

- **Existing behavior:** facts checked against the ledger revision in Section 2.
- **V2 reference requirement:** a requirement or construction defined in this specification,
  not evidence that code implementing it exists or has been validated.
- **Unresolved:** a value, algorithm, artifact, or security result not yet defined or established.
  Section 13 lists these explicitly. No illustrative identifier below supplies a missing wire tag.

MUST, MUST NOT, SHOULD, and MAY have their RFC 2119/RFC 8174 meanings for the reference
construction. All cryptographic guarantees depend on the corresponding security assumptions
and correct enforcement of the specified relations. This is not a cryptographic audit report.

## 2. Existing Ledger Baseline

Ledger revision: `67f9f97246339539af0091c4d8c6bc5ab236a191`, corresponding to the checked
`release/ledger-9.1.0.0-rc.5` checkout. The source links below are pinned to this revision.

| Existing behavior | Source evidence |
| --- | --- |
| The user branch of `spend` witnesses a v1 coin secret key and uses it in ownership and nullifier derivation. | [Zswap circuit](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/zswap.compact#L34) |
| The circuit uses a depth-32 historic commitment tree and a nullifier set. Its value base depends on asset type and segment. | [Zswap state and spend relation](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/zswap.compact#L27) |
| The seed sampler prefixes an outer SHA-256 with a separator and hashes a little-endian round counter with the seed inside it. | [Seed sampler](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/keys.rs#L80) |
| Field-to-persistent-hash upgrade copies the stored low bytes into a zero-initialized 32-byte array. The generic transient hash is upgradeable. | [Hash conversions](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/transient-crypto/src/hash.rs#L64) |
| Input records expose a nullifier, value commitment, optional contract address, Merkle root, and proof, not plaintext coin data. Their equality and ordering include the proof. | [Input structure](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/structure.rs#L210) |
| Transient input verification uses a root reconstructed from the transient's output commitment at index 0. | [Transient conversion](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/structure.rs#L421) |
| Spend application rejects an input whose Merkle root is not retained. Each block inserts the current root and evicts entries older than the ledger's `global_ttl`, an on-chain parameter whose pinned crate default is 3600 seconds. | [Root check](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/ledger.rs#L73), [retention](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/ledger.rs#L241), [default](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/structure.rs#L1371) |
| Offer merge checks disjointness of complete records, not the proposed proof-erased scope identities. | [Offer merge](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/structure.rs#L584) |
| Spend verification uses fixed built-in legacy material. Output verification binds ciphertext through its public statement. | [Verifier](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/src/verify.rs#L58) |
| Both user circuits derive their value base with `hashToCurve`. The ledger evaluates that instruction natively through `transient_crypto::hash::hash_to_curve`, the CPU form of `HashToCurveGadget` from the locked `midnight-circuits` 7.2.4 crate, and synthesizes it in-circuit through the zk standard library's `hash_to_curve`. | [Circuit value base](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zswap/zswap.compact#L62), [host hash-to-curve](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/transient-crypto/src/hash.rs#L91), [zkir dispatch](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/zkir/src/ir_vm.rs#L812), [lock](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/Cargo.lock#L3015) |

That gadget, read at midnight-zk `f0e786c5d55f92a29bbf4cc2d825060f3803e365` in
[`htc_gadget.rs`](https://github.com/midnightntwrk/midnight-zk/blob/f0e786c5d55f92a29bbf4cc2d825060f3803e365/circuits/src/ecc/hash_to_curve/htc_gadget.rs#L99)
and [`mtc.rs`](https://github.com/midnightntwrk/midnight-zk/blob/f0e786c5d55f92a29bbf4cc2d825060f3803e365/circuits/src/ecc/hash_to_curve/mtc.rs#L55),
whose `hash_to_curve` sources are byte-identical to the locked crate, absorbs the input field
elements into a Poseidon sponge, squeezes two elements, maps each with a Shallue-van de
Woestijne map whose square root and sign are constrained in-circuit, clears the cofactor by
constrained multiplication by 8, and adds the two points. It takes no domain-separation
parameter and does not assert a non-identity output.

Existing serialization tags such as `zswap-input[v2]` are record-format versions; they do not
mean that these records already implement the proposed shielded note version 2. Likewise, a
source-level v2 formula is not a proof that a current verifier accepts it.

### 2.1 Local compatibility boundaries

These observations concern the local revisions checked on 2026-09-10, not a deployment manifest:

| Component | Checked revision | Compatibility observation |
| --- | --- | --- |
| Node | `e01e41f83e391b5d36d30b6d62eccc19dcf0fd16` | [Runtime `spec_version`](https://github.com/midnightntwrk/midnight-node/blob/e01e41f83e391b5d36d30b6d62eccc19dcf0fd16/runtime/src/lib.rs#L283) is `2_001_000`; [its ledger dependency](https://github.com/midnightntwrk/midnight-node/blob/e01e41f83e391b5d36d30b6d62eccc19dcf0fd16/Cargo.toml#L459) pins rc.4, one release candidate behind this attachment's rc.5 baseline. The runtime version is the value by which downstream components select ledger handling. |
| Indexer | `668ed0258ac92bb25ed02cabe6075274d8fbaac8` | [Version dispatch](https://github.com/midnightntwrk/midnight-indexer/blob/668ed0258ac92bb25ed02cabe6075274d8fbaac8/indexer-common/src/domain/protocol_version.rs#L71) accepts ledger-9 runtimes from `2_000_000` up to, but excluding, `2_001_000`; [its ledger dependency](https://github.com/midnightntwrk/midnight-indexer/blob/668ed0258ac92bb25ed02cabe6075274d8fbaac8/Cargo.toml#L122) pins rc.3. A v2 activation that bumps the runtime version therefore needs an indexer release whose dispatch accepts it. |
| Midnight.js | `41a29e756a4448dfad84ee40001f9803f86f198d` | [The HTTP proving provider](https://github.com/midnightntwrk/midnight-js/blob/41a29e756a4448dfad84ee40001f9803f86f198d/packages/http-client-proof-provider/src/http-client-proving-provider.ts#L149) forwards serialized preimages to checking/proving endpoints. It does not remove spending secrets from them. |
| Wallet | `66dd2b1d4962400d422a97854628da928a137758` | [The existing shielded-address codec](https://github.com/midnightntwrk/midnight-wallet/blob/66dd2b1d4962400d422a97854628da928a137758/packages/address-format/src/index.ts#L155) concatenates coin public key and encryption public key under its existing address type; it does not define this proposal's v2 address type. |
| Compact | `56c3b7796f78de582bee907318737077fb6e210f` | [The standard library's `coinCommitment`](https://github.com/LFDT-Minokawa/compact/blob/56c3b7796f78de582bee907318737077fb6e210f/compiler/standard-library.compact#L249) fixes the `midnight:zswap-cc[v1]` label and a 32-byte `ZswapCoinPublicKey` recipient; `mintShieldedToken` and `sendShielded` claim that commitment, and [`ownPublicKey()` and `createZswapOutput`](https://github.com/LFDT-Minokawa/compact/blob/56c3b7796f78de582bee907318737077fb6e210f/doc/api/CompactStandardLibrary/exports.md?plain=1#L902) use the same type. [The `hashToCurve` reference](https://github.com/LFDT-Minokawa/compact/blob/56c3b7796f78de582bee907318737077fb6e210f/doc/api/CompactStandardLibrary/exports.md?plain=1#L696) states that outputs are not guaranteed to be unique. |

The checked indexer cannot accept the checked node's runtime version through that dispatch path.
The local repositories therefore do not form one mutually compatible rc.5 stack even before
this protocol change. These rows locate where a runtime version bump must propagate; they are
not evidence of a tested v2 stack. No repository refs or dependency pins were changed for this
specification.

Compatibility here means retaining existing coin, tree, value, and execution semantics where
specified while introducing new ownership/proof formats. It does not mean a v2 request or record
is accepted by current binaries. V2 key derivation, address decoding, witnesses, verifier dispatch,
and scope validation need implementation; new artifacts are not supplied by these definitions.

## 3. Notation and Cryptographic Profile

The reference design uses the prime-order Jubjub subgroup. `G` is its selected generator,
`r` its subgroup order, and `p` its base-field order, matching the proof-system scalar field.
The exact group profile and generator representation must be shared by signer and circuit.

| Symbol | Meaning |
| --- | --- |
| `sk_o` | Nonzero v2 spending scalar, retained by the key holder or as MPC shares. |
| `pk_o` | Public spending point, `sk_o * G`. |
| `pk_v2` | Field-valued recipient-key representation derived from `pk_o`. |
| `esk`, `epk` | Existing encryption secret/public key pair, separate from spending authority. |
| `coin_info` | Existing shielded coin data `{nonce, type_, value}`. |
| `rc` | Existing Pedersen value-commitment blinding; not the signing nonce. |
| `H_x`, `gamma` | Unique per-coin curve base and its VRF evaluation. |
| `auth_digest` | Digest of the network and protected payment scope. |
| `k`, `z` | Secret signing nonce and fresh entropy used to derive it. |
| `R_1`, `R_2`, `e`, `s` | DLEQ nonce commitments, challenge, and response. |
| `spend_sig` | Payment-bound VRF/DLEQ evidence, not the final ZK proof. |

The main MIP writes the same DLEQ relation with `P`, `H`, `Y`, `R_G`, `R_H`, and `c`; these are
`pk_o`, `H_x`, `gamma`, `R_1`, `R_2`, and `e` here. `G` and the response `s` are common to both.

### 3.1 Encodings and functions

The following functions define the reference relations. Unresolved functions require a common
cryptographic profile; their names are not runnable APIs or existing implementation symbols.

| Function | Specified meaning | Unresolved part |
| --- | --- | --- |
| `PoseidonV2(inputs)` | A field hash frozen for this note version, independently of later generic transient-hash upgrades. | Exact parameters, framing, padding, and output selection. |
| `Domain(label)` | ASCII domain label converted into the form consumed by `PoseidonV2`. | Exact conversion. |
| `EncodeCoin(coin_info)` | Existing v1 shielded-coin binary hash layout inside SHA-256; existing field representation of the same struct inside Poseidon. | A byte-exact cross-language profile and vectors; tagged record serialization must not be assumed to equal binary hash representation. |
| `Coords(P)` | Affine `(P.x, P.y)`, in that order. Coordinates enter hashes as field elements. | Point transport encoding; coordinate vectors use canonical 32-byte little-endian field encodings. |
| `EmbedScalar(s)` | Inject the canonical integer in `[0, r)` into the base field. | None at the logical level once the field profile is fixed. |
| `EncodeDigest(D)` | Two field elements: bytes `0..16` and `16..32`, each interpreted as an unsigned little-endian 128-bit integer. | None at the logical level. |
| `EncodeField32(f)` | Low 31 little-endian bytes of the canonical field integer followed by a zero byte. | Agreement with the selected existing upgrade conversion and vectors. |
| `HashToScalar(domain, inputs)` | Hash using `PoseidonV2(Domain(domain), inputs...)`, take the canonical field integer, then reduce modulo `r`. | The underlying frozen hash profile. |
| `UniqueHashToCurve(domain, inputs)` | One uniquely determined, non-identity prime-subgroup point for each encoded input. | Exact hash-to-group algorithm, sign/root choices, cofactor treatment, and identity handling. |
| `ScalarFromUniformBytes(b64)` | Interpret 64 bytes as a little-endian integer and reduce modulo `r`. | None at the logical level once `r` is fixed. |

**Existing coin representations.** The pinned
[coin type and hash methods](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/coin-structure/src/coin.rs#L580),
[binary primitives](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/base-crypto/src/repr.rs#L66),
and [field primitives](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/transient-crypto/src/repr.rs#L165)
give the following distinct representations. `LE(bytes)` means the unsigned little-endian
integer of a byte slice; slice end indices are exclusive.

```text
BinaryHashRepr(coin_info) = nonce[0..32] || type_[0..32] || LE128(value)

FieldRepr(coin_info) = [
  nonce[31], LE(nonce[0..31]),
  type_[31], LE(type_[0..31]),
  value
]
```

The binary representation is 80 bytes, with no tag or length prefix inside this hash preimage.
The field representation is five elements, not an 80-byte value split into 128-bit digest limbs.
Each 32-byte value contributes its final byte first, then its low 31 bytes. The selected curve
field can hold a `u128` value in one element. `QualifiedInfo.mt_index` is path metadata and is
not part of `coin_info` in the commitment or nullifier. This follows declaration-order
[representation derivation](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/base-crypto-derive/src/lib.rs#L101).

The binary commitment/nullifier preimage is thus 134 bytes: 21-byte domain, 80-byte coin,
one-byte user discriminator, and 32-byte owner-derived data. This matches the native v1
preimage shape; it does not make a v2 hash equal to a v1 hash. Tagged serialization is used
separately for records and scopes and must not be substituted for these hash representations.

`EncodeField32` is a lossy 248-bit upgrade representation, not a canonical full-field encoding.
Do not use it for the address's `pk_v2` encoding. This design uses it only inside the SHA-256
commitment/nullifier preimages; the resulting persistent values, not the truncated field bytes,
appear as commitments and nullifiers on chain. The truncation limits generic inner collision
resistance to at most 124 bits (the birthday bound); the generic second-preimage bound against a
specific `pk_v2` or `vrf_out` remains 248 bits. Neither observation is a security proof for the
composition.

The 64-byte seed sampler follows the existing ledger sampler:

```text
SampleBytes(seed, 64, sep) =
    SHA-256(sep || SHA-256(LE64(0) || seed)) ||
    SHA-256(sep || SHA-256(LE64(1) || seed))
```

Hash-to-scalar reduction must use the full canonical field integer, not truncation or byte
reinterpretation. Analysis of reduction bias assumes a pseudorandom hash output; it does not
establish the security of an unspecified Poseidon suite.

### 3.2 Reference domain labels

These labels belong to the reference construction. A row marked Poseidon uses
`Domain(label)`; raw prefixes use the ASCII bytes directly. The curve-map profile must define
how its domain is incorporated. Input grammars must be domain-separated and fixed.

| Purpose | Label | Use |
| --- | --- | --- |
| Spending-key derivation | `midnight:osk[v2]` | Raw seed-sampler prefix. |
| Recipient key | `midnight:zswap-pk[v2]` | Poseidon. |
| Coin commitment | `midnight:zswap-cc[v2]` | Raw 21-byte SHA-256 prefix. |
| VRF input | `midnight:zswap-vrf[v2]` | Poseidon. |
| VRF curve map | `midnight:zswap-vrf-curve[v2]` | Curve-map profile. |
| VRF output digest | `midnight:zswap-vrf-out[v2]` | Poseidon. |
| Nullifier | `midnight:zswap-cn[v2]` | Raw 21-byte SHA-256 prefix. |
| Scoped input | `midnight:zswap-auth-input[v2]` | Raw SHA-256 prefix. |
| Scoped output | `midnight:zswap-auth-output[v2]` | Raw SHA-256 prefix. |
| Scoped transient | `midnight:zswap-auth-transient[v2]` | Raw SHA-256 prefix. |
| Payment scope | `midnight:zswap-spend-auth[v2]` | Raw SHA-256 prefix. |
| Digest binding | `midnight:zswap-auth-bind[v2]` | Poseidon. |
| Signing nonce | `midnight:zswap-spendsig-nonce[v2]` | Poseidon. |
| DLEQ challenge | `midnight:zswap-spendsig[v2]` | Poseidon. |

### 3.3 Point and scalar constraints

For the v2 relation, `pk_o`, `gamma`, `R_1`, and `R_2` MUST be canonically encoded, on-curve,
in the prime-order subgroup, and non-identity. The circuit must enforce the mathematical
constraints; host parsing alone does not establish them. Use typed,
subgroup-constrained point inputs rather than unchecked reconstruction from coordinates.

The subgroup and identity requirements on `gamma` are load-bearing. Jubjub has cofactor 8; if
`gamma + T` were accepted for a small-order point `T`, a key holder could leave `R_2`
unchanged and resample its nonce until the recomputed challenge is divisible by the order of
`T`, obtaining up to eight distinct valid `gamma` values, hence up to eight nullifiers, for
one coin. Binding `gamma` and `R_2` in the challenge only forces the retry.

`H_x` must satisfy the exact unique curve-map relation. Host determinism is insufficient if
the proof accepts another root or sign: each accepted alternative is another base on which the
key holder can honestly evaluate `gamma`, and therefore another nullifier for the same coin.
Reusing a host curve-map implementation is not evidence that its circuit relation enforces
uniqueness; the Compact reference for `hashToCurve`, at the revision pinned in Section 2.1,
does not itself guarantee output uniqueness. The existing gadget recorded in Section 2 is a
candidate: its in-circuit map constrains the square root, its sign, and cofactor clearing, but
adopting it requires fixing how the `midnight:zswap-vrf-curve[v2]` domain and `vrf_input`
enter the sponge, adding the non-identity check it does not perform, settling its exceptional
inputs, and reviewing it for this relation. The selected profile must define and constrain
one valid output; this attachment does not provide a completed canonical gadget.
`HashToScalar(input) * G` is not a suitable substitute: its publicly known discrete logarithm
would allow computation of `gamma` from `pk_o` without the secret.

The DLEQ response must satisfy `0 <= s < r`. A canonical `s = 0` is valid if the equations and
all other constraints hold. Zero spending keys and zero signing nonces are rejected by the
reference key-generation and nonce rules. Scalar assignment by itself is not evidence that the
compiled circuit enforces the range.

## 4. Keys, Addresses, and Note Creation

```text
sk_o  = ScalarFromUniformBytes(SampleBytes(seed, 64, "midnight:osk[v2]"))
pk_o  = sk_o * G
pk_v2 = PoseidonV2(Domain("midnight:zswap-pk[v2]"), pk_o.x, pk_o.y)
address = (pk_v2, epk)
```

The seed is the existing Zswap HD role seed, not a newly specified wallet root derivation.
The separator distinguishes v2 spending-key derivation from the existing v1 spending and
encryption derivations. A zero result is a derivation failure, not permission to substitute an
unspecified retry counter or fallback derivation. DKG can instead establish a
shared nonzero key, subject to the MPC requirements in Section 12.

The reference address uses a distinct Bech32m address type containing exactly the canonical
32-byte little-endian encoding of `pk_v2`, followed by the existing serialized 32-byte `epk`.
Decoders check length, canonical field representation, and the address type. The type
discriminator and network-specific HRPs are unresolved. Unknown versions must fail closed.

The reference coin commitment is:

```text
coin_info   = { nonce, type_, value }
coin_com_v2 = SHA-256("midnight:zswap-cc[v2]" || EncodeCoin(coin_info) || 0x01
                      || EncodeField32(pk_v2))
```

The separator, coin encoding, user discriminator `0x01`, and 32-byte owner representation
retain the v1 preimage shape, with the v2 domain and recipient representation substituted.
The commitment enters the existing depth-32 tree. The existing tree's leaf conversion and
internal hashing remain unchanged; this is not a second v2 tree.

Output encryption remains the existing encryption of `coin_info` to `epk`, without an added
recipient key or note-version field in the plaintext. A wallet holding both receiving keys
recomputes both commitment candidates and compares them with the actual output. Exactly one
match identifies its coin and version. No match means not recognized by those keys; two matches
indicate an error requiring investigation, not permission to choose either version. A v2-only
viewing wallet checks only its v2 candidate. Decryption without a commitment match must not
credit funds.

The SHA-256 outer representation avoids a trivial field-format distinguisher. It does not by
itself prove anonymity, unlinkability, or indistinguishability of complete transactions.

## 5. VRF Nullifiers and Viewing

```text
vrf_input = PoseidonV2(Domain("midnight:zswap-vrf[v2]"), EncodeCoin(coin_info), pk_v2)
H_x       = UniqueHashToCurve("midnight:zswap-vrf-curve[v2]", vrf_input)
gamma     = sk_o * H_x
vrf_out   = PoseidonV2(Domain("midnight:zswap-vrf-out[v2]"), gamma.x, gamma.y)
nullifier = SHA-256("midnight:zswap-cn[v2]" || EncodeCoin(coin_info) || 0x01
                    || EncodeField32(vrf_out))
```

The encoded coin fields are flattened according to the frozen field profile. The display above
does not imply that `EncodeCoin` is a single field element.

The payment digest, signing randomness, Merkle path, and MPC participant subset MUST NOT affect
the nullifier for an unchanged coin and key. Fresh payment evidence changes the authorization,
not the VRF output. Identical coin data sent to the same recipient repeats the commitment and
nullifier; senders therefore use fresh coin nonces.

The key holder derives `pk_v2` from its own key, checks a supplied recipient representation,
and derives or independently recomputes `H_x` before using `sk_o`. It must not evaluate the
secret against arbitrary points chosen by the host. Each MPC participant performs the same
base validation. Nonce generation and signing remain within the key-holding boundary.

Given `gamma` and the coin data, the host can compute the nullifier. Possession of `gamma`
alone does not permit a spend: Section 7's digest-bound DLEQ evidence is also required.

The incoming viewing key is `(esk, pk_v2)`. It can recognize incoming v2 notes but cannot
derive their VRF outputs without the key holder. The host can obtain `gamma` once per received
note and maintain a nullifier list. That list plus incoming viewing data supports spend
detection. Missing entries mean incomplete spend visibility, not proof that all receipts remain
unspent. A transport or authenticated batching protocol for this list is outside this specification.

## 6. Payment Scope and Digest

These are logical reference schemas, not deployed ledger structs or completed binary encodings.

```text
AuthScope {
  segment: u16
  inputs:  [InputRef]
  outputs: [(u16, OutputRef)]
  intents: [(u16, IntentHash)]
}

auth_digest = SHA-256("midnight:zswap-spend-auth[v2]" ||
                      CanonicalSerialize(network_id) || CanonicalSerialize(scope))

ScopedInput     { circuit_kind, nullifier, value_commitment, contract_address, merkle_tree_root }
ScopedOutput    { circuit_kind, coin_commitment, value_commitment, contract_address, ciphertext }
ScopedTransient { input_circuit_kind, output_circuit_kind, nullifier, coin_commitment,
                  input_value_commitment, output_value_commitment, contract_address, ciphertext }

InputDigest     = SHA-256("midnight:zswap-auth-input[v2]"     || CanonicalSerialize(ScopedInput))
OutputDigest    = SHA-256("midnight:zswap-auth-output[v2]"    || CanonicalSerialize(ScopedOutput))
TransientDigest = SHA-256("midnight:zswap-auth-transient[v2]" || CanonicalSerialize(ScopedTransient))

InputRef  = Input(InputDigest) | Transient(TransientDigest)
OutputRef = Output(OutputDigest) | Transient(TransientDigest)
```

The scope's segment identifies its carrying offer; segment 0 is the guaranteed offer. Input
references use that segment. Output and intent references carry explicit segments so a scope can
name records its builder placed in other offers or intents of the same transaction, such as an
intent it created (Section 6.1). Referencing another participant's records is permitted but not
required; it presumes those records and their segments are fixed before the digest is computed.
Intent references use segments `>= 1`; the existing ledger
[rejects an intent declared at segment 0](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/verify.rs#L687)
as malformed. Reference identity includes segment.

`CanonicalSerialize` is the ledger's tagged binary serialization, not JSON, CBOR, or SCALE.
Use the existing string encoding of `network_id`, including length framing.
Tags, discriminant values, nested framing, and byte-exact vectors for the new types are not
provided. Field names above correspond conceptually to proof-erased records; they are not
an assertion that native field names are identical. `IntentHash` is the persistent hash of
the existing per-segment intent signing envelope, not the final transaction hash.

`ScopedInput` mirrors the proof-erased input record, so it names the Merkle root the input was
proven against. The root adds no authorization over effects: it is public in the record and in
the proof statement, and the coin, its committed value, and the referenced outputs are fixed by
the other fields and references. Naming it ties the evidence to that root's retention window,
`global_ttl` after the last block whose tree had it (Section 2). A path refreshed after the root
ages out is therefore a changed protected item under Section 6.1 and needs new evidence; the
benefit is a ledger-enforced bound on evidence reuse that needs no intent reference.

### 6.1 Construction and coverage

1. Every scope has at least one input reference. Scope lists and nested lists are canonically
   sorted by their serialization and contain no duplicates.
2. Each compatibility input, including a transient input, is covered exactly once by a scope
   of its carrying offer. Output and intent references may be shared between different scopes,
   but must not repeat inside one scope.
3. A transient is one public record. Its `TransientDigest` binds both circuit kinds, both value
   commitments, the nullifier, commitment, and ciphertext, so one digest serves as both its
   input reference and its output reference. Listing `Transient(TransientDigest)` under
   `inputs` is the coverage required by rule 2 and already binds the transient's output side;
   the covering scope need not repeat it under `outputs`. A scope MAY list
   `(segment, Transient(TransientDigest))` under `outputs` to reference a transient it does not
   cover; as for any output reference, the stated segment is the transient's carrying offer
   (Section 10, rule 4).
4. A wallet MUST list all compatibility inputs it spends, every transient it creates (as an
   input reference), all outputs it creates, and every intent it created. One scope per
   participating wallet per offer is the normal case. Several key holders may authorize the
   same scope.
5. The wallet finalizes coin data, recipient and change outputs, blinding, ciphertexts,
   segments, and referenced intents before computing the digest and requesting local evidence.
   Changing a protected item requires rebuilding the affected scopes and regenerating evidence.

The ledger checks coverage of public records, not whether a wallet omitted an intended output.
That completeness is an authorizing-wallet or custody-policy responsibility. A scope authorizes
inclusion of exact records at exact segments; it does not make a fallible effect guaranteed or
make effects in different segments atomic.

### 6.2 Merge semantics

The offer's proposed `auth_scopes` field is merged by canonical union alongside coins and
deltas. A merge must reject overlapping scoped input identities and duplicate input/transient
nullifiers even if proof bytes differ. Existing full-record disjointness is not sufficient.
The merged transaction still passes the complete transaction-level checks in Section 10.

Proof bytes and `spend_sig` do not enter scope records, avoiding a dependency on the proof being
constructed. Offer merging may add compatible records, but must not remove or mutate records
already protected by an input's evidence. This preserves merge-based flows, not arbitrary
post-authorization restructuring of a payment.

## 7. Local Payment-bound DLEQ Proof

Per v2 input, after the scope is fixed:

```text
z   = 32 fresh random bytes from the key-holding component
k   = HashToScalar("midnight:zswap-spendsig-nonce[v2]",
                   EmbedScalar(sk_o), vrf_input, EncodeDigest(auth_digest), EncodeDigest(z))
R_1 = k * G
R_2 = k * H_x
e   = HashToScalar("midnight:zswap-spendsig[v2]",
                   R_1.x, R_1.y, pk_o.x, pk_o.y, R_2.x, R_2.y,
                   gamma.x, gamma.y, H_x.x, H_x.y,
                   EncodeDigest(auth_digest))
s   = k + e * sk_o mod r
spend_sig = { pk_o, gamma, R_1, R_2, s }
```

The challenge has 12 field inputs after the domain: ten coordinates and two digest limbs,
in the order above. The circuit recomputes the challenge and the unique coin base, then checks:

```text
s * G   == R_1 + e * pk_o
s * H_x == R_2 + e * gamma
```

The same response and challenge bind ownership and VRF evaluation to one secret. A single
Schnorr equation would not constrain an unrelated `gamma`. A DLEQ proof that omitted the
payment digest would establish VRF correctness without limiting its use to the intended scope.

This is one combined authorization/correctness proof, created locally as part of Send. It is
not an additional on-chain approval or the final ZK proof. The remote prover knows this evidence,
not the spending key behind it.

### Nonce requirements

- `z` is fresh entropy generated inside the key holder, never supplied or observed by the host.
  The hedged derivation includes the key, coin input, and payment digest.
- A derived zero `k` triggers new entropy and retry. Nonce collisions must be negligible;
  hashing different inputs is not an injectivity guarantee.
- Signers must not deliberately reuse a nonce across coins or digests. `k` and `z` are erased
  after use and kept out of proving requests, logs, traces, and crash reports.
- Threshold signers do not compute this formula over a reconstructed `sk_o`. They require a
  separately analyzed distributed nonce/signing protocol producing the same aggregate relation.

The single-key nonce formula is a reference choice defined here. The main MIP
leaves secure nonce derivation to implementation design; it does not select a threshold protocol.

## 8. Compatibility Circuits and Public Statements

The reference `spend_compat` circuit has private v1 and v2 ownership branches. The logical
selector `note_version` is 1 or 2; its boolean representation and inactive-branch encoding are
part of the unresolved circuit ABI. Both branches use one verifier identity, proof shape,
and public statement shape. Their selection must not be visible through serialized metadata.

### 8.1 Spend statement and witness

```text
Logical public effects:
  coin_com_root, nullifier, segment, value_com, EncodeDigest(auth_digest)

Logical private inputs:
  note_version, owner_witness, merkle_path, auth_binding, coin_info, rc
```

The first four public values describe existing transcript effects: root check, nullifier
insertion, segment read, and value-commitment write. They are not a literal flat verifier ABI.
The reference adds a read-only transcript cell for the two digest limbs and reads it in both
branches. Its position, operation encoding, and compiled statement are not specified here.

For v1, `owner_witness` supplies the original v1 user secret. For v2 it supplies the DLEQ
points and response. Unused branch slots must not include a real v1 key in a v2 request.
They must also satisfy any unconditional typing constraints. Preserving the existing
`coin_info, rc` witness suffix is recommended for compatibility with existing extraction helpers,
not asserted as a measured compiled layout.

### 8.2 Branch-independent digest binding

The same proof must fail when either public digest limb changes, for either branch. The
reference adds a private `auth_binding` value with the relation:

```text
auth_binding == PoseidonV2(Domain("midnight:zswap-auth-bind[v2]"), EncodeDigest(auth_digest))
```

This value is computable by the host or prover; it is not another secret or authorization.
For v1, a holder of the v1 key can create a new proof for a new digest; the property above
concerns modifying the statement of an existing proof, not removing the v1 secret requirement.

A cheaper branch-independent constraint MAY replace it if it enforces the same binding.
Whether the compiler preserves the required constraints is unverified without the actual
compiled circuit and statement-mutation checks. The formula alone is not that evidence.

### 8.3 Spend constraints

| ID | Required relation |
| --- | --- |
| S1 | Read and bind both public digest limbs independently of branch selection. |
| S2 | Select exactly one complete v1 or v2 relation without revealing the private version. |
| S3 | V1: enforce the existing user-key, coin-commitment, and nullifier relations. |
| S4 | V2: enforce canonical point/scalar handling, subgroup checks, identity rejection, and `s < r`. |
| S5 | V2: recompute `pk_v2`, `coin_com_v2`, `vrf_input`, `H_x`, and `vrf_out`. |
| S6 | V2: recompute the digest-bound challenge and verify both DLEQ equations. |
| S7 | V2: recompute and match the public nullifier. |
| S8 | Match the selected commitment to the path leaf and verify the path against the required root. |
| S9 | Recompute the existing value commitment from value, asset type, segment, and `rc`. |

S9 retains the existing value-base construction over asset type and segment; it does not
substitute the v2 VRF curve map. The reference suggests sharing SHA-256 work across branches
using selected preimage fields, but no constraint count or cost improvement is established.

### 8.4 Outputs and transients

`output_compat` proves the selected v1 user-recipient relation or the v2 commitment relation,
retaining the existing value commitment, segment, and ciphertext binding. A sender needs the
recipient representation, not its spending key. Binding ciphertext bytes does not by itself
prove that they encrypt the committed plaintext correctly; ownership detection still checks
the decrypted commitment.

A transient carries an input and an output proof. Its input root is reconstructed using the
existing depth-32 tree with that output's commitment at index 0. The input path must open the
same commitment, and each value commitment uses the carrying segment with its own randomness.
This does not assert that the current code supplies a v2 user-transient constructor.

Contract-owned coins use the existing relation. They do not become v2 user-owned coins merely
because a transaction also includes a v2 user spend.

Contract-created user outputs are v1 notes at the checked revisions. A contract computes its
recipient commitment in-circuit with the standard library's `coinCommitment`, which fixes the
`midnight:zswap-cc[v1]` label and a 32-byte `ZswapCoinPublicKey`, and claims it through
`kernel.claimZswapCoinSpend`; the
[ledger requires](https://github.com/midnightntwrk/midnight-ledger/blob/67f9f97246339539af0091c4d8c6bc5ab236a191/ledger/src/verify.rs#L1609)
each claimed commitment to equal an output commitment in the same segment. No contract compiled
against that library can produce or claim `coin_com_v2`. A dapp or wallet MUST NOT place
`EncodeField32(pk_v2)` in the `ZswapCoinPublicKey` slot as a workaround: the ledger accepts the
resulting v1-relation output, but opening it under v1 requires a preimage of the v1 key
derivation and the v2 branch recomputes the `[v2]` label, so the coin is unspendable. Paying a
v2 recipient from a contract needs the recipient extension listed in Section 13 and redeployment
of the contract. The output proof for such a coin is built by the wallet or dapp with the existing
output constructor, which emits only legacy proofs; after legacy-user-proof retirement (Section
10, rule 7) that constructor must emit compatibility proofs for v1 user recipients as well, a
wallet-side change that leaves the contract's claim unchanged. Until then a user who receives
from contracts keeps a v1 receiving address.

## 9. Circuit Kinds and Serialization

The reference introduces `ZswapCircuitKind` with symbolic cases `Legacy` and `CompatV2`:

- Each proof-bearing input and output has a circuit kind.
- A transient has separate input and output kinds, which must survive conversion to its
  input/output views.
- `Legacy` selects the existing static verifier material and backend; `CompatV2` selects the
  matching new compatibility material. The kind does not identify the private note branch.
- This is not the existing contract-call proof-version enum.

New tagged versions are required for affected records and enclosing transactions.
Historical formats decode as legacy records with no scopes. Unknown kinds or versions fail
closed. This specifies required decoder behavior, not an existing backward-decoder implementation.

Numeric discriminants, tags, serialized proof type and size, verifier identifiers, and artifact
hashes are unresolved. Names such as `spend_compat` are reference identifiers, not published
resolver paths. Legacy size or public-input counts must not be assumed to describe new artifacts.

## 10. Transaction-level Validation

For each guaranteed or fallible offer, after all offers and intents are available:

1. Scope lists and nested lists have canonical order and no duplicates; each scope has an input.
2. A scope's segment equals its carrying offer's segment.
3. Every compatibility input or transient-input record appears in exactly one scope of that
   offer, and every listed input reference matches exactly one public record. Legacy input
   proofs appear in no scope.
4. Each output reference matches exactly one output or transient in the stated segment.
5. Each intent reference matches the transaction's intent hash for the stated segment.
6. The validator recomputes `auth_digest` from the transaction's network and scope, and supplies
   it as the public digest for each listed input proof.
7. Scoped inputs use `CompatV2`. After legacy-user-proof retirement, user-owned inputs and
   user-recipient outputs must use compatibility verification.
8. Public `contract_address = Some(_)` requires legacy verification; compatibility records have
   `None`. A contract transient uses legacy verification for both proofs.

Input/transient double-spend rejection, root validity, balancing, binding commitments, TTL,
replay protection, and actual segment execution remain ledger responsibilities. Passing DLEQ
verification alone does not satisfy them. Scope references bind inclusion, not execution success.

Scope validation must be bounded by transaction limits and run in `O(N log N)` or better in
the number of records plus nested references. No concrete limit, fee, or benchmark is supplied.
Proof size, public-input count, and scope-validation work must be accounted for; unchanged fee
rules do not establish identical fees.

## 11. Proving Boundary and Operational Behavior

| Boundary | Permitted data | Excluded data |
| --- | --- | --- |
| Key holder to wallet | Public spending key, per-note VRF outputs, payment-bound DLEQ evidence. | Spending key and signing nonce/entropy. |
| Wallet to v2 prover | Selector, DLEQ evidence, coin data, path, `rc`, digest transcript and its binding witness. | Spending key, seed, key shares, actual v1 secret, `k`, and `z`. |
| Transaction to validator | Final proofs, public records, circuit kinds, and scopes. | Private note selector, private DLEQ witness, and coin openings. |

The v2 prohibition covers serialized request bodies, including batches and both checking and
proving endpoints where used. Checking property names alone is insufficient. Coin nonces and
Pedersen randomness are legitimate private proof inputs, unlike secret signing nonces.

The wallet verifies returned proofs before broadcast. It may retry an unchanged request with
another prover while its scope and ledger prerequisites remain valid, including retention of the
Merkle root named by each scoped input. Once that root has aged out, refreshing the path changes
the scope and requires new evidence (Section 6). Changing a protected record, network, or segment
requires new evidence. Refusal by a signer or prover does not justify exporting the spending key.

Issued evidence neither reserves a coin nor creates revocable on-chain permission. Switching
provers or refreshing shares under the same key does not cancel it. Required expiry must be
enforced by existing ledger validity mechanisms and bound through the relevant referenced
intent, not enforced only by a wallet timer. Checking a transaction's TTL without binding the
owner's evidence to it is not equivalent to authorizing that expiry.
Because each scoped input names its Merkle root, evidence also expires when that root leaves the
ledger's retention window. That bound is coarse and network-configured; it does not replace an
intent TTL where a tighter or payment-specific expiry is required.

Wallets reconcile actual per-segment outcomes and canonical chain state. Proof completion,
submission, and confirmation are distinct. Rebuilding after a state change can require new
paths, scopes, and evidence, but does not change the nullifier of an unchanged coin and key.

## 12. Migration, MPC, and Security Limits

Existing v1 notes retain their original commitment and nullifier relations. They cannot be
spent through the v2 branch by replacing the old key with a VRF output. To avoid remote exposure
of the old key, migration uses a locally proven v1 spend and a new v2 output. The v1 branch of
the compatibility circuit still requires the v1 key.

The reference uses a coordinated activation height or epoch. Before activation, compatibility
proofs and scope-bearing offers are rejected. Legacy user proofs remain accepted during the
transition, then retire at a network-selected height while v1 notes remain spendable through
compatibility proofs. Contract-owned coins retain legacy relations. No concrete activation or
retirement value is supplied, and the main MIP leaves scheduling to the network upgrade.

Both compatibility circuits are intended to fit the published, provenance-checked universal
parameters without a new ceremony. Their fit, performance, equal branch-visible proof shape,
and correct verifier dispatch require actual artifacts; no such measurements are claimed here.

For MPC, public-key and VRF shares combine with the reconstruction coefficients of the signing
subset. The full nullifier is independent of that subset. DKG, partial-response validation,
coordinated nonce commitments, and the abort/retry protocol remain unspecified. One-base
FROST is background, not a complete two-base signing protocol. A reviewed construction must
avoid reconstructing both the full key and full signing nonce.

A deterministic seed-derived key uses the reference derivation in Section 4. A DKG-created
key needs separate share recovery; it is not recoverable from an unrelated wallet seed.
Recovery and share refresh preserve the public key and nonce safety. A genuinely new key
requires transferring coins while the old key can still authorize. Viewing data cannot recover
lost spending authority.

A compromised prover or request observer sees sensitive coin information and can link requests,
refuse service, or submit an authorized scope. Key isolation does not hide that data. A signer
that blindly accepts a hostile wallet's digest also does not protect against host compromise.
That stronger guarantee requires independent reconstruction of the protected records and digest,
and trusted user approval or independently configured policy over the actual payment.

Public scopes reveal contribution structure. Compatibility proofs hide their private branch,
not all historical or application-level distinctions between notes. The hash composition, DLEQ,
curve-map uniqueness, authorization binding, and MPC protocol need their own cryptographic
analysis. Kachina's model does not certify these additions or external witness disclosure.

Dust, contract ownership, and the separate rewards-claim path are unchanged. The dormant claim
relation is not reactivated. In particular, delegated Dust and v1 proving remain secret-bearing
and must not be described as key-free because v2 shielded proving excludes its spending key.

## 13. Unresolved Protocol Choices and Evidence

This section records unresolved elements of the protocol, not a rollout checklist.
The attachment is detailed at the relation level but is not a complete interoperable wire
specification until these choices have a shared definition. They must not be filled independently
by each implementation.

| Item | What this specification defines | What remains unresolved |
| --- | --- | --- |
| Frozen field hash | The `PoseidonV2` abstraction and domain-separated uses. | Parameters, domain conversion, framing, padding, and full vectors. |
| VRF curve map | Uniqueness, subgroup, identity, and hash-to-group requirements; an existing in-circuit gadget as candidate (Section 3.3). | Adoption of that candidate or another algorithm; exact algorithm, constants, root/sign conventions, domain and input framing, non-identity check, exceptional inputs, and review of the gadget for this relation. |
| Canonical encodings | Native coin byte/field mapping and logical scalar/digest representations. | Complete frozen profile and vectors, including point transport and agreement across languages. |
| Address | Bech32m with full-field `pk_v2` followed by serialized `epk`. | Address discriminator and network HRPs. |
| Scope serialization | Logical schemas and tagged serialization choice. | Tags, discriminants, framing, and byte-exact examples. |
| Scope enforcement | Coverage rules, merge union, and rejection of overlapping public identities. | Concrete algorithms and resource limits; consistency under all supported segment/transient combinations. |
| Circuit ABI | Logical witnesses, public effects, and branch-independent digest binding. | Transcript cell encoding, selector and dummy layout, compiled constraints, and statement vectors. |
| Proof artifacts | Legacy/compatibility dispatch and common branch shape. | Actual circuits, matching keys/backend, size/cost measurements, artifact hashes, and parameter fit. |
| Migration | Activation plus possible retirement of legacy user proof formats. | Network scheduling and actual historical decoder implementations. |
| Contract-created user outputs | Contract-created user outputs keep the existing v1 relation; the ledger's claimed-commitment check is unchanged. | A v2 user-recipient arm in the Compact standard library (`coinCommitment` and output creation) and a ledger output constructor that emits `coin_com_v2` under the `CompatV2` output kind, and that contract-created v1 user outputs also use after retirement. |
| Threshold custody | Aggregate public-key/VRF/DLEQ relations. | A complete, analyzed DKG/signing/recovery protocol and device capabilities. |
| Security evidence | Required properties, reference formulas, and algebraic completeness. | Independent analysis, native/circuit differential results, and verification of final serialized transactions. |

No test vectors, benchmark results, compiled circuits, tags, or deployment status are implied
by these definitions. Document checks establish consistency, not the cryptographic soundness or
implementability of the design.

## References

- [mips/mip-xxxx.md](../mip-xxxx.md): high-level requirements and design latitude.
- [Pinned ledger source](https://github.com/midnightntwrk/midnight-ledger/tree/67f9f97246339539af0091c4d8c6bc5ab236a191): existing implementation baseline.
- Chaum and Pedersen, "Wallet Databases with Observers," CRYPTO 1992: DLEQ background.
- [RFC 9381](https://www.rfc-editor.org/rfc/rfc9381): VRF definitions, not wire compatibility
  for this construction.
- [RFC 9591](https://www.rfc-editor.org/rfc/rfc9591): FROST background, not the required complete
  two-base threshold protocol.
- [Nightpaper: A litepaper introducing Midnight](https://45047878.fs1.hubspotusercontent-na1.net/hubfs/45047878/Midnight%20litepaper.pdf),
  architecture, pp. 10-11.
- Kerber, Kiayias, and Kohlweiss,
  [Kachina - Foundations of Private Smart Contracts](https://eprint.iacr.org/2020/543.pdf),
  revision 4, 2021. Sections 4-5 and Appendix C.4 describe transcripts and private payments;
  Section 4.2 and Appendix I delimit composition and multiparty trust treatment.
