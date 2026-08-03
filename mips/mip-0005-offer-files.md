---
MIP: 0005
Title: Offer Files
Authors: Andrew Fleming @andrew-fleming, Edward Alvarado <edward.alvarado@midnight.foundation>
Status: Proposed
Category: Standards
Created: 2026-02-25
Requires: none
Replaces: none
License: Apache-2.0
---

<!--
 This file is part of midnight-improvement-proposals.
 Copyright (C) 2025-2026 Midnight Foundation
 SPDX-License-Identifier: Apache-2.0
 Licensed under the Apache License, Version 2.0 (the "License");
 You may not use this file except in compliance with the License.
 You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS,
 WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 See the License for the specific language governing permissions and
 limitations under the License.
-->

## Abstract

This MIP specifies a standardized format for encoding and sharing Midnight offers off-chain between users.
An offer is a proven, intentionally *imbalanced* transaction:
it spends some assets and expects others in return,
and can be merged with one or more complementary offers to produce a balanced, submittable transaction.
The format uses bech32m encoding with a `swapoffer` human-readable prefix,
enabling offers to be shared as text strings across any ASCII-compatible medium.

This format generalizes different valid offers given they are **a proven Midnight `Transaction`**,
so it covers shielded offers and unshielded (UTXO) offers.

## Motivation

Midnight natively supports imbalanced transactions on both value layers:

- **Shielded** (Zswap): a `ZswapOffer` may input more of one token type than it outputs,
  leaving a per-token imbalance (`deltas`).
- **Unshielded** (UTXO): an `UnshieldedOffer` inside an intent may spend UTXOs of one token
  and create outputs of another.

A user can create such an imbalanced partial transaction (e.g., offering 5 TokenA in exchange for 3 TokenB)
that is later merged with complementary offers to produce a balanced, submittable transaction.
This process is non-interactive:
offers carry the proofs (shielded) and signatures (unshielded) for their own inputs and outputs.

Since this capability is native to the protocol,
a standardized encoding for sharing offers across a variety of mediums reduces friction
and enables interoperability between wallets, UIs, indexers, and tools.

