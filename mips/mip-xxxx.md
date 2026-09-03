---
MIP: X
Title: Signature-Authorized Shielded Spends with VRF Nullifiers
Authors:
  - Ricardo Rius (riusricardo)
Status: Draft
Category: Core
Created: 2026-08-28
Requires: none
Replaces: none
MPS: MPS-0035
Related-MPS: MPS-0024, MPS-0016 # not the primary problem statements, but addressed by this design
Related-MIP: MIP-0005, MIP-0006 # not dependencies; deployed merge semantics this design preserves
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

> **Scope.** This document is the complete implementation specification for shielded note
> version 2: every normative formula, encoding, constraint, validation rule, and acceptance gate
> needed to implement and activate the protocol change is defined here. Behavior is specified by
> observable properties; implementers are free to choose any internal structure that satisfies
> them.

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT", and "MAY" are to be
interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

## 1. Summary

Today the Zswap spend circuit witnesses the coin secret key, so whoever produces the proof holds
full spend authority over every coin under that key. This MIP defines shielded note **version 2**,
whose single secret `sk_o` (a Jubjub scalar) never enters a proving payload. The nullifier is a
VRF output `gamma = sk_o * H_x` over a per-coin base point, and the spend authorization is a
Chaum-Pedersen DLEQ proof, produced locally by the key holder and bound to an authorization digest
over the items the wallet built. The circuit verifies that proof against the public key committed
in the note. Proving becomes delegable to untrusted services, the key fits in a secure element,
and because every operation is linear in `sk_o`, a threshold group can hold and use the key as
shares that are never reconstructed.

Unchanged by design: the commitment tree, nullifier set, value commitments, balancing, fees,
transaction lifecycle, replay protection, the encryption key pair, ciphertext format, and trial
decryption. V2 commitments and nullifiers keep the v1 SHA-256 outer shape so they are
indistinguishable from v1 values on chain, and compatibility circuits with a **private**
note-version selector keep v1 and v2 notes in one anonymity set. Existing v1 notes remain
spendable. Activation requires a coordinated network upgrade.

## 2. Roles and data flow

| Party | Holds | Receives | Never sees |
| --- | --- | --- | --- |
| Key holder (secure element or threshold group) | `sk_o` | `coin_info`, `pk_v2`, `auth_digest` | nothing withheld; it is the trust root |
| Wallet host | `esk`, `pk_v2`, notes, nullifier list | `pk_o` once; `gamma` per note; `spend_sig` per spend | `sk_o` |
| Prover | nothing | `coin_info`, Merkle path, `rc`, `pk_o`, `gamma`, `spend_sig`, `auth_digest`, `auth_binding` | `sk_o`, nonces `k`/`z` |
| Validator and chain | ledger state | proof, nullifier, commitments, value commitments, scopes | `sk_o`, `pk_o`, `gamma`, `spend_sig` |
| Sender | nothing | address `(pk_v2, epk)` | everything else |
| Auditor (if granted) | `esk`, `pk_v2`, nullifier list | nothing further | `sk_o` |

Flow: **(1) Setup** — key holder derives `sk_o` (or a DKG produces shares) and exports `pk_o`
once; the host computes `pk_v2` and publishes `(pk_v2, epk)`. **(2) Receive** — a sender builds
`coin_com_v2`, encrypts `coin_info` to `epk`, proves the output with `output_compat`; the host
decrypts, matches the v1 and v2 commitment candidates against the on-chain commitment, stores the
note with its version, and asks the key holder once for the note's `gamma`. **(3) Spend** — the
host assembles the offer, lists its own items in an `AuthScope`, computes `auth_digest`; the key
holder derives `H_x` itself, evaluates `gamma`, returns `spend_sig` bound to the digest.
**(4) Prove** — any untrusted prover receives openings plus `spend_sig` (never a secret) and
produces the `spend_compat` proof. **(5) Validate** — the validator checks the scope against the
transaction, recomputes `auth_digest`, verifies the proof with the digest as a public input, and
applies the existing nullifier-uniqueness, balance, and replay rules. Merging offers remains
possible before or after; nothing the wallet listed can be removed or altered.

## 3. Cryptographic profile

Curve: the prime-order Jubjub subgroup with generator `G`, subgroup order `r`, base field of
order `p` (the proof-system scalar field). Hashes: a version-frozen Poseidon suite `PoseidonV2`,
and SHA-256 as the existing persistent hash. No point is ever accepted merely because its affine
coordinates satisfy the curve equation.

A conforming implementation MUST publish byte-exact definitions of all of the following before
the MIP leaves Draft, and freeze them for the lifetime of note version 2:

- **`PoseidonV2(inputs)`** — exact field, state width, rate/capacity, S-box, round counts, round
  constants, MDS matrix, input framing/padding, and output element. The parameters MAY equal the
  chain's current upgradeable transient hash at activation but MUST remain frozen for v2 values
  even if the generic transient hash changes later.
