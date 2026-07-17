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

---
MIP: "0013"
Title: Multi-key Account Authorisation for Custody Contracts
Authors:
  - Hector Bulgarini (hbulgarini)
  - Nicolas Di Prima (NicolasDP)
Status: Proposed
Category: Standards
Created: 2026-07-10
License: Apache-2.0
Requires: MIP-0012: Contract Custody of Midnight-Native Assets
Replaces: none
---

## Abstract

This MIP specifies the multi-key account standard for Midnight custody
contracts: who may release an account's assets, how that authority is
proven, and how it is granted, revoked, and rotated across a set of
independent device keys. It is the second building block of the
multi-key account contract recommended by MPS-0018, and it instantiates
the authorisation seam that the Contract Custody of Midnight-Native
Assets MIP (hereafter "the custody MIP") deliberately left abstract.

Authorisation is by digital signature: each device holds an independent
Schnorr keypair on the JubJub curve, and every asset-releasing circuit
verifies, in-circuit, a signature over a challenge that binds the
account, the circuit, its arguments, the witness values it consumes
(pinning the private state the operation executes against), and a
per-account authorisation counter. The choice of an asymmetric signature scheme over the cheaper
hash-preimage alternative is deliberate, and carries the standard's
three load-bearing properties. First, Schnorr admits the FROST
threshold-signing protocol, so a key can be held as shares across an
MPC committee that never reconstructs it — the composability with
threshold custody that MPS-0018 records as broken at the proving
boundary. Second, JubJub is the embedded curve of Midnight's proof
system, so the curve arithmetic of verification — `ecMulGenerator`,
`ecMul`, `ecAdd` — is native in-circuit, with no foreign-field
emulation; the challenge hash is the standard library's
version-stable persistent hash (Rationale R3). Third, a signature
separates approval from
proving: the party that authorises an operation produces a short
artefact and needs no Midnight stack, while the party that proves needs
the artefact and not the credential, so key custody and proof
generation can live in different trust domains.

The scheme has been validated end to end on a devnet node, with
independent TypeScript and Rust signers producing interchangeable
signatures against the same in-circuit verifier. The device set stores
rolling, single-use entries, so no stable device identifier ever rests
in ledger state; it carries an epoch mechanism that revokes every
stale authorisation in one step, and a recovery seam, left abstract
here, for the recovery-paths MIP recommended by MPS-0018.

## Motivation

MPS-0018 frames the problem: Midnight has no ratified model for an
on-chain account that custodies and authorises a user's native assets
under multi-key, multi-device control while satisfying lost-device
recovery, total-loss recovery, and key non-exfiltration at once. The
custody MIP delivered the asset half of the recommended keystone: how a
contract holds and releases value, with every release gated by a single
abstract seam, `require_authorised()`, whose observable semantics it
fixes and whose credential scheme it deliberately does not. This
document is the companion "instantiating standard" its section 1
anticipates: it supplies the credential scheme and the key-management
lifecycle, and nothing in the custody rules changes underneath it.

Three specific inadequacies motivate the scheme chosen here.

**Threshold custody has nothing to attach to.** MPS-0018 records that
threshold-signature MPC does not compose with proving: shielded and
contract operations are authorised by proof generation, the secret
enters the proof as a witness, and the witness must be assembled in one
place to prove — reintroducing the single point of key exposure that
MPC exists to remove. MPS-0024 raises the institutional form of the
same gap: custodian control frameworks are built around signature
ceremonies, and Midnight's proof-authorised asset movement bypasses
them. An account whose seam verifies a *signature* inverts this: the
nodes of a threshold committee jointly produce the signature, the
circuit verifies it, and no step requires the key — or even a key
share — to reach the proving machine.

**Hash-preimage authorisation binds the credential to the prover.** The
cheapest seam instantiation, and the one the account-custody prototype
used as a placeholder, gates circuits on knowledge of a committed
32-byte secret. The witness *is* the long-term credential: whoever
generates the proof holds it, whole, for every authorised call. That
excludes threshold custody structurally, makes delegated proving
(MPS-0004) equivalent to handing over the account, and turns every
proving environment into key-custody infrastructure.

**Upstream caller identity is unsettled and single-key.** In-circuit
verification of native address ownership has been requested in the
token-standards discussion, a stdlib caller-identity primitive has been
proposed, and MPS-0010 and MPS-0011 (the problem statements on
signature verification and native crypto primitives) frame the wider
demand. A caller identity derived from ledger-scheme keys binds
authority to a single secp256k1 key rather than to an account policy,
cannot authenticate shielded callers, and does not compose with a
threshold scheme on another curve (custody MIP, section 4). An account
standard needs authority to attach to a *policy over a device set*,
not to one wallet key — and it needs it on a curve the circuit can
afford.

The absence of any standard here has a demonstrated cost: contracts in
the wild authenticate callers with `ownPublicKey()`, which is
wallet-supplied and unverified, and the attack has been reproduced
against deployed examples. The upstream position is that the primitive
was never intended for authentication; the safe pattern — verify a
signature in-circuit against a stored key — is exactly what this MIP
standardises.

## Specification

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are to be
interpreted as described in RFC 2119.

### 1. Terminology

- **Account**: a custody contract instance conforming to the custody
  MIP whose authorisation seam this standard instantiates.
- **Device**: a key-holding party able to authorise account operations:
  a phone, a laptop, a hardware module, or a threshold committee
  presenting a single group key (section 7).
- **Device key**: an independent Schnorr keypair `(sk, pk = sk·G)` on
  the JubJub curve. Device keys are never derived from one another or
  from a shared seed.
- **Device entry**: the domain-separated hash of a device public key
  together with the account address, the current epoch, and the
  device's use counter, stored as a single-use element of the
  on-ledger device set. Every authorised call consumes the acting
  device's current entry and records its successor.
- **Use counter**: a per-device monotone counter bound into the device
  entry; it advances by one each time the device authorises a call.
- **Epoch**: a monotone counter over the device set; device entries are
  valid only if their recorded epoch equals the current one.
- **Authorisation counter** (`auth_nonce`): a monotone counter bound
  into every challenge; the freshness element of the custody MIP's
  seam semantics.
- **Challenge**: the scalar a signature is verified against, computed
  in-circuit over the call it authorises (section 5).

### 2. Curve and hash