This use case is inspired by the offer files used on the [Chia blockchain](https://docs.chia.net/wallets/offers).

## Specification

### What is encoded

The binary payload MUST be the canonical ledger serialization of a proven, imbalanced `Transaction` —
the top-level ledger transaction type.
The payload MUST be a *standard* transaction. Receivers verify this during validation (see Decoding).

A single `Transaction` is used because it is the only ledger type
that carries every kind of offer in one container:

| Leg | Field in the standard transaction (ledger type) | Segment | Value applied in phase |
| --- | --- | --- | --- |
| Shielded, guaranteed | `guaranteed_coins` (`ZswapOffer<Proof>`) | 0 (implicit) | guaranteed |
| Shielded, fallible | `fallible_coins[seg]` (`ZswapOffer<Proof>`) | ≥ 1 | fallible, but validity is checked in the guaranteed phase |
| Unshielded, guaranteed | `intents[seg].guaranteed_unshielded_offer` (`UnshieldedOffer<SignatureEnabled>`) | ≥ 1 (SDK: 1) | guaranteed |
| Unshielded, fallible | `intents[seg].fallible_unshielded_offer` (`UnshieldedOffer<SignatureEnabled>`) | ≥ 1 | fallible |

Encoding the whole `Transaction` (rather than a bare `ZswapOffer`) means shielded, unshielded offers all share one wire format and one HRP.
It also matches how the transaction is reconstructed by a taker,
who deserializes a full `Transaction` and merges/balances it.

### Encoding

MIP-0005 Offer files MUST be encoded using [bech32m](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki)
with the following parameters:

| Parameter | Value |
| --- | --- |
| Human-Readable Part (HRP) | `swapoffer` |
| Separator | `1` (standard bech32 separator) |
| Data Part | bech32m-encoded binary payload |
| Checksum | bech32m checksum (BIP-350) |
| Size Limit | bech32's standard 90-character limit MUST NOT be enforced |

The resulting string has the form:

```text
swapoffer1<bech32m-encoded-data><checksum>
```

> **On the HRP name.** The HRP is the layer-neutral `swapoffer`:
> the payload is a full `Transaction`,
> which may contain no zswap offer at all (a purely unshielded offer).
> Implementations MUST NOT assume the presence of a shielded component from the HRP alone.

### bech32m is a display encoding, not the canonical form

The canonical, authoritative representation of an offer is the **raw serialized bytes** of the `Transaction`.
bech32m exists purely to make those bytes safely shareable as ASCII text (messaging, files, clipboards).
Systems that store or transmit offers as opaque bytes, notably a data-availability layer (see MIP-0006), 
MUST use the raw bytes, not the bech32m string, to avoid the ~1.6× size expansion that bech32 imposes. 
The two are losslessly interconvertible: `decode(encode(bytes)) == bytes`.

### Binary Payload

The binary payload MUST be produced by the ledger's canonical transaction serialization:
`Serializable::serialize` (Rust) or `Transaction.serialize()` (TypeScript).
No tag prefix is added by this specification;
the ledger's own tagged serialization is used as-is.

Offer files MUST contain **authorized** offers only:

- Shielded components MUST carry valid zero-knowledge proofs
  (`ZswapOffer<Proof>`, not `ProofPreimage` or proof-erased).
- Unshielded components MUST carry the required signatures
  (`UnshieldedOffer<SignatureEnabled>`, not signature-erased).

Unproven, proof-erased, or unsigned transactions MUST NOT be encoded as offer files:
they cannot be verified or non-interactively merged by an untrusted taker.

### Decoding

To decode an offer file:

1. Verify the HRP is `swapoffer`.
2. Decode the bech32m data part and verify the checksum.
3. Deserialize the binary payload as a `Transaction` using `Deserializable::deserialize` (Rust)
   or `Transaction.deserialize('signature', 'proof', 'binding', bytes)` (TypeScript).

Implementations MUST reject strings with an incorrect HRP, invalid checksum,
or payloads that fail deserialization.

Decoding establishes only that the payload is a well-formed `Transaction`.
Semantic acceptance is the responsibility of the **receiver**: 
a user, an indexer, or any other system acting on the offer. 
Before acting, the receiver MUST verify that the transaction is a *standard* transaction, that its components verify (proofs, signatures), and that its backing inputs are still live on-chain. 
Systems that relay offers (see MIP-0006) may filter best-effort, but each receiver MUST validate independently. 
Note also that the format does not require the transaction to be imbalanced: a balanced transaction is a degenerate offer that could be submitted on its own, and a receiver may simply dismiss it.

### Encoding Constraints

- The encoded string MUST use only the bech32 character set: `qpzry9x8gf2tvdw0s3jn54khce6mua7l`.
- bech32 specifies a 90-character length limit designed for short address strings.
  Proven offer payloads are far larger (see Offer String Size).
  Implementations MUST NOT enforce the 90-character limit
  (e.g. use `Bech32m` in the `bech32` Rust crate,
  or `bech32m.encode(hrp, words, false)` / `bech32m.decode(str, false)` in the `@scure/base` JavaScript library).

### Offer lifetime

An encoded offer is a snapshot against ledger state and expires independently of this format.

- **Intent TTL.** Each intent carries a `ttl` timestamp that must satisfy
  `t_block ≤ ttl ≤ t_block + global_ttl` at apply time,
  where `global_ttl` is a network parameter.
- **Merkle-root recency.** Each shielded input's proof commits to a Zswap Merkle-tree root,  and the ledger accepts only roots it has seen within its root-recency window,  also a network parameter; older roots are rejected as unknown.

Implementations SHOULD surface the effective expiry to users (see also MIP-0006).

### Versioning

The payload format is versioned by the ledger's tagged transaction serialization:
a payload produced under one ledger serialization version fails to deserialize under an incompatible one, so there is no silent misinterpretation.


## Rationale

### Format Selection

bech32m produces ASCII-only strings shareable in any text context,
the `swapoffer` prefix makes strings immediately identifiable,
and the format is double-click selectable in most environments.
bech32m (BIP-350) is used rather than the original bech32 (BIP-173):
bech32 has a known checksum weakness for certain lengths,
and bech32m is the current recommendation for any new HRP-prefixed encoding.

**Alternatives considered:**

- **Base64**: ~20% more compact but lacks a built-in prefix and checksum.
  Contains `+` and `/` characters that break double-click selectability.
- **Hex**: Simple but doubles the payload size.

### Encoding a `Transaction` rather than a bare offer

The shielded-only revision encoded a `zswap::Offer<Proof>`.
That type cannot represent an unshielded offer,
and the TypeScript/WASM bindings do not expose a standalone serializer or `deltas` for `UnshieldedOffer`.
The top-level `Transaction` is the smallest type that
(a) serializes from both Rust and TypeScript,
(b) carries shielded and unshielded offers together,
and (c) is what a taker deserializes to merge and balance.
Encoding it keeps one HRP and one code path for all offer kinds.

### Serialization Format

The binary payload uses the ledger's native transaction serialization without an additional prefix.
This matches the serialization used by the Midnight WASM SDK's `Transaction.serialize()`,
ensuring cross-language compatibility between Rust and TypeScript.
If the ledger's serialization format changes, existing offer strings become unparseable —
but such a change is a hard fork,
so offer invalidation coincides with a network upgrade rather than causing silent incompatibility.

### Authorized Offers Only

Non-interactive merging requires that each side already carries its own authorization:
proofs on the shielded side, signatures on the unshielded side.
Unproven/unsigned offers would require the taker to have the maker's proving or signing capability,
breaking the non-interactive property.
Proof- or signature-erased offers cannot be verified,
making them unsuitable for sharing with untrusted parties.

### Offer String Size

A minimal proven shielded offer (1 input, 1 output) produces a binary payload
of approximately 10,000 bytes, dominated by zero-knowledge proofs
(measured empirically: ~4.9 KB per input proof and ~5.2 KB per output proof,
plus nullifiers, commitments, and deltas).
After bech32m encoding this yields a string of approximately 16,000 characters.
Every additional spent UTXO (e.g. change) adds roughly another 5 KB before encoding.
A useful approximation:

```text
bech32_chars ≈ (4.9·inputs + 5.2·outputs + 0.1) KB · 1.6
```

To keep offers small,
a maker MAY pre-merge their UTXOs into a single well-shaped input before constructing the offer,
yielding a ~10 KB (1-in/1-out) payload.
Keeping offers small is RECOMMENDED throughout:
publication cost on a data-availability layer is proportional to size (MIP-0006),
and receivers are free to cap the payload sizes they accept.
This format imposes no maximum size of its own,
but the merged transaction must ultimately fit within the network's transaction byte limit
(a ledger parameter, currently 1 MiB), which bounds offers in practice.

At this size, offer strings remain practical for messaging platforms, paste services, files,
and the discovery layer defined in MIP-0006.
They are too large for common QR codes (max ~4,296 alphanumeric characters).
QR extensions such as [Structured Append](https://segno.readthedocs.io/en/latest/structured-append.html)
and [HCC2D](https://qrcodecreator.com/hcc2d-code) exist but lack practical scanner support.
URI schemes are also impractical as offer strings exceed common URL length limits.
Applications needing compact sharing may use URLs pointing to hosted offer files instead.

### Checksum Suitability

bech32m's error-detection properties degrade for strings longer than 90 characters.
This is less concerning for offer files than for addresses
because a corrupted offer cannot cause loss of funds:
there is simply no valid offer to complete, so the transaction cannot be constructed.
The more realistic risk is a man-in-the-middle substituting one valid offer for another
(a different asset or worse terms);
the checksum is a first line of defense,
and parsers/UIs MUST still show the decoded terms for the user to confirm.

## Path to Active

### Acceptance Criteria

- Cross-language round-trip conformance (Rust ↔ TypeScript):
  byte-identical serialization for shielded and unshielded offers.
- At least two independent consumers of the format
  (e.g. the wallet SDK offer builder and a MIP-0006-compliant indexer)
  interoperate on the same encoded offers.
- The decoding rejection rules (HRP, checksum, deserialization)
  and receiver validation rules (standard transaction, authorization)
  are covered by conformance tests.

### Implementation Plan

1. Reference codec published as a small library in TypeScript and Rust
   (the reference implementations below).
2. Integration into the wallet SDK's offer-construction flow (see MIP-0006 dependencies).
3. Conformance tests added alongside the reference codec.

## Backwards Compatibility Assessment

This MIP specifies an off-chain encoding and requires no changes to the core protocol.

Relative to the shielded-only revision,
the payload type widens from `zswap::Offer<Proof>` to `Transaction`.
A decoder written for the old revision would fail to deserialize the new payload as a bare offer;
decoders SHOULD deserialize a `Transaction`.
Because the shielded-only revision was never deployed to a network,
no on-chain or persisted data is affected.

Changes to the ledger's `Transaction` serialization would invalidate previously encoded offer strings,
but would only occur during a hard fork.

## Security Considerations

### Tamper Resistance

Any modification to any part of the encoded payload invalidates the zero-knowledge proofs (shielded)
or signatures (unshielded) it contains, rendering the offer unusable.

### Human Readability

The raw encoded string does not reveal which assets are being swapped or in what amounts.
Users MUST use a compliant parser to inspect offer terms before accepting.

### Offer Validity

An offer becomes invalid when the maker spends a backing input
(shielded nullifier consumed, or unshielded UTXO spent),
when an intent TTL is reached,
or when the Merkle roots referenced by its shielded proofs age out of the ledger's recency window
(see Offer lifetime).
Verifying any of these requires checking against current ledger state.

Deserialization alone establishes nothing beyond well-formedness:
anyone can publish a malicious or stale offer,
so receivers MUST revalidate each component
(proof verification, signature verification, input liveness)
against current ledger state before acting on it.
A valid offer-file body is a necessary, not sufficient, condition for a takeable offer.

## Relationship to Other MIPs

This MIP defines the low-level encoding for an individual offer.
MIP-0006 (P2P Atomic Swaps) builds on it:
the raw bytes defined here are what a maker publishes to a data-availability layer,
and the bech32m string is what an indexer serves for display and discovery.

## Implementation

### Reference Implementation (Rust)

```rust
use bech32::{Bech32m, Hrp};
use midnight_serialize::{Deserializable, Serializable};
use midnight_storage::db::DB;
use midnight_transient_crypto::proofs::Proof;
use midnight_ledger::structure::Transaction;
use std::io::{BufWriter, Cursor};

const OFFER_HRP: &str = "swapoffer";

// Transaction<S, P, B, D>; concrete signature/proof/binding kinds elided.
fn offer_to_bech32<S, P, B, D: DB>(
    tx: &Transaction<S, P, B, D>,
) -> Result<String, OfferFileError> {
    let hrp = Hrp::parse_unchecked(OFFER_HRP);
    let mut buf = BufWriter::new(Vec::with_capacity(16384));
    tx.serialize(&mut buf).map_err(OfferFileError::Serialize)?;
    bech32::encode::<Bech32m>(hrp, buf.buffer()).map_err(OfferFileError::Encode)
}

fn offer_from_bech32<S, P, B, D: DB>(
    text: &str,
) -> Result<Transaction<S, P, B, D>, OfferFileError> {
    let checked = bech32::primitives::decode::CheckedHrpstring::new::<Bech32m>(text)
        .map_err(OfferFileError::Decode)?;
    if checked.hrp().as_str() != OFFER_HRP {
        return Err(OfferFileError::InvalidHrp(checked.hrp().as_str().to_string()));
    }
    let bytes: Vec<u8> = checked.byte_iter().collect();
    Transaction::deserialize(&mut Cursor::new(bytes), 0).map_err(OfferFileError::Deserialize)
}
```

> Note: `Transaction` derives its serialization via the ledger's `Storable`/tag system,
> which provides the `Serializable`/`Deserializable` impls used above.

### Reference Implementation (TypeScript)

```typescript
import { bech32m } from '@scure/base';
import { Transaction } from '@midnight-ntwrk/ledger-v8';

const OFFER_HRP = 'swapoffer';
const NO_LIMIT = false; // disable bech32's 90-char cap

function offerToBech32(tx: Transaction<'signature', 'proof', 'binding'>): string {
  return bech32m.encode(OFFER_HRP, bech32m.toWords(tx.serialize()), NO_LIMIT);
}

function offerFromBech32(text: string): Transaction<'signature', 'proof', 'binding'> {
  const { prefix, words } = bech32m.decode(text, NO_LIMIT);
  if (prefix !== OFFER_HRP) throw new Error(`Invalid HRP: expected '${OFFER_HRP}', got '${prefix}'`);
  const bytes = new Uint8Array(bech32m.fromWords(words));
  return Transaction.deserialize('signature', 'proof', 'binding', bytes);
}
```

## Testing

| Scenario | Expected Outcome |
| --- | --- |
| Encode and decode a proven shielded offer | Round-trip produces byte-identical transaction |
| Encode and decode a proven unshielded offer | Round-trip produces byte-identical transaction |
| Decode with wrong HRP | Rejected with HRP error |
| Decode with corrupted checksum | Rejected with checksum error |
| Decode with corrupted data part | Deserialization fails or proofs/signatures invalid |
| Decode with truncated string | Deserialization fails |
| Decode an unproven / unsigned payload | Rejected (not authorized) |
| Decode a non-standard transaction payload | Decodes, but receiver validation rejects it |
| Cross-language round-trip (Rust ↔ TypeScript) | Byte-identical serialization |
| Offer string length for single-input/output shielded swap | Approximately 16,000 characters |
| Double-click selection in common environments | Entire string selected |

## References

- [BIP-350: bech32m](https://github.com/bitcoin/bips/blob/master/bip-0350.mediawiki)
- [BIP-173: bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)
- [Chia offer files](https://docs.chia.net/wallets/offers)
- [`@scure/base`](https://github.com/paulmillr/scure-base) (JavaScript bech32m)
- [`bech32` Rust crate](https://crates.io/crates/bech32)

## Acknowledgements

This MIP supersedes an earlier draft (#39) that established the HRP-prefixed bech32 encoding approach,
and a subsequent shielded-only revision.
This revision generalizes the payload to a full `Transaction`
so the format covers shielded and unshielded offers,
and switches the reference encoding to bech32m.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement,
which includes the assignment of copyright for your contributions to the Foundation.