- **`Domain(label)`** — the conversion of each ASCII domain label to the field form consumed by
  `PoseidonV2`.
- **`EncodeCoin(coin_info)`** — canonical encoding of `{nonce, type_, value}`. Inside a SHA-256
  preimage it is the existing v1 shielded-coin byte layout (so v1 and v2 preimages have identical
  shape); inside a `PoseidonV2` input it is the existing field representation of the same struct.
- **`Coords(P)`** — the affine coordinates `(P.x, P.y)`, in that order, as two field elements;
  the only form in which a point enters a hash. Test vectors record each coordinate as its
  canonical 32-byte little-endian field encoding.
- **`EmbedScalar(s)`** — the canonical integer of a Jubjub scalar in `[0, r)` injected into the
  base field as one element.
- **`EncodeDigest(D)`** — a 32-byte value as two field elements: bytes `0..16` as an unsigned
  128-bit little-endian integer, then bytes `16..32` the same way.
- **`EncodeField32(f)`** — the canonical 32-byte encoding of a field element used inside SHA-256
  preimages: the low 31 little-endian bytes of the canonical integer followed by one zero byte
  (retains 248 bits). It MUST match the chain's existing field-to-bytes upgrade conversion, and
  MUST NOT appear on-chain by itself — every consensus-visible v2 value is a SHA-256 output.
- **`HashToScalar(domain, inputs)`** — `PoseidonV2(Domain(domain), inputs...)`, taken as the
  canonical integer of the output element and reduced modulo `r`. This is the ONLY permitted map
  from a hash output to a scalar; truncation or byte reinterpretation is forbidden. Because
  `p = 8r + d` with `d < 2^126`, the reduction bias is below `2^-129`; no rejection sampling is
  needed for challenges or nonces, under a pseudorandom-output assumption on `PoseidonV2`.
- **`UniqueHashToCurve(domain, inputs)`** — a byte-exact hash-to-Jubjub relation admitting
  **exactly one** valid output point per encoded input: fixed sponge/absorption/squeeze sequence,
  map-to-curve algorithm and constants, canonical sign/root choices, point combination, cofactor
  clearing, and identity handling (reject, or a deterministic uniqueness-preserving retry). The
  chain's existing hash-to-curve relation MUST NOT be used unchanged: its contract permits one
  input to prove correct against multiple outputs, which would let a key holder mint several
  nullifiers for one coin. Hardening it (or adding a canonical gadget) is a Phase 0 gate.
- **`SampleBytes(seed, 64, sep)`** — the ledger's existing seed sampler: for rounds 0 and 1,
  `SHA-256(sep || SHA-256(LE64(round) || seed))`, concatenating the two 32-byte outputs.
- **`ScalarFromUniformBytes(b64)`** — interpret 64 bytes as a little-endian integer, reduce
  mod `r` (statistical distance from uniform ≈ `2^-260`).

**Reserved domain labels** (frozen once the profile is accepted):

| Purpose | Label | Enters |
| --- | --- | --- |
| spend-key derivation | `midnight:osk[v2]` | SHA-256 prefix (like the existing csk/esk derivations) |
| recipient key hash | `midnight:zswap-pk[v2]` | `Domain()` → `PoseidonV2` |
| coin commitment | `midnight:zswap-cc[v2]` | 21-byte raw ASCII SHA-256 `sep` (as v1 does) |
| VRF input | `midnight:zswap-vrf[v2]` | `Domain()` → `PoseidonV2` |
| VRF hash-to-curve | `midnight:zswap-vrf-curve[v2]` | per the `UniqueHashToCurve` profile |
| VRF output digest | `midnight:zswap-vrf-out[v2]` | `Domain()` → `PoseidonV2` |
| nullifier | `midnight:zswap-cn[v2]` | 21-byte raw ASCII SHA-256 `sep` |
| scoped input digest | `midnight:zswap-auth-input[v2]` | raw ASCII SHA-256 prefix |
| scoped output digest | `midnight:zswap-auth-output[v2]` | raw ASCII SHA-256 prefix |
| scoped transient digest | `midnight:zswap-auth-transient[v2]` | raw ASCII SHA-256 prefix |
| authorization digest | `midnight:zswap-spend-auth[v2]` | raw ASCII SHA-256 prefix |
| digest binding | `midnight:zswap-auth-bind[v2]` | `Domain()` → `PoseidonV2` |
| authorization nonce | `midnight:zswap-spendsig-nonce[v2]` | `Domain()` → `PoseidonV2` |
| DLEQ challenge | `midnight:zswap-spendsig[v2]` | `Domain()` → `PoseidonV2` |

No other protocol object may reuse any of these domains with a different input grammar; the
input arity of every `PoseidonV2` use is fixed by the profile.