The signature scheme is Schnorr over the JubJub curve with a challenge
computed by the Compact standard library's persistent hash:

- **Group.** The prime-order subgroup of JubJub, the twisted Edwards
  curve defined over the BLS12-381 scalar field. Its subgroup order is
  `r_J = 0x0e7db4ea6533afa906673b0101343b00a6682093ccc81082d0970e5ed6f72cb7`
  and its cofactor is 8. `G` is the generator used by Compact's
  `ecMulGenerator`.
- **Challenge hash.** Compact's `persistentHash`: SHA-256 over the
  compiler's field-aligned binary encoding of the typed preimage of
  section 5, recomputed in-circuit. This is a deliberate departure
  from RedJubjub as deployed elsewhere, which hashes with BLAKE2b-512
  (unavailable in-circuit), and a deliberate rejection of the cheaper,
  circuit-optimised `transientHash` (Poseidon-family), whose output
  the toolchain does not guarantee across upgrades. Both choices are
  argued in Rationale R3 and reviewed under Path to Active.
- **Scalar domain.** Signatures are computed modulo `r_J`. Because the
  hash output is a 32-byte value that may exceed `r_J`, signers
  grind a nonce until the challenge falls below `r_J` (section 5.2), so
  both sides of the verification equation operate on the same scalar.

Off-circuit signers MUST reproduce `persistentHash` bit-exactly,
including the Compact compiler's field-alignment encoding of the typed
preimage. This is demonstrably implementable outside the TypeScript
toolchain: the reference signers include a pure-Rust implementation
producing signatures interchangeable with the TypeScript one
(Implementation section).

### 3. Public ledger state

An account extends the custody MIP's ledger state (its section 3) with
the device set. This extension is of the kind the custody MIP permits:
no invariant of its section 7 is weakened, and no coin material is
recorded.

```compact
export ledger devices: Set<Bytes<32>>;
// Single-use device entries (below). Every gated call consumes the
// acting device's current entry and inserts its successor (section 4).

export ledger device_epoch: Uint<32>;
// Bumped by recovery. Entries hashed under an older epoch can never
// match an in-circuit recomputation, so they are inert (AUTH-6).

export ledger device_count: Uint<8>;
// Devices active in the current epoch. Guards the last-device rule.

export ledger auth_nonce: Uint<64>;
// Bumped by every seam-gated call, and only by seam-gated calls.
// Every challenge covers it (section 5); a signature authorises at
// most one call.
```

The device entry for a public key `pk` at use counter `k` is:

```compact
persistentHash<[Bytes<32>, ContractAddress, JubjubPoint, Uint<32>, Uint<64>]>(
  [DST_DEVICE, kernel.self(), pk, device_epoch, k]
)
```

where `DST_DEVICE` is the domain-separation tag
`"midnight:account:device:v1"`, zero-padded to 32 bytes. All tags
introduced by this MIP are to be registered under the registry
recommended by MPS-0027.

Each bound element carries an obligation. The account address makes
entries for the same key in different accounts unequal, so a device
key registered in several accounts produces no matching ledger values
anywhere. The epoch makes revocation total without enumerating the
set: after a bump, no stale entry can match a recomputation under the
current epoch. The use counter makes every entry single-use, so the
ledger never holds a device identifier that is stable across two
authorised calls (AUTH-9). This is the departure from a static device
commitment, which is a fixed pseudonym disclosed on every call; a
rolling entry is disclosed at most twice, once when inserted and once
when consumed (Rationale R8, Security Considerations S4).

The entry construction carries a bootstrap constraint: a contract's
address is derived from the deploy transaction's content, so no
deploy-time code can know it, and `kernel.self()` evaluates to the
zero address inside a constructor. The initial entry therefore cannot
be computed at deploy time. An account is deployed dormant and
activated by its first call:

- The constructor takes a **boot commitment**

  ```compact
  persistentHash<[Bytes<32>, Bytes<32>, JubjubPoint]>([DST_BOOT, salt, pk])
  ```

  where `DST_BOOT` is the tag `"midnight:account:boot:v1"` zero-padded
  to 32 bytes and `salt` is 32 bytes fresh per account, and stores it
  with an empty device set, `device_count = 0`, and `auth_nonce = 0`.
  Gated circuits are unusable before activation, and assets deposited
  in the gap are releasable only by the committed key.
- A permissionless circuit `activate_initial_device(pk, salt)`
  asserts the account is not yet activated, recomputes the commitment
  from its arguments and asserts it matches the stored one, inserts
  the device entry for `pk` at epoch 0 and use counter 0, sets
  `device_count = 1`, and burns the commitment. Activation is
  deterministic in the committed key: a front-runner replaying an
  observed `(pk, salt)` can only install the key the deployer
  committed to, so the call needs no authorisation.
- The salt MUST be fresh per account. Without it the commitment is a
  stable function of the device key alone, equal across every account
  the same key deploys, and pre-activation state would link a user's
  accounts (the property the address-bound entries exist to prevent).

Deployment and activation SHOULD be submitted back to back; the
account is usable once both are final.

### 4. Seam instantiation

The custody MIP requires every asset-releasing circuit to be gated by
`require_authorised()` and fixes its observable semantics: the proof of
authority MUST bind to the circuit identity, its arguments, and a
freshness element the contract state advances, and a transferable
artefact such as a signature MUST cover the freshness element
explicitly. This section instantiates that seam.

A gated circuit takes, in addition to its own arguments, the
authorising material:

```compact
pk:          JubjubPoint,  // the signing device's public key
use_counter: Uint<64>,     // the device's current use counter
sig_r:       JubjubPoint,  // Schnorr commitment R = r·G
sig_s:       Field,        // Schnorr response s = r + c·sk (mod r_J)
grind_nonce: Uint<64>,     // challenge-grinding counter (section 5.2)
```

and performs, in order:

1. **Consume the device entry.** Compute the device entry of `pk` at
   `use_counter` under the current `device_epoch` (section 3); assert
   `devices.member(entry)`, remove it, and insert the successor entry
   at `use_counter + 1`. An unregistered key, a stale epoch, and a
   wrong counter all fail this step identically.
2. **Challenge.** Recompute the challenge `c` over this call's preimage
   (section 5).
3. **Verification.** Assert `ecMulGenerator(sig_s) ==
   ecAdd(sig_r, ecMul(pk, c))`.
