---
MIP: (assigned by a MIP Editor)
Title: Offer Files
Authors: Andrew Fleming @andrew-fleming
Status: Proposed
Category: Standards
Created: 2026-02-25
Replaces: MIP-? (Offer Files draft)
---

## Abstract

This MIP specifies a standardized format for encoding and sharing zswap offers off-chain between users.
The format uses bech32 encoding with a `zswapoffer` human-readable prefix,
enabling offers to be shared as text strings across any ASCII-compatible medium.

## Motivation

Zswap offers provide a way to exchange assets without involving a DEX or custodian.
Users can create imbalanced partial transactions (e.g., offering 5 TokenA in exchange for 3 TokenB)
that can be merged with one or more complementary offers to produce a balanced, submittable transaction.
This process is non-interactive, as offers contain proofs for the partial transactions.

Since this capability is natively supported at the protocol level,
a standardized encoding format for sharing offers across a variety of mediums
reduces friction and enables interoperability between wallets, UIs, and tools.

This use case is inspired by the offer files used on the [Chia blockchain](https://docs.chia.net/wallets/offers).

## Specification

### Encoding

Offer files MUST be encoded using [bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki)
with the following parameters:

| Parameter | Value |
| --- | --- |
| Human-Readable Part (HRP) | `zswapoffer` |
| Separator | `1` (standard bech32 separator) |
| Data Part | Bech32-encoded binary payload |
| Checksum | Bech32m checksum (BIP-350) |
| Size Limit | Bech32's standard 90-character limit MUST NOT be enforced |

The resulting string has the form:

```text
zswapoffer1<bech32-encoded-data><checksum>
```

### Binary Payload

The binary payload MUST be the canonical ledger serialization of a proven zswap offer,
produced by calling `Serializable::serialize` (Rust) or `ZswapOffer.serialize()` (TypeScript)
on an `Offer<Proof>` value. No tag prefix is included.

Offer files MUST contain proven offers only.
Unproven and proof-erased offers MUST NOT be encoded as offer files.

### Decoding

To decode an offer file:

1. Verify the HRP is `zswapoffer`
2. Decode the bech32m data part and verify the checksum
3. Deserialize the binary payload as a proven zswap offer using `Deserializable::deserialize` (Rust) or `ZswapOffer.deserialize('proof', bytes)` (TypeScript)

Implementations MUST reject strings with an incorrect HRP, invalid checksum,
or payloads that fail deserialization.

### Encoding Constraints

- The encoded string MUST use only the bech32 character set: `qpzry9x8gf2tvdw0s3jn54khce6mua7l`.
- Bech32 specifies a 90-character length limit, designed for short address strings.
  Proven offer payloads are significantly larger (see Offer String Size in Rationale).
  Implementations MUST NOT enforce bech32's 90-character limit
  (e.g. use `Bech32m` in the `bech32` Rust crate or `bech32m.encodeNoLimit/bech32m.decodeNoLimit` in JavaScript bech32 libraries).

## Rationale

### Format Selection

Bech32 was chosen because it produces ASCII-only strings shareable in any text context,
the `zswapoffer` prefix makes strings immediately identifiable,
and the format can be double-click selected in most environments.
While proven offer strings are larger than ideal (see Offer String Size),
bech32 remains suitable as the encoding format for these reasons.

**Alternatives considered:**

- **Base64**: ~20% more compact but lacks a built-in prefix and checksum.
  Contains `+` and `/` characters that break double-click selectability.
- **Hex**: Simple but doubles the payload size.

### Serialization Format

The binary payload uses the ledger's native `midnight_serialize` serialization without a tag prefix.
This matches the serialization used by the Midnight WASM SDK's `ZswapOffer.serialize()`,
ensuring cross-language compatibility between Rust and TypeScript implementations.
If the ledger's serialization format changes, existing offer strings become unparseable,
but such changes would constitute a hard fork,
so offer invalidation would coincide with a network upgrade and not cause silent incompatibility.

### Proven Offers Only

This specification mandates proven offers because they are the only variant that enables non-interactive merging.
Unproven offers would require the taker to have access to the proving system,
breaking the non-interactive property.
Proof-erased offers can't be verified which makes them unsuitable for sharing with untrusted parties.

### Offer String Size

A minimal proven offer (1 input, 1 output, 1 delta) produces a binary payload
of approximately 10,000 bytes, dominated by zero-knowledge proofs
(4,832 bytes each for input and output proofs).
After bech32 encoding, this yields an offer string of approximately 16,000 characters.

At this size, offer strings remain practical for sharing via messaging platforms,
paste services, files, and the indexer API defined in the P2P Atomic Swaps MIP.
They are too large for common QR codes (which support a maximum of ~4,296 alphanumeric characters).
QR extensions such as [Structured Append](https://segno.readthedocs.io/en/latest/structured-append.html)
and [HCC2D](https://qrcodecreator.com/hcc2d-code) exist but lack practical scanner support.
URI schemes (e.g. `web+midnight://`) are also impractical as offer strings exceed common URL length limits.
Applications needing compact sharing formats may use URLs pointing to hosted offer files instead.
Multi-asset offers with additional inputs and outputs will be proportionally larger.

### Checksum Suitability

Bech32's error-detection properties degrade for strings longer than 90 characters.
This is less concerning for offer files than for addresses because a corrupted offer cannot result in loss of funds.
There is no valid offer to complete, so the transaction simply cannot be constructed.

The more realistic risk is a man-in-the-middle that substitutes one valid offer for another,
potentially swapping a different asset or less favorable terms.
The checksum provides a first line of defense against such substitution.
Validating that the offer content matches the user's expectations is a separate concern
that should be addressed by parsers and UIs.

## Backwards Compatibility Assessment

This MIP specifies a standard for encoding offer files off-chain and does not require any changes to the core protocol.

Changes to the ledger's `zswap::Offer` serialization format would invalidate previously encoded offer strings,
but would only occur during a hard fork.

## Security Considerations

### Tamper Resistance

Any modification to any part of the encoded offer invalidates the zero-knowledge proofs contained within,
rendering the offer unusable.

### Human Readability

The raw encoded string does not reveal which assets are being swapped or in what amounts.
Users MUST use a compliant parser to inspect offer terms before accepting.

### Offer Validity

An encoded offer becomes invalid when the maker spends the backing UTXO.
Offers also expire when their transaction TTL is reached.
In both cases, verification requires checking against current ledger state.

## Relationship to Other MIPs

This MIP defines the low-level encoding for individual offers.
The P2P Atomic Swaps MIP (MIP-?) builds on this foundation by specifying an application-layer payload,
discovery protocol, and optional authentication.
The `transaction` field in that MIP's `OfferPayload` contains a bech32-encoded offer as specified here.

## Implementation

### Reference Implementation (Rust)

```rust
use bech32::{Bech32m, Hrp};
use midnight_serialize::{Deserializable, Serializable};
use midnight_storage::db::DB;
use midnight_transient_crypto::proofs::Proof;
use midnight_zswap::Offer;
use std::io::{BufWriter, Cursor};

const OFFER_HRP: &str = "zswapoffer";

fn offer_to_bech32<D: DB>(
    offer: &Offer<Proof, D>,
) -> Result<String, OfferFileError> {
    let hrp = Hrp::parse_unchecked(OFFER_HRP);
    let mut buf = BufWriter::new(Vec::with_capacity(4096));
    offer.serialize(&mut buf)
        .map_err(OfferFileError::Serialize)?;
    bech32::encode::<Bech32m>(hrp, buf.buffer())
        .map_err(OfferFileError::Encode)
}

fn offer_from_bech32<D: DB>(
    text: &str,
) -> Result<Offer<Proof, D>, OfferFileError> {
    let checked = bech32::primitives::decode::CheckedHrpstring::new::<Bech32m>(text)
        .map_err(OfferFileError::Decode)?;
    if checked.hrp().as_str() != OFFER_HRP {
        return Err(OfferFileError::InvalidHrp(
            checked.hrp().as_str().to_string(),
        ));
    }
    let bytes: Vec<u8> = checked.byte_iter().collect();
    Offer::deserialize(&mut Cursor::new(bytes), 0)
        .map_err(OfferFileError::Deserialize)
}

#[derive(Debug)]
enum OfferFileError {
    Serialize(std::io::Error),
    Deserialize(std::io::Error),
    Encode(bech32::EncodeError),
    Decode(bech32::primitives::decode::CheckedHrpstringError),
    InvalidHrp(String),
}
```

### Reference Implementation (TypeScript)

```typescript
import { bech32m } from '@scure/base';
import { ZswapOffer, Proof } from '@midnight-ntwrk/ledger-v7';

const OFFER_HRP = 'zswapoffer';

function offerToBech32(offer: ZswapOffer<Proof>): string {
  const bytes = offer.serialize();
  return bech32m.encodeNoLimit(OFFER_HRP, bech32m.toWords(bytes));
}

function offerFromBech32(text: string): ZswapOffer<Proof> {
  const { prefix, words } = bech32m.decodeNoLimit(text);
  if (prefix !== OFFER_HRP) {
    throw new Error(`Invalid HRP: expected '${OFFER_HRP}', got '${prefix}'`);
  }
  const bytes = new Uint8Array(bech32m.fromWords(words));
  // 'proof' marker selects the proven offer variant for deserialization
  return ZswapOffer.deserialize('proof', bytes);
}
```

## Testing

| Scenario | Expected Outcome |
| --- | --- |
| Encode and decode a proven offer | Round-trip produces identical offer |
| Decode with wrong HRP | Rejected with HRP error |
| Decode with corrupted checksum | Rejected with checksum error |
| Decode with corrupted data part | Deserialization fails or ZK proofs invalid |
| Decode with truncated string | Deserialization fails |
| Decode an unproven offer payload | Deserialization fails |
| Decode a proof-erased offer payload | Deserialization fails |
| Cross-language round-trip (Rust ↔ TypeScript) | Byte-identical serialization |
| Offer string length for single-input/output swap | Approximately 16,000 characters |
| Double-click selection in common environments | Entire string selected |

## Acknowledgments

This MIP supersedes an earlier draft (#39) that established the bech32 encoding approach and `zswapoffer` HRP.
This MIP extends that work with proven-offer-only constraints
and reference implementations informed by ledger source analysis.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License,
Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement,
which includes the assignment of copyright for your contributions to the Foundation.