**Point and scalar validation.** Every externally supplied Jubjub point (`pk_o`, `gamma`, `R_1`,
`R_2`) MUST use a canonical encoding and MUST be constrained in-circuit to be on-curve, in the
prime-order subgroup, and not the identity. `H_x` MUST be the unique, cofactor-cleared,
non-identity output of `UniqueHashToCurve`. Witnessed scalars MUST be canonical values strictly
below `r`; the circuit MUST range-constrain the witnessed response `s < r` explicitly (typed
scalar assignment alone does not guarantee it). Witnessed points MUST enter as typed,
subgroup-constrained circuit inputs; reconstructing points from raw coordinates with an
unvalidated constructor is forbidden. The generated circuit artifact MUST be inspected to confirm
that typed-input decoding, subgroup constraints, the `s < r` check, and identity rejection are
actually present, rather than trusting host-language input parsing.

## 4. Keys and addresses

```text
sk_o  = ScalarFromUniformBytes(SampleBytes(seed, 64, "midnight:osk[v2]"))    (or DKG shares)
pk_o  = sk_o * G
pk_v2 = PoseidonV2(Domain("midnight:zswap-pk[v2]"), pk_o.x, pk_o.y)
address = (pk_v2, epk)          epk = existing encryption public key, unchanged
```

`sk_o` derives from the existing Zswap HD role seed under its own separator, so it is independent
of every v1 coin secret key and encryption key from the same seed; no new HD role or backup
material. A zero result MUST be treated as a derivation failure. The key holder exports `pk_o`
once; its only other outputs, ever, are per-note `gamma` values and per-spend `spend_sig`
authorizations.

A v2 address is a new Bech32m address type (distinct from the existing shielded type, which
encodes the 32-byte v1 coin public key followed by the 32-byte `epk`). It MUST encode exactly the
canonical 32-byte little-endian field encoding of `pk_v2` followed by the existing 32-byte
serialized `epk`. Decoders MUST check both lengths and canonical field encoding. Unknown address
versions MUST fail closed; senders select the output construction from the decoded version.

## 5. Note commitment and ownership detection

```text
coin_info   = { nonce, type_, value }                                (unchanged payload)
coin_com_v2 = SHA-256("midnight:zswap-cc[v2]" || EncodeCoin(coin_info) || 0x01
                      || EncodeField32(pk_v2))
```

This is the v1 preimage layout — 21-byte separator, coin bytes, the user-branch byte `0x01`, then
32 bytes of data — with the v2 separator and `EncodeField32(pk_v2)` in place of the v1 coin
public key. The commitment is a SHA-256 output like every v1 commitment and is inserted into the
**existing** commitment tree (depth 32, 32-byte leaves; the tree hashes internal nodes with the
transient hash over the leaf degraded to its low 248 bits, identically for v1 and v2). No second
tree or nullifier set is introduced. Keeping the SHA-256 outer shape is a privacy requirement: a
bare field-element commitment would carry fixed zero bits in its 32-byte encoding and label the
note version on-chain.

Outputs are encrypted to `epk` exactly as today; the plaintext is `coin_info` only. It carries
neither recipient key nor note version, so the receiving wallet MUST recompute both candidate
commitments — v1 under its v1 coin public key, v2 under its `pk_v2` — and compare with the exact
on-chain output commitment. Exactly one match identifies ownership and version (store the version
with the coin); no match means not owned; two matches is an implementation error (SHA-256
collision across separators).

## 6. VRF nullifier

```text
vrf_input = PoseidonV2(Domain("midnight:zswap-vrf[v2]"), EncodeCoin(coin_info), pk_v2)
H_x       = UniqueHashToCurve("midnight:zswap-vrf-curve[v2]", vrf_input)
gamma     = sk_o * H_x
vrf_out   = PoseidonV2(Domain("midnight:zswap-vrf-out[v2]"), gamma.x, gamma.y)
nullifier = SHA-256("midnight:zswap-cn[v2]" || EncodeCoin(coin_info) || 0x01
                    || EncodeField32(vrf_out))
```

Properties the implementation MUST preserve:

- `vrf_input` contains only coin data and `pk_v2`, never transaction data, so the nullifier is
  one deterministic value per coin, in every attempted spend. The digest of the authorization
  (§8) MUST NOT enter `vrf_input`, `gamma`, or the nullifier.
- The key enters exactly once, at `gamma`. Everything from `gamma` onward is public arithmetic —
  given `gamma`, anyone can compute the nullifier.
- The nullifier keeps the v1 SHA-256 preimage shape (with `coin_info` repeated) purely so v1 and
  v2 nullifiers are identically distributed on chain and the compatibility circuit can share one
  SHA evaluation across branches.
- Two outputs with identical `coin_info` to one recipient collide on commitment and nullifier, so
  only one is spendable — existing v1 behavior, unchanged; senders draw fresh nonces per output.