4. **Freshness.** Increment `auth_nonce`, and increment `round` per the
   custody MIP.

On any assertion failure the circuit aborts and no state changes. Any
active device is sufficient to authorise (1-of-n at the contract
surface); threshold policies live behind a single device entry
(section 7), and richer on-ledger policies (m-of-n across device
entries, scoped grants) are permitted extensions that interact only
with this seam and MUST NOT weaken any invariant of section 9.

The authorising material is witness data of the transaction's proof: it
appears in no public output, and MUST NOT be disclosed by the circuit.
The entry consumed and the successor inserted in step 1 are disclosed
by the ledger operations themselves; both are single-use values, and
the consequences are bounded in Security Considerations S4. Transport of the signature between signer and prover is
client-internal and out of scope: the circuit consumes the typed values
above, and no wire format is consensus-relevant.

### 5. Challenge construction

#### 5.1 Preimage

The challenge for a gated circuit is:

```compact
const h: Bytes<32> = persistentHash<[Bytes<32>, ContractAddress, JubjubPoint, JubjubPoint, ...Args, ...Witnesses, Uint<64>, Uint<64>]>(
  [DST_CIRCUIT, kernel.self(), sig_r, pk, ...args, ...witness_values, auth_nonce, grind_nonce]
);
const c: Field = h as Field;
```

where:

- `DST_CIRCUIT` is a per-circuit domain-separation tag of the form
  `"midnight:account:auth:v1:<circuit-name>"`, zero-padded or hashed to
  32 bytes, distinct for every gated circuit and registered under the
  MPS-0027 registry. Distinct tags make signatures for one circuit
  unusable for any other, independent of argument encoding.
- `kernel.self()` is the account's own contract address. Its inclusion
  makes a signature for one account unusable against another account
  in which the same device key is registered.
- `...args` is the gated circuit's full argument list, excluding the
  authorising material itself, in declaration order, with their Compact
  types.
- `...witness_values` is the ordered sequence of values returned by
  every witness invocation the gated circuit performs, excluding the
  authorising material, with their Compact types. This pins the
  private state. Witness values are drawn from a client-held mutable
  store (a JSON document in current client stacks) that the proving
  environment reads, and can edit; a challenge that did not cover them
  would authorise a call shape rather than an execution (Rationale R9,
  AUTH-10). Because the circuit recomputes the challenge over the
  witness values it actually consumes, a proof generated against any
  altered value cannot satisfy the verification equation.
- `auth_nonce` is the current authorisation counter (pre-increment).

The preimage MUST include every element above. Implementations
SHOULD additionally bind a network identifier (the CAIP-2 identifier of
MIP-0008) where deployments may exist on more than one network;
inclusion is via the `DST_CIRCUIT` tag or an explicit element.

#### 5.2 Grinding

The signer computes `h` for `grind_nonce = 0, 1, 2, …` and uses the
first value whose little-endian integer interpretation is strictly less
than `r_J`. Since `r_J ≈ 2^251.85`, each attempt succeeds with
probability about 5.7%, and the expected number of attempts is about
17.5; the search is over public-to-the-signer data and reveals nothing
secret. The circuit recomputes the identical hash, so the `as Field`
cast (range-checked against the BLS12-381 scalar modulus, which
exceeds `r_J`) always succeeds for an honestly ground challenge, and
the scalar the circuit multiplies by equals the scalar the signer
signed with. Rejection sampling leaves the challenge distribution
uniform over `[0, r_J)`.

#### 5.3 Signing

To authorise a call, a device:

1. Samples a nonce scalar `r` uniformly from `[0, r_J)`. Nonce
   generation MUST ensure `r` is never reused across distinct
   challenges and never predictable; deterministic derivation in the
   manner of RFC 6979, hedged with randomness, is RECOMMENDED
   (Security Considerations S1).
2. Computes `R = r·G`.
3. Grinds the challenge per section 5.2 over the exact preimage of
   section 5.1, using the `auth_nonce` value and the witness values the
   call will execute against. The approver therefore MUST be given the
   witness values by the client pipeline, and SHOULD render them for
   review before signing: approval covers the state the operation runs
   against, not only its arguments (R9).
4. Computes `s = r + c·sk mod r_J`.
5. Outputs `(R, s, grind_nonce)`.

Steps 1–2 and 4 require only JubJub scalar arithmetic; step 3 requires
SHA-256 and the field-alignment encoding of the preimage. A signer
therefore needs no Midnight node, indexer, prover, or contract
runtime — the property Rationale R5 builds on.

### 6. Device lifecycle

- `add_device(new_pk: JubjubPoint)` (gated): computes the device entry
  of `new_pk` at the current epoch and use counter 0, inserts it, and
  increments `device_count`. MUST fail if that entry is already
  present. The contract cannot detect re-registration of an active key
  whose chain has advanced past counter 0 (its current entry is not
  recomputable without the counter); a re-registered key yields two
  parallel entries for one credential and inflates `device_count`
  until both are removed. Clients MUST track the device roster and
  MUST NOT re-add an active key (S11).
- `remove_device(entry: Bytes<32>)` (gated): removes a device's
  current entry. MUST fail if the entry is not present, or if
  `device_count == 1` (AUTH-5). Decrements `device_count`. The caller
  supplies the literal set element, so removing a lost device requires
  that device's public key and current use counter; a client recovers
  the counter from its roster or by the rescan of S11.
- Lost-device response is `remove_device` from a surviving device.
  Compromised-device response is the same, and Security Considerations
  S7 bounds the exposure window.

Adding a device transfers no key material: the new device generates its
keypair locally and exports only `pk`. How `pk` reaches an existing
device (QR code, ceremony) is client behaviour, out of scope; clients
SHOULD authenticate the channel, since whoever's key is added gains
full 1-of-n authority.

### 7. Threshold devices (FROST profile)

The verifier of section 4 is deliberately threshold-agnostic: it checks
one Schnorr equation against one public key and does not know or care
whether `(R, s)` was produced by one signer or by a quorum. A
**threshold device** is a `t`-of-`n` committee registered as a single
device entry whose public key is the committee's joint key.

A conforming threshold device MUST satisfy:

- **Key generation.** The joint key is produced by a distributed key
  generation protocol (or by a trusted dealer who verifiably deletes
  the dealt key), such that no party ever holds the full private key.
- **Signing.** Signature shares are produced and aggregated in the
  manner of FROST (RFC 9591): each participant contributes a nonce
  commitment and a signature share computed from its key share; the
  coordinator aggregates shares into a single `(R, s)`. The full key is
  never reconstructed, at signing time or otherwise.
- **Ciphersuite.** The challenge MUST be computed exactly per section
  5: the coordinator aggregates the group commitment `R`, performs the
  grinding search of section 5.2 (grinding varies only `grind_nonce`,
  after `R` is fixed), and distributes the resulting challenge to
  participants. This constitutes a new FROST ciphersuite — JubJub with
  the section 5 challenge — and its specification and review are
  acceptance criteria under Path to Active, not deliverables of this
  document.

The account contract is unchanged under this profile; that is the
point. Committee-internal concerns — participant authentication, share
refresh, liveness — are the committee operator's, and an account MAY
mix threshold and single-signer devices freely.

### 8. Recovery seam

The contract MUST expose exactly one path that changes the device set
without an active device: a recovery circuit that (a) verifies a
recovery authorisation whose mechanism this MIP does not specify,
(b) increments `device_epoch`, (c) registers exactly one fresh device
entry at the new epoch and use counter 0 and sets `device_count` to 1,
and (d) increments `auth_nonce` and `round`. Entries recorded under
earlier epochs are not removed (the set cannot be enumerated
in-circuit); they are permanently inert (AUTH-6), and their number is
bounded by `device_count` at the moment of the bump.

Under contract custody, recovery restores asset access by construction:
assets sit in the contract, the epoch bump changes who may authorise,
and the custody MIP's inbox walk rebuilds the coin store from chain
data. The recovery authorisation mechanism is the subject of the
recovery-paths MIP recommended by MPS-0018 and MUST be specified there
before this seam is instantiated.

### 9. Invariants

A conforming implementation MUST satisfy all of the following, in
addition to the custody MIP's INV-1 through INV-8.

- **AUTH-1 (signature gate).** Every seam-gated circuit verifies a
  Schnorr signature per sections 4–5 against a device key active in
  the current epoch; no gated circuit proceeds on any other authority.
- **AUTH-2 (single use).** Every challenge covers `auth_nonce`, and
  every gated call increments it: a signature authorises at most one
  executed call.
- **AUTH-3 (full binding).** Every challenge covers the account
  address, the per-circuit tag, and the complete argument list: a
  signature authorises only the exact call it was computed for, on the
  exact account.
- **AUTH-4 (non-exfiltration).** No circuit and no conforming client
  flow requires a device private key, or a share of one, to leave the
  device that generated it. In particular, proof generation consumes
  the signature and never the signing key.
- **AUTH-5 (no lock-out by ceremony).** `remove_device` cannot remove
  the last active device.
- **AUTH-6 (revocation totality).** After an epoch bump, no credential
  recorded under a previous epoch can authorise any operation. Entries
  bind the epoch inside the hash, so staleness needs no per-entry
  state: a stale entry can never match an in-circuit recomputation.
- **AUTH-7 (seedlessness).** Device keys are mutually independent; no
  flow requires a shared seed or derives one device's key from
  another's.
- **AUTH-8 (deposit independence).** Permissionless operations
  (deposits, inbox appends) do not read or advance `auth_nonce`; a
  third party's deposit cannot invalidate a pending authorisation
  (Rationale R4).
- **AUTH-9 (single-use device handles).** No value identifying a
  device appears in ledger state, or in any circuit's disclosed
  operations, more than twice: once when its entry is inserted and
  once when it is consumed. In particular, no device identifier is
  stable across two authorised calls, and no ledger value is equal
  between two accounts that register the same device key.
- **AUTH-10 (private-state binding).** Every challenge covers every
  witness value the gated circuit consumes. A proof generated against
  a private state that differs, in any consumed value, from the state
  the signer approved cannot satisfy the verification equation: the
  signature authorises an execution, not a call shape.

### 10. Versioning

- This specification is versioned by its MIP number and revision
  history; substantive changes after acceptance require a new MIP that
  lists this one in `Replaces`.
- The credential scheme specified here carries the identifier
  `midnight:account:auth:v1` (the prefix of every `DST_CIRCUIT` tag).
  A successor scheme — a different curve, hash, or policy structure —
  is a new tag prefix under a new or revising MIP; the domain
  separation of section 5.1 keeps artefacts of distinct schemes
  mutually unusable by construction.
- Implementations MAY record a scheme tag alongside the epoch in the
  device map to support future per-device scheme agility; under this
  MIP alone, all devices use the v1 scheme.

## Rationale

**R1. An asymmetric signature scheme, not a hash preimage.** A
hash-preimage gate is cheaper in-circuit — one hash evaluation
against three curve operations and a hash evaluation — and the
account-custody prototype used it as a placeholder for exactly that
reason. It was rejected as the standard because the preimage *is* the
credential, and that identity has three costs the signature scheme
removes:

| Property | Hash preimage | Schnorr over JubJub (this MIP) |
|---|---|---|
| Witness content | the long-term secret itself | a one-call signature |
| Threshold / MPC custody | structurally excluded — the whole secret must be assembled as one witness | native — FROST produces the same `(R, s)` from shares (section 7) |
| Delegated proving (MPS-0004) | delegate holds the account credential | delegate can execute at most the one approved call (S6) |
| Signer needs Midnight stack | prover and credential-holder are the same party | signer needs JubJub and SHA-256 only (5.3) |
| Compromise via proving environment | permanent — secret exposed on every call | bounded — key never enters the proving path (AUTH-4) |
| In-circuit cost | one hash | three native JubJub ops and one hash (R2, R3) |

The last row is the price and the first five are what it buys. The
custody MIP's seam semantics were written so that either scheme
satisfies them; this MIP standardises the signature scheme because the
account standard is precisely where threshold custody (MPS-0018,
MPS-0006, MPS-0016, MPS-0024) and proving delegation (MPS-0004) attach.

**R2. JubJub, not Ed25519 or secp256k1.** The candidate signature
schemes differ not in their algebra — all are Schnorr-family over a
prime-order group — but in what their group arithmetic costs inside a
BLS12-381 circuit:

| Curve | In-circuit arithmetic | Threshold story | Status |
|---|---|---|---|
| JubJub | native — its base field is the BLS12-381 scalar field; `ecMul`, `ecAdd`, `ecMulGenerator` are Compact built-ins | FROST over any prime-order group; existing library support (Implementation) | validated on devnet |
| Ed25519 | non-native — base field `2^255 − 19` requires emulated field arithmetic, hundreds of constraints per field operation | FROST (RFC 9591 ciphersuite) | rejected: cost |
| secp256k1 (ECDSA) | non-native — the demand exists precisely because it is absent (MPS-0010) | ECDSA thresholdisation is substantially more complex than Schnorr's | rejected: cost and threshold complexity |
| Ledger-scheme caller identity (stdlib proposal) | n/a — verified outside the circuit | bound to a single ledger key; no account policy; cannot authenticate shielded callers | rejected: wrong attachment point (custody MIP §4) |

The curve-economics assessment was confirmed in cryptographer review
during the feasibility phase, and the JubJub path was then validated
empirically: the reference experiment verifies the section 4 equation
in a Compact circuit on a devnet node, through the standard proof
server, at interactive latencies. This MIP does not modify or compete
with MIP-0003, which governs ledger-level wallet keys (Schnorr and
ECDSA over secp256k1, verified by the node): those keys authenticate
transactions, this scheme authenticates *account operations* inside
circuit logic, and MIP-0003 itself places JubJub as the in-circuit
curve.

**R3. The persistent hash for the challenge — not BLAKE2b, and not the
cheaper transient hash.** Standard RedJubjub hashes its challenge with
BLAKE2b-512, which Compact circuits cannot evaluate; the challenge
must be recomputed *inside* the circuit — that is what binds the
signature to the call — so the scheme must use a hash the language
exposes, and there are two. `transientHash` is the circuit-optimised
one: an algebraic (Poseidon-family) compression to a field element,
markedly cheaper in-circuit than `persistentHash`, which is SHA-256
over the compiler's field-aligned binary encoding. This MIP
nevertheless specifies `persistentHash`, because the two differ in
their stability contract, and the stability contract is what a
standard is made of: the toolchain documents `transientHash` as *not
guaranteed to persist between upgrades* and unfit for deriving state
data, and `persistentHash` as guaranteed consistent between versions.
Three considerations make that decisive. Device commitments are
ledger state, where the custody MIP's hash discipline (its S6)
already requires cross-version stability — the account-custody
prototype used `transientHash` for its commitments and records
non-survival of a toolchain upgrade as an accepted prototype-only
caveat. A deployed account's circuit is frozen at deploy while its
signers are rebuilt continually, so a challenge hash that drifted
with the toolchain would strand every signer built after an upgrade.
And the signing side of this scheme is meant to be implemented
independently — in HSMs, in threshold committees, in codebases with
no Midnight toolchain at all — which requires a challenge function
with a fixed public definition; "whatever the current compiler emits"
cannot be profiled into a FROST ciphersuite or put in front of a
reviewer. SHA-256 is also the most widely available primitive in
exactly the constrained signers R5 contemplates. The price is paid
in-circuit, where SHA-256 costs far more than an algebraic hash; the
reference experiment carries that cost through a production proof
server at interactive latencies, so the trade is affordable, and the
curve arithmetic — the part JubJub was chosen for (R2) — is
unaffected. Should the toolchain later expose a version-pinned
algebraic hash, adopting it is a successor scheme under section 10:
cheaper verification, same architecture. The substitution of SHA-256
for BLAKE2b is sound in principle — Fiat–Shamir requires a
collision-resistant hash binding over the preimage — but it makes the
scheme a variant rather than an implementation of a deployed
standard, so independent cryptographer review of the challenge
construction (including the grinding of 5.2 and the scalar-domain
alignment) is an acceptance criterion, not a formality. The grinding
loop costs the signer ~17.5 hash evaluations in expectation,
negligible on any hardware; the alternative — reducing the hash
modulo `r_J` in-circuit — was rejected because the reduction is
itself circuit logic while grinding costs the circuit nothing.

**R4. A dedicated `auth_nonce`, not `round`.** The custody MIP's
`round` counter advances on *every* state-changing call, including
permissionless deposits and inbox appends. A signature bound to `round`
would be invalidated by any third-party deposit that lands between
signing and submission — for a threshold device, between the start of
a signing ceremony and the transaction's acceptance — turning
permissionless deposits into a griefing surface against authorisation
liveness. `auth_nonce` advances only on gated calls, so only the
account's own authorised activity can race a pending signature. Safety
is unaffected: `auth_nonce` is contract state the gated call itself
advances, satisfying the custody MIP's freshness requirement, and
AUTH-2 gives at-most-once execution.

**R5. Approval separated from proving.** Binding authorisation to a
verified signature rather than to witness-knowledge of a secret splits
the transaction pipeline into two roles with distinct trust
requirements. The *approver* holds key material and produces `(R, s,
grind_nonce)`; it needs JubJub arithmetic and SHA-256 and nothing
else — no node, no indexer, no prover, no contract artefacts — so it
can be an HSM-like device, a TEE, an air-gapped machine, or an MPC
committee, and it can render the operation for human review before
signing, in the manner of a hardware wallet. The *prover* holds the
signature and the circuit's other witnesses and produces the
transaction; by AUTH-2 and AUTH-3 the worst it can do with the
signature is execute the exact approved call once, or drop it. This is
the property that makes the scheme MPC-compatible (the committee is
just an approver), audit-friendly (approval is an explicit artefact a
control framework can gate, log, and require quorums for — the shape
MPS-0024 asks for), and compatible with delegated proving (MPS-0004)
without the delegate becoming a key custodian. The two reference
signers demonstrate the boundary concretely: the signing side runs as
a standalone binary with no Midnight dependency beyond the curve and
hash.

**R6. 1-of-n at the contract, thresholds inside a device entry.** The
contract-surface policy is deliberately minimal: any active device
authorises. Rich policy (m-of-n across entries, per-operation limits,
scoped grants) is layered, not baked in, for the same reason the
custody MIP kept the seam abstract: policy is where requirements
diverge — personal accounts, institutional custody, and agent identity
(MPS-0015) will not want the same one — and the FROST profile already
provides thresholds where they are cryptographically strongest, behind
one key, invisible to the ledger and to observers. An on-ledger m-of-n
extension remains possible against the same seam without revising this
standard.