**Key-holder discipline.** The key holder MUST derive `H_x` itself from `coin_info` and `pk_v2`
(or verify a supplied `H_x` by recomputation) before evaluating the VRF or authorizing. It MUST
NOT multiply `sk_o` into any point it did not derive — a host-chosen point is a static
Diffie-Hellman oracle. Hash-to-curve, scalar multiplication, and the frozen `PoseidonV2` suite are
therefore the key holder's minimum operation set. In a threshold group each participant derives
`H_x` independently.

**Viewing.** The v2 incoming viewing key is `(esk, pk_v2)`: it decrypts and identifies incoming
v2 notes but cannot derive nullifiers. For spend detection without the key, the host requests
`gamma` once per received note (one scalar multiplication in the secure element; batchable) and
stores it; the resulting **nullifier list** plus the incoming viewing key give complete viewing of
the account's v2 notes, including spends — suitable for auditors. A `gamma` reveals nothing about
`sk_o` beyond that one note's nullifier. A wallet restored from the incoming viewing key alone
sees receipts but not spends until it obtains the list from the key holder.

## 7. Note-version privacy and circuit kinds

Pre-activation leaves are publicly known to be v1, so a dedicated v2 verifier would shrink the
effective anonymity set even under one Merkle root. The activated protocol therefore uses
**compatibility circuits with a private boolean note-version selector**:

- `spend_compat` proves the v1 user-owned spend relation OR the v2 relation of this MIP, under
  one verifier identifier and one public-input shape.
- `output_compat` proves the v1 user-recipient output relation OR the v2 commitment relation,
  likewise.

Current Zswap proofs carry no circuit identifier on chain (validators use fixed built-in verifier
keys), so activation MUST add a consensus-visible **`ZswapCircuitKind`** — `Legacy` or `CompatV2`
— to every proof-bearing input and output, with separate input and output kinds on a transient (a
coin created and spent within the same transaction, which carries both an output and an input
proof).
It selects which static verifier key (and matching proof backend) applies; it MUST NOT be
conflated with the existing contract-call proof-version enum. The discriminant reveals use of the
compatibility circuit, never the private v1/v2 branch inside it.

Requirements: the selector MUST be private and constrained boolean; the selected branch MUST
constrain every value it uses; both branches MUST yield the same proof type, serialized proof
size, verifier identifier, and public statement shape; and canonical serialized transactions and
events MUST be inspected to confirm no other field associates a compatibility proof with its
branch. A v2 wallet MUST use the compatibility circuits; updated wallets SHOULD use
`spend_compat` for v1 notes too. Legacy v1 user-owned proofs remain valid only until a
**retirement height** fixed by the activation upgrade; afterwards user-owned inputs and
user-recipient outputs MUST use the compatibility circuits (paying a v1 address uses the private
v1 branch). Contract-owned coins keep the legacy circuits permanently and are outside the user
anonymity set. If equal-size/equal-shape branch privacy cannot be met within the performance
gate, the MIP returns to review with the measured anonymity regression stated.

## 8. Authorization scope and digest

The prover holds all the wallet's openings, so without a binding authorization it could attach
the wallet's inputs to outputs of its own. The authorization must bind what the wallet built —
but not the whole transaction, because independently proven offers are merged after proving: the
mechanism behind offer files (MIP-0005) and P2P atomic swaps (MIP-0006). Those MIPs are not
dependencies of this specification — nothing here requires them to be implemented — but their
merge-based flows are deployed semantics this design MUST preserve, which is why the
authorization is scoped rather than transaction-wide. Each wallet therefore attaches scopes to
every offer in which it spends v2 notes:

```text
AuthScope {
  segment:  u16                    // the carrying offer's segment; 0 = guaranteed offer
  inputs:   [InputRef]             // compatibility inputs/transients this scope authorizes
  outputs:  [(u16, OutputRef)]     // every output/transient the wallet created, in any offer
  intents:  [(u16, IntentHash)]    // intents this authorization depends on
}

auth_digest = SHA-256("midnight:zswap-spend-auth[v2]" ||
                      CanonicalSerialize(network_id) || CanonicalSerialize(scope))
```

The scoped records mirror the public (proof-erased) fields of the existing offer records, plus
the new circuit kinds, and exclude all proof bytes:

```text
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

`CanonicalSerialize` is the ledger's canonical tagged binary encoding (never JSON/CBOR/SCALE);
the new types MUST freeze field order and tags in the profile, `network_id` uses the ledger's
existing string encoding with its length framing, and test vectors MUST include complete encoded
bytes, not only hashes. `IntentHash` is the existing per-segment intent hash (the persistent hash
of the intent's signing envelope).

Rules:

- Each list is sorted by canonical serialization and contains no duplicates. Every scope contains
  at least one `InputRef`.
- An `InputRef` occurs in **exactly one** scope (that scope supplies its proof's digest). An
  `OutputRef` or `IntentHash` MAY occur in several scopes (co-authorization), never twice in one.
- A transient appears as `Transient(TransientDigest)` in both the input and output lists, binding
  both proof kinds and both value commitments as one record.
- A reference's segment is part of its identity; moving an item between segments invalidates
  every scope naming it. Outputs carry the segment of the offer holding them, so guaranteed-offer
  inputs can bind fallible-offer outputs and vice versa.
- A wallet MUST list every compatibility input it spends in the offer, every output and transient
  it created anywhere in the transaction, and every intent it created. It SHOULD list nothing it
  did not create. The scopes of an initially constructed offer MUST partition its compatibility
  inputs. A scope does not encode an owner identity and MAY cover inputs of multiple key holders
  if they co-sign one digest.
- Scopes authorize **inclusion of exact records at exact segments**; they do not alter execution
  semantics or make a fallible effect guaranteed. Custody policy that requires atomicity MUST
  reject scopes whose protected input and expected effects do not share the required segment.
- The scope list is a new offer field (`auth_scopes`). Offer merge MUST produce the union of both
  scope lists in canonical order, alongside the existing merge of coin sets and deltas; merging
  only adds items, so every scope survives every merge, and the merge's existing disjointness
  check guarantees no input is covered twice.
- No circularity: nullifiers, output commitments, ciphertexts, value commitments, and intents all
  exist before authorization; proof bytes and `spend_sig` never enter the scope.

## 9. Spend authorization (`spend_sig`)

After the scope is final and before proving, the key holder computes, per v2 input:

```text
z   = 32 fresh random bytes from the key-holding component

k   = HashToScalar("midnight:zswap-spendsig-nonce[v2]",
                   EmbedScalar(sk_o), vrf_input, EncodeDigest(auth_digest), EncodeDigest(z))

R_1 = k * G
R_2 = k * H_x

e   = HashToScalar("midnight:zswap-spendsig[v2]",
                   R_1.x, R_1.y, pk_o.x, pk_o.y, R_2.x, R_2.y,
                   gamma.x, gamma.y, H_x.x, H_x.y,
                   EncodeDigest(auth_digest))          // 12 inputs after the domain, this order

s   = k + e * sk_o  mod r