**R7. Per-device keys, not derivation.** Deriving device keys from a
common root would reintroduce the seed-shaped single point of failure
that MPS-0018's seedlessness goal excludes, and would make one device's
compromise a fleet event. Independent keys make `remove_device` and the
epoch mechanism meaningful: revocation removes exactly one credential,
and recovery invalidates all of them (AUTH-6, AUTH-7).

**R8. Rolling entries, not static commitments.** A static device
commitment of the form `H(tag, pk)` is a fixed pseudonym with three
unwanted properties: it is disclosed by the membership lookup of every
call the device authorises, so an account's whole history is groupable
by a constant; it is equal across every account in which the same key
is registered, so shared devices link accounts; and anyone holding a
candidate public key can test it against any account's state, at any
time, offline, forever. Binding the account address, the epoch, and a
per-device use counter into the entry and rolling the entry on every
call removes all three: nothing at rest identifies a device, no two
accounts ever hold an equal value, and testing a candidate key now
requires enumerating counters per account and epoch rather than
comparing one constant. The counter is per-device deliberately.
Binding the account-wide `round` would let any permissionless deposit
orphan every device entry permanently, a strictly worse variant of the
griefing R4 removes; binding `auth_nonce` would let each authorised
call orphan every other device's entry. A device's chain must advance
exactly when that device acts. What rolling does not provide is
unlinkability against an observer of the contiguous transaction
transcript: consumption and successor insertion happen in one call, so
that observer can chain a device's entries across calls (S4). The
claim is deliberately bounded. Rolling eliminates the static, offline,
and cross-account correlators at essentially no cost (one set removal
and one insertion in place of one map lookup), while full per-call
unlinkability requires the membership-proof-over-accumulator extension
S4 contemplates, at materially higher circuit cost.

The entries are hashed with `persistentHash`, and the cost is
acknowledged: a gated call now evaluates it twice (the consumed entry
and its successor) in addition to the challenge hash, and SHA-256 is
the expensive primitive in-circuit by a wide margin. `transientHash`
would remove most of that cost, but absent a version-pinned algebraic
hash it cannot be trusted for anything that outlives a compilation:
the toolchain documents its output as not guaranteed to persist
between upgrades. Device entries rest in ledger state while an
account's circuits are replaceable in place through contract
maintenance; a replacement circuit compiled with a toolchain whose
`transientHash` had drifted would orphan every registered device in a
single upgrade, recoverable only through the recovery seam. Entries
are also recomputed off-circuit, by continually rebuilt client
runtimes, for the rescan and lost-device removal of S11. The
account-custody prototype hashed its device commitments with
`transientHash` and records the non-survival of a toolchain upgrade as
an accepted prototype-only caveat; a standard cannot inherit that
caveat. R3's stability argument therefore applies to the entries with
more force than to the challenge, and the same relief applies: should
the toolchain expose a version-pinned algebraic hash, adopting it for
both is a successor scheme under section 10.

**R9. The challenge pins the private state, not only the arguments.**
The private state is not consensus data: it is a mutable, client-held
document (JSON in current client stacks) from which witness invocations
draw their values, and the proving environment reads it and can edit
it. Delegated and remote proving (MPS-0004) make that environment a
distinct trust domain, and even a local proving stack is a larger
attack surface than a signing device. A challenge that covered only
the circuit, its arguments, and the nonce would let such an
environment keep the signature valid while substituting what the call
actually consumes: a different coin selected for an approved
withdrawal, an altered stored value steering a branch. The approver
would have authorised a call shape; something else executes. Binding
the witness values closes the gap by construction: the circuit
recomputes the challenge over the values it actually consumes, so any
substitution diverges the challenge and no satisfying assignment
exists: the prover's discretion reduces to executing exactly the
approved call against exactly the approved state, or dropping it (S6).
The consequence for R5's division of roles is deliberate: the approver
receives the full preimage, witness values included, so approval is
informed in the hardware-wallet sense (what is reviewed is what
executes), while the prover contributes proving capacity and nothing
else. The cost is preimage growth: SHA-256 input proportional to the
witness data a circuit consumes, on the same accounting as R8;
custody-class circuits consume few, small witness values, and a
circuit whose witness data is large pays proportionally and should
weigh a digest-carrying design under a successor scheme.

## Path to Active

### Acceptance Criteria

- [ ] MIP number assigned by an editor; listed alongside the custody
      MIP in MPS-0018's header as a proposed solution; `Requires`
      updated with the custody MIP's assigned number.
- [ ] Independent cryptographer review of the signature scheme —
      the SHA-256 Fiat–Shamir substitution (R3), challenge grinding
      and scalar-domain alignment, subgroup and point-validation
      requirements, and nonce-generation guidance — with findings
      addressed.
- [ ] Domain-separation tags (`midnight:account:device:v1`,
      `midnight:account:auth:v1:*`) registered under the MPS-0027
      registry once it ratifies.
- [ ] A specified and reviewed FROST ciphersuite for JubJub with the
      section 5 challenge (section 7), with a demonstration that a
      `t`-of-`n` committee's aggregated signature verifies against an
      unmodified conforming contract.
- [ ] A public reference implementation passing the conformance suite
      (Testing section), integrated with a custody-MIP-conforming
      contract (both seams in one deployment).
- [ ] Cross-implementation interoperability: a signature produced by
      an independent second signer implementation accepted by the
      reference contract (the TypeScript and Rust reference signers
      already demonstrate this pairwise).
- [ ] Deployment exercised on a public test network.

### Implementation Plan

1. Extract the reference contract and the two reference signers from
   the validation experiments into a reference implementation against
   the custody MIP's contract surface.
2. Commission the cryptographer review; fold findings into the
   specification before editor numbering if substantive.
3. Specify the FROST ciphersuite with the threshold-library maintainers
   and run the committee demonstration.
4. Register domain-separation tags when the MPS-0027 registry lands.
5. Submit for editor numbering; iterate through community commentary.
6. Author the recovery-paths MIP against the section 8 seam.

## Backwards Compatibility Assessment

This MIP introduces a new contract standard. It requires no ledger,
consensus, or node change and no hard fork: every mechanism used
(in-circuit JubJub arithmetic, in-circuit hashing, contract state,
witness-supplied arguments) exists in the current stable network
protocol. It composes with, and requires, the custody MIP; deployments
of the custody surface under other seam instantiations (including
hash-preimage placeholders) remain valid custody-MIP citizens but do
not conform to this standard. MIP-0003 wallet keys and address
derivation are unaffected. Wallets unaware of this standard are
unaffected.

## Security Considerations

- **S1. Nonce reuse and bias.** Reusing the Schnorr nonce `r` across
  two distinct challenges reveals the private key algebraically, and
  biased nonces admit lattice recovery. Signers MUST follow section
  5.3: unique, unpredictable nonces, with hedged deterministic
  derivation recommended. Threshold devices inherit FROST's
  nonce-commitment discipline (RFC 9591), whose one-time nonces MUST
  NOT be reused across signing sessions.
- **S2. Point validation.** JubJub has cofactor 8. `pk` and `sig_r`
  enter the circuit as `JubjubPoint` values; conforming contracts MUST
  ensure both are validated on-curve, and the review under Path to
  Active MUST confirm the subgroup semantics of the built-in operations
  — in particular that a small-order or mixed-order component in
  `sig_r` or a registered `pk` cannot make the section 4 equation
  accept a signature the key holder did not produce. `add_device`
  SHOULD reject keys outside the prime-order subgroup.
- **S3. Non-standard challenge hash.** The SHA-256 Fiat–Shamir
  substitution (R3) means this scheme's security argument does not
  inherit RedJubjub's deployment history. The cryptographer review is
  the control; until it concludes, the scheme's status is
  experimentally validated, not reviewed.
- **S4. Device linkability.** Step 1 (section 4) discloses the entry
  consumed and the successor inserted. Both are single-use (AUTH-9),
  so the ledger carries no stable per-device pseudonym: an observer of
  state snapshots, an observer of other accounts, and an adversary
  testing a candidate public key all lose their correlator (R8). What
  remains is transcript chaining: the two values appear in one call,
  so an observer of the contiguous transaction history can still link
  one device's calls into a chain, and the set's cardinality still
  discloses the active device count. The residual leak is bounded as
  before, revealing which handle acted and never the operation's
  contents, on top of the custody MIP's activity metadata (its S4). A
  membership proof over a commitment accumulator would remove
  transcript chaining as well and is a permitted extension.
- **S5. Key compromise.** Any active device is sufficient to authorise
  (1-of-n), so a compromised device is full compromise of asset
  release until removed. The response surface is `remove_device` from
  a surviving device, the epoch bump on recovery (AUTH-6), and — for
  holdings warranting it — a threshold device (section 7), which turns
  single-device compromise into a `t−1`-share non-event. Note the
  asymmetry with the custody MIP's S2: the encryption secret leaks
  reading, a device key leaks spending; the two are deliberately
  different secrets.
- **S6. Delegated proving is bounded, not trustless.** A prover given
  a signature can execute the approved call or withhold it, and it
  sees the circuit's other witnesses — for a shielded withdrawal, the
  coin description being spent. Signature-based authorisation removes
  the *credential* from the proving environment (AUTH-4), and the
  private-state binding removes its discretion (AUTH-10): editing the
  witness store before proving changes a covered value, the in-circuit
  challenge diverges, and no valid proof exists. What remains is
  visibility (the prover is not blind) and liveness (it can withhold
  the call). Delegation choices should weigh MPS-0004's analysis.
- **S7. Add-device channel.** `add_device` grants full authority to
  the holder of the added key. The gated circuit authenticates *who
  authorised the addition*, not *whose key is added*; clients MUST
  present the new key's provenance to the approving device
  out-of-band (section 6). A poisoned addition is recoverable by
  `remove_device` or epoch bump, minus whatever it authorised
  meanwhile.
- **S8. Signer-side channels.** Scalar multiplication and nonce
  handling on the signer MUST be constant-time in secret inputs, per
  standard Schnorr practice. The grinding loop of 5.2 iterates a hash
  over signer-known data; its iteration count is independent of the
  private key and leaks nothing about it, and its expected ~17.5
  SHA-256 evaluations are negligible even on constrained hardware,
  where SHA-256 support is near-universal.
- **S9. Cross-context replay.** AUTH-2 and AUTH-3 close replay within
  and across accounts and circuits. Residual contexts are deployments
  sharing a contract address across networks; the network-identifier
  binding of section 5.1 is the mitigation and SHOULD be adopted
  wherever multi-network deployment is plausible.
- **S10. Toolchain hazards.** The reference experiment exposed a
  compact-runtime defect in which `JubjubPoint` equality compiled to
  reference rather than structural equality, silently breaking the
  section 4 comparison off-circuit; it is reported upstream and worked
  around in the experiment. Implementations MUST include the negative
  conformance tests (Testing 2) that catch a verifier which accepts
  everything or rejects everything, and MUST pin toolchain versions
  whose hash outputs are stable, per the custody MIP's S6.
- **S11. Counter tracking and desync.** The use counter is client
  state: it appears in no readable ledger value (only inside hashes),
  so a device that loses it cannot form its next entry. Recovery is a
  rescan: recompute entries for counters 0, 1, 2, … under the current
  epoch and test membership against the set (or walk the account's
  transcript); the chain length equals the number of calls the device
  has authorised, so the search is short and over the client's own
  data. Two sessions signing for the same device race on one entry;
  the loser fails step 1 with no state change and re-syncs. Clients
  MUST maintain, per account, the device roster (public keys and
  last-known counters): it is what keeps `remove_device` executable
  against a lost device (section 6) and what prevents duplicate
  registration of an active key.

## Implementation

- **Evidence base.** The Schnorr-wallet experiments in the Midnight
  Passport workspace validate the scheme end to end against a devnet
  node: an in-circuit verifier (Compact, the section 4 equation with
  the section 5 challenge shape) gating token release from a
  contract, a TypeScript client, and two independent signers — a Rust
  CLI over the contract's curve parameters, and a pure-Rust client
  built directly on the `midnight-ledger` crates (`midnight-curves`)
  that reproduces key derivation, challenge hashing, signing, and
  verification with no TypeScript, WASM, or npm dependency. The two
  produce interchangeable signatures against the same deployed
  verifier, demonstrating AUTH-4's client-agnostic signing boundary.
  The experimental verifier binds a transaction counter with the
  semantics of `auth_nonce`; it predates this specification and
  differs in known ways (single owner key, no device set or epochs,
  no per-circuit tags, no contract-address binding).