spend_sig = { pk_o, gamma, R_1, R_2, s }
```

The circuit recomputes `H_x` and `e` and enforces both Chaum-Pedersen equations:

```text
s * G   == R_1 + e * pk_o        // knowledge of the key behind the note
s * H_x == R_2 + e * gamma       // the same key produced the VRF output
```

One Schnorr equation is insufficient: it would authorize outputs without binding the nullifier's
VRF output to the note's key. Binding `e` to `auth_digest` is REQUIRED — without it `spend_sig`
is a bearer credential a prover could attach to different outputs. Changing any listed item, the
segment, or the network identifier changes the digest and requires a new authorization; the
nullifier never changes.

Nonce rules (consensus-independent but MANDATORY for signers):

- The nonce MUST be generated inside the key-holding boundary, unpredictable to every outside
  party; a host MUST NOT supply, suggest, or observe it.
- `z` MUST be fresh per authorization from the key holder's own entropy. The derivation is
  hedged (it commits to key, coin, and digest), so weak randomness degrades toward deterministic
  signing, not nonce reuse. A key holder without an entropy source is non-conforming.
- A nonce MUST NOT repeat across coins or digests (the derivation inputs already ensure distinct
  nonces for distinct coins even under repeated `z`).
- A zero-valued derived nonce MUST trigger fresh `z` and retry. `k` and `z` MUST be erased after
  computing `s` and MUST NOT reach logs, traces, crash reports, or the proving payload.
- Threshold groups do not evaluate `PoseidonV2` over a reconstructed key: they MAY produce the
  same aggregate DLEQ relation via a distributed protocol with distributed nonce generation,
  provided it has a published security analysis and reconstructs neither `sk_o` nor a complete
  nonce. (FROST covers one-base Schnorr, not this two-base proof, by itself.) The circuit is
  unaffected; it verifies only the aggregate proof.

## 10. The `spend_compat` circuit

Logical public statement — the existing user-owned spend statement plus the digest:

```text
coin_com_root, nullifier, segment, value_com, EncodeDigest(auth_digest)
```

The first four are the existing public-transcript effects (Merkle-root check, nullifier insert,
segment read, value-commitment write), not raw statement elements. `spend_compat` MUST add a
read-only public-transcript cell holding the two digest limbs and MUST read it unconditionally
before branch selection; the validator supplies those two values when constructing the statement,
exactly as it supplies the segment read today. Exported circuit arguments remain private, and a
compatibility input never discloses a contract address.

Private inputs:

```text
note_version, owner_witness, coin_info, merkle_path, rc, auth_binding
```

`rc` is the coin's existing Pedersen value-commitment randomness and `merkle_path` opens the
commitment in the shared tree. For `note_version == 1`, `owner_witness` is the existing v1 user
secret; for `note_version == 2` it is `(pk_o, gamma, R_1, R_2, s)`.

**Digest-binding property (REQUIRED).** Both public digest limbs MUST be constrained independently
of branch selection: a proof from either branch MUST fail under a modified digest, and the
implementation MUST demonstrate exactly that. The reference realization is a private input
`auth_binding`, enforced before branch selection as

```text
auth_binding == PoseidonV2(Domain("midnight:zswap-auth-bind[v2]"), EncodeDigest(auth_digest))
```

which forces the limbs through a constraint that circuit compilation cannot eliminate as dead
v2-only code. A cheaper branch-independent constraint over the limbs MAY replace it if the
compiled-circuit inspection (§3) confirms the property; the chosen realization — including the
dropped `auth_binding` witness and domain label, if unused — is frozen with the circuit.

| ID | Constraint |
| --- | --- |
| S1 | Read both `auth_digest` limbs from the dedicated public-transcript cell and constrain them branch-independently per the digest-binding property, before branch selection. |
| S2 | Constrain `note_version` to 1 or 2 and select exactly one complete ownership branch without revealing which. |
| S3 | v1 branch: enforce the existing v1 user owner-key, coin-commitment, and nullifier relations. |
| S4 | v2 branch: typed subgroup-constrained points; reject identity for `pk_o`, `gamma`, `R_1`, `R_2`; explicitly constrain `s < r`. |
| S5 | v2 branch: recompute `pk_v2`, `coin_com_v2`, `vrf_input`, `H_x`, `vrf_out`. |
| S6 | v2 branch: recompute `e` (including the public digest) and verify both DLEQ equations. |
| S7 | v2 branch: recompute the nullifier from `vrf_out` and constrain it to the public nullifier. |
| S8 | Both branches: verify the selected commitment's Merkle path against `coin_com_root`. |
| S9 | Both branches: recompute the existing Pedersen `value_com` from `(value, type_, rc)`. |

Derived values (`vrf_input`, `pk_v2`, `H_x`, `vrf_out`, `e`, `coin_com_v2`, nullifier) SHOULD be
computed in-circuit; any supplied as an optimization MUST be recomputed and constrained. Because
both branches hash an identically shaped preimage, the circuit SHOULD evaluate the commitment and
nullifier SHA-256 relations once each over `sep`/`data` fields multiplexed by the selector.

**Proving payload prohibition.** The v2 proving preimage contains the selector, `spend_sig`,
`coin_info`, `merkle_path`, `rc`, and `auth_binding` — and MUST NOT contain `sk_o`, any v1 coin
secret key, the nonce `k`, the value `z`, or wallet seed material. An automated wallet test MUST
decode the final proving-request body and enforce this in continuous integration. The proof
server remains a generic executor; a minimal per-proof request needs `auth_digest` and
`auth_binding`, not the scope, though the security analysis assumes a prover that sees the whole
transaction.

## 11. The `output_compat` circuit

Public statement: the existing user-recipient output statement (coin commitment, value
commitment, segment, ciphertext binding). Private inputs: `note_version`, the selected v1 or v2
recipient witness, `coin_info`, `rc`. For `note_version == 1` it enforces the complete existing
v1 user-recipient relation; for `note_version == 2` it recomputes `coin_com_v2` from `pk_v2` and
enforces the same value-commitment, segment, and ciphertext-binding rules as v1. The selector is
private and constrained as in §7. A sender that decodes a v2 address MUST use `output_compat`
with the v2 branch; outputs to contracts keep the legacy circuit.

## 12. Consensus encoding changes

- Add `ZswapCircuitKind` to every proof-bearing Zswap input and output (two kinds on a
  transient), and `auth_scopes: [AuthScope]` to the offer.
- These fields require new tagged serialization versions of the Zswap input, output, transient,
  and offer records and of the enclosing transaction formats. Historical decoders MUST interpret
  all old versions as `Legacy` with an empty scope list. Unknown discriminants and versions MUST
  fail closed, never be interpreted as the latest version.
- The accepted domain labels, discriminants, encodings, public-input shapes, and test vectors of
  this note version are immutable; future changes require new note/transcript/circuit versions in
  a subsequent MIP.

## 13. Validator rules

Static verifier material for the two compatibility circuits is embedded like the existing
built-in Zswap verifier keys and dispatched from `ZswapCircuitKind` (proven transactions carry no
resolver hints). Proving/verification material MUST be published with pinned hashes and verified
provenance, registered wherever proving material is resolved and prefetched, and included in
proof-size, public-input-count, and validation-cost accounting.

The following run in a transaction-level validation pass, after all offers and intents are
available (a scope may name records in other segments), for the guaranteed offer (segment 0) and
each fallible offer:

1. Scope lists and nested reference lists are canonically ordered with no duplicates; every scope
   has at least one `InputRef`.
2. Every scope's `segment` equals the segment of the offer carrying it.
3. The digest of every compatibility input or transient-input proof in the offer appears as an
   `InputRef` in exactly one scope of that offer, and every listed `InputRef` matches exactly one
   public record. Legacy input proofs appear in no scope.
4. Every listed `(segment, OutputRef)` matches exactly one public output or transient record in
   the offer at that segment.
5. Every listed `(segment, intent_hash)` matches the transaction's intent hash at that segment.
6. `auth_digest`, recomputed from the transaction's network identifier and the scope, is supplied
   as the public digest for every input the scope lists; a proof whose public inputs differ is
   rejected.
7. A scoped input MUST carry `CompatV2`. After the retirement height, an unscoped user-owned
   input is rejected and every user-recipient output MUST carry `CompatV2`.
8. Contract classification uses the existing public `contract_address`: `Some(_)` requires
   `Legacy`; `CompatV2` requires `None`; a transient with a contract address requires `Legacy`
   for both kinds.

Validation MUST run in `O(N log N)` or better over public records plus nested references
(canonical order / indexed maps); an unmetered quadratic scope surface MUST NOT be accepted, and
transaction size/cost limits MUST bound worst-case scope validation. The validator selects only
the compatibility verifier and MUST NOT learn or infer the private branch. The v2 nullifier uses
the existing 32-byte nullifier type, set, and uniqueness rule; root freshness, balancing, binding
commitments, fees, TTL, and replay protection are unchanged.

**Activation.** Height- or epoch-gated network upgrade. Before activation: reject compatibility
proofs and scope-bearing offers (mempools SHOULD reject with a specific error). At and after:
accept compatibility proofs and, until the retirement height, legacy v1 user-owned proofs. Both
compatibility circuits MUST fit the maximum circuit size covered by the already published,
provenance-checked universal KZG parameters — no new ceremony, and no locally generated parameter
file as consensus material; if they do not fit, the MIP returns to review.

## 14. Out of scope / unchanged

- **Contract-owned coins**: keep the legacy circuits and v1 formulas; no v2 contract address; a
  contract never holds `sk_o`.
- **Claims**: the dormant claim circuit stays unavailable to active transaction variants;
  reactivating it for user-owned coins requires a subsequent MIP that does not witness `sk_o`.
  (The separate rewards-claim transaction is signature-authorized and unaffected.)
- **Dust**: unchanged; its spend circuit still witnesses the Dust key, so a delegated prover
  still receives it. Exposure is bounded to fee-capacity griefing (the Dust key cannot move NIGHT
  or other assets); expected deployment keeps Dust as a host-side hot key until a follow-up MIP.
- **Encryption path**: key pair, ciphertext format, and trial decryption unchanged; indexers keep
  deciding relevance by decrypting `coin_info` and only need to decode the new tagged formats.
- No permits, allowances, delegates, or authorization objects with on-chain lifecycle; no fee,
  balancing, expiry, or replay changes; no privacy from the prover itself.

## 15. Security requirements and accepted tradeoffs

- **Prover authority**: a prover learns the transaction's witnesses, `pk_o`, and `gamma`; it can
  link requests under one key (`pk_v2 = PoseidonV2(pk_o)` identifies the spending address),
  refuse service, or front-run — but cannot recover `sk_o` (discrete log), compute any other
  coin's nullifier (CDH/DDH), or alter anything covered by `auth_digest`. Delegated proving is
  non-custodial, not private; wallets MAY rotate provers or self-host. A failed or withheld proof
  costs nothing: the same request, including the cached `spend_sig`, can go to another prover
  without involving the key holder; wallets MUST verify returned proofs before broadcast.
- **Scope completeness is a wallet obligation**: the validator proves listed items exist and all
  compatibility inputs are covered, but cannot know which outputs a wallet created. Conformance
  tests MUST include a malicious prover attempting every omission, alteration, and re-pairing.
- **Nonce reuse reveals the key** (standard Schnorr algebra) — hence the hedged derivation,
  digest/coin inclusion, and host-supplied-nonce prohibition. Fixed-`z` conformance vectors MUST
  exist so independent signers can be cross-checked.
- **Digest/nullifier separation is load-bearing**: tests MUST show every scope-field change
  alters `spend_sig` but never the nullifier, and every coin-field change alters the nullifier.
- **Transcript agreement fails closed** (an unsatisfiable proof) but can strand funds at scale:
  cross-language byte-exact vectors — including leading-zero coordinates, maximum-width values,
  non-canonical encodings, empty and multi-entry scopes, and every domain label — are an
  activation requirement.
- **Commitment binding**: v2 binds ownership by SHA-256 over Poseidon-derived inner values;
  `EncodeField32` retains 248 bits, bounding inner collision resistance at 124 bits (matching the
  tree's existing leaf degradation). The composition is part of the mandatory cryptographic
  review.
- **Scopes reveal transaction-graph structure**: every reference maps to a public record, so
  merged transactions expose each party's partition where current Zswap shows one undifferentiated
  set. Parties needing that ambiguity merge first and co-authorize one scope. This regression MUST
  be measured and documented for offer files, atomic swaps, and multi-party merges; accepting the
  MIP accepts the tradeoff.
- **Custody boundary**: hardware and threshold deployments keep `sk_o`, nonce generation, and VRF
  evaluation inside the boundary, derive `H_x` themselves, and export only `pk_o`, per-note
  `gamma`, and authorizations. Host compromise reveals history (privacy), never funds.

## 16. Acceptance gates

1. **Phase 0 (gate)**: byte-exact cryptographic profile frozen (every encoding, transcript order,
   hash-to-scalar and unique hash-to-curve relation, scope serialization); circuit prototypes;
   serialized-metadata inspection confirming no note-version discriminant and equal proof sizes
   across branches; published test vectors; independent cryptographic review (DLEQ soundness,
   nonce derivation, transcript binding, subgroup/identity checks, SHA-over-Poseidon composition,
   scope completeness against a malicious prover, cross-protocol domain separation) with all
   critical findings resolved; v1-vs-v2 benchmarks (constraint counts, parameter size `k`,
   proving-key size, proving/verification time, payload size) on declared hardware; both circuits
   fit the published SRS range. Material changes return to MIP review.
2. Independent Rust and TypeScript implementations reproduce all canonical vectors byte-for-byte:
   seed→`sk_o`, `pk_o`, `pk_v2`, commitment, `vrf_input`, `H_x`, `gamma`, `vrf_out`, nullifier,
   scope serialization, `auth_digest`, DLEQ points, challenge, response (fixed `z` per vector);
   plus rejection tests for every mutated transcript field, alternative-`H_x` attempts, zero and
   out-of-range scalars, identity/small-order/off-curve/non-canonical points, and wrong domains;
   plus native-vs-circuit differential tests on randomized valid inputs; and the vectors remain
   unchanged under a test-only replacement of the generic transient hash.
3. Ledger/validator conformance: the rule-by-rule scope rejection matrix of §13 (unlisted input,
   mismatched digest, absent output, wrong intent/segment, duplicate coverage, empty scope,
   non-canonical order, scope-bearing contract input); merge of proven offers stays valid while
   removal of any listed item invalidates; same nullifier across retries and double-spend
   rejection; digest change invalidates proofs of **both** branches (preventing a
   branch-identification oracle); activation- and retirement-boundary behavior; old formats
   decode as `Legacy` with empty scopes; existing v1 vectors still verify.
4. End-to-end on a public testnet: create, receive, scan, export VRF outputs, authorize locally,
   prove through an independently operated remote prover, broadcast, verify, reject a double
   spend; a merged transaction with one party's v2 inputs; a host holding only the incoming
   viewing key plus nullifier list detecting the key holder's spend; the captured proving request
   containing no key, nonce, `z`, or seed material; v1 notes spent after activation and one
   migrated to v2.
5. Network upgrade deployed with v1 notes remaining spendable; at least one production wallet
   completes the v2 round trip without ever placing the spend key in a proving request.

## References

- [MPS-0035: Shielded Spend Authorization Requires Exposing the Spend Key](../mps/mps-0035-shielded-spend-key-exposure.md)
- [MPS-0024: Custodian-Safe Native Shielded Asset Transfer](../mps/mps-0024-custodian-safe-shielded-spends.md)
  and [MPS-0016: Custodian-Compatible Shielded Wallet Generation](../mps/mps-0016-custodian-shielded-wallet-generation.md)
  — related problem statements also addressed: the threshold-usable key and share-derivable
  address this design provides.
- [MIP-0005: Offer Files](mip-0005-offer-files.md) and
  [MIP-0006: P2P Atomic Swaps](mip-0006-p2p-atomic-swaps.md) — related, not required: the
  deployed merge-based flows the authorization scope is designed to preserve.
- Chaum, Pedersen, "Wallet Databases with Observers," CRYPTO 1992 (DLEQ proof).
- [RFC 9381: Verifiable Random Functions](https://www.rfc-editor.org/rfc/rfc9381) (security
  definitions only; this construction is not wire-compatible with any RFC 9381 ciphersuite).
- [RFC 6979](https://www.rfc-editor.org/rfc/rfc6979) (deterministic/hedged nonce background),
  [RFC 9591 (FROST)](https://www.rfc-editor.org/rfc/rfc9591) (threshold background).
- [Zcash Protocol Specification (NU5)](https://zips.z.cash/protocol/protocol.pdf) (comparison
  point for spend authorization and nullifier design).

## Copyright

Licensed under Apache-2.0, per the Midnight Foundation Contributor License Agreement.