- **Seam fit.** The account-custody prototype validates the device-set,
  epoch, and lifecycle surfaces under a hash-preimage placeholder
  expressly structured so that this MIP's verifier replaces a single
  internal circuit; the custody MIP's section 4 was specified against
  that seam. The prototype's device set is a static-commitment map;
  the rolling entry set of section 3 supersedes it and is exercised by
  the reference implementation, not by the prototype.
- **Threshold path.** An existing audited threshold-signatures library
  (MIT-licensed) implements FROST-style RedDSA signing over JubJub with
  distributed key generation; the FROST profile of section 7 requires
  profiling its ciphersuite to the section 5 challenge (Path to
  Active). Committee operation, participant authentication, and share
  lifecycle are that deployment's concern, not this standard's.
- This MIP contains no implementation code; the artefacts above are
  its evidence base, and the reference implementation is an acceptance
  criterion, to be linked at its public location on submission.

## Testing

Conformance is demonstrated by a suite exercising, against a real node:

1. **Authorisation happy path**: a gated call with a valid signature
   from an active device executes; `auth_nonce` and `round` advance
   (AUTH-1, AUTH-2).
2. **Rejection matrix**: the same call aborts, with no state change,
   under each single fault — wrong `sig_s`; `sig_r` for a different
   challenge; unregistered `pk`; `pk` registered under a stale epoch;
   wrong `use_counter` with an otherwise-valid signature; an entry
   already consumed by a prior call; tampered argument with an
   otherwise-valid signature; a substituted witness value (an edited
   private state, e.g. a different coin selected than the one signed
   over) with an otherwise-valid signature; stale `auth_nonce`; reused
   signature after a successful call (AUTH-1, AUTH-2, AUTH-3, AUTH-6,
   AUTH-9, AUTH-10).
   This matrix is the guard against a vacuously accepting verifier
   (S10).
3. **Cross-account replay**: one device key registered in two
   accounts; a signature for account one is rejected by account two
   (AUTH-3).
4. **Cross-circuit replay**: a signature for one gated circuit is
   rejected by another gated circuit with an identical argument list
   (AUTH-3, per-circuit tags).
5. **Deposit independence**: a permissionless deposit lands between
   signing and submission; the pending authorisation still executes
   (AUTH-8).
6. **Lifecycle**: add device, authorise from it repeatedly (entry
   rolls each time), remove it by presenting its current entry,
   observe rejection; last-device removal fails; epoch bump
   invalidates all prior devices in one step (AUTH-5, AUTH-6).
   Additionally: a client with no local counter state re-derives its
   current counter by the rescan of S11 and then authorises
   successfully; the ledger transcript of the run contains no repeated
   device handle (AUTH-9).
7. **Cross-implementation signing**: a signature produced by an
   independent signer implementation, exercising its own hash and
   curve stack, is accepted (AUTH-4's boundary; already demonstrated
   pairwise by the reference signers).
8. **Threshold signature**: an aggregated `t`-of-`n` FROST signature
   verifies against an unmodified contract (section 7; acceptance
   criterion).
9. **Non-exfiltration audit**: the proving pipeline's inputs for every
   gated call are enumerated and contain no device private key or key
   share (AUTH-4).
10. **Bootstrap**: activation with the committed `(pk, salt)` succeeds
    and installs the entry at epoch 0 and use counter 0; a second
    activation aborts; an activation whose arguments do not match the
    commitment aborts (section 3).

## References (Optional)

- [MPS-0018](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0018-asset-custody-model.md):
  Multi-key Account Custody for Midnight-Native Assets (the parent
  problem statement, including the proving-boundary analysis).
- Contract Custody of Midnight-Native Assets (the custody MIP; number
  pending): the asset surface and the seam semantics this MIP
  instantiates.
- [MPS-0024](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0024-custodian-safe-shielded-spends.md):
  custodian-safe shielded spends (proof-authorised versus
  signature-authorised custody); MPS-0006, MPS-0016, MPS-0025: the
  companion institutional custody problem statements.
- [MPS-0004](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0004-trusted-proof-serving.md):
  trusted proof serving (the delegated-proving trust problem R5 and S6
  address).
- [MPS-0010](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0010-sig-verify-ecdsa.md)
  and
  [MPS-0011](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0011-native-crypto-primitives.md):
  the upstream demand for in-circuit signature verification and native
  crypto primitives.
- [MPS-0027](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0027-domain-separation.md):
  Domain Separation for Midnight Hash Constructions (tag registry).
- MIP-0003: ECDSA support (ledger-level wallet keys; positions JubJub
  as the in-circuit curve); MIP-0008: CAIP-2 network identifiers (the
  network binding of 5.1); MIP-0011: Native Shielded Token Standard
  (the `ownPublicKey()` caller-authentication prohibition).
- [RFC 9591](https://www.rfc-editor.org/rfc/rfc9591): The Flexible
  Round-Optimized Schnorr Threshold (FROST) Protocol; RFC 6979:
  deterministic nonce generation; RFC 2119: requirement levels.
- The Jubjub curve specification (Zcash protocol specification,
  §5.4.9.3) for curve parameters, subgroup order, and cofactor;
  RedDSA/RedJubjub (ibid., §5.4.7) as the BLAKE2b-challenge relative
  of this scheme.
- Upstream caller-identity thread: the token-standards
  [discussion](https://github.com/midnightntwrk/midnight-improvement-proposals/discussions/142)
  and the stdlib caller-identity
  [proposal](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/213);
  the `ownPublicKey()` intent
  [clarification](https://github.com/LFDT-Minokawa/compact/issues/283)
  and the `JubjubPoint` equality
  [defect](https://github.com/LFDT-Minokawa/compact/issues/278) (S10).
- Midnight Passport workspace: the Schnorr-wallet validation
  experiments (`experiments/redjubjub-wallet/`,
  `experiments/redjubjub-wallet-rs/`) and the account-custody
  prototype (`experiments/account-custody-prototype/`); to be linked
  at their public locations on submission.

## Acknowledgements

The IOG Advanced Research and Creativity department and the Midnight
Foundation. The curve-economics assessment underpinning R2 was
confirmed in review by Jesus Diaz Vico.

## Copyright Waiver

This document is licensed under the Apache License, Version 2.0, and its
contribution is made under the Midnight Foundation Contributor License
Agreement.

Portions of this document were drafted with the assistance of a large
language model. The named authors reviewed all content and are solely
accountable for it.
