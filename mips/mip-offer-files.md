---
MIP: (assigned by a MIP Editor) \
Title: Offer Files \
Authors: rooooooooob \
Status: Proposed \
Category: Standards \
Created: 2025-12-09
---

This is still an early draft

## Abstract

This MIP proposes a standardized format for sharing zswap offers off-chain between users.

## Motivation

Zswap offers provide a way to exchange assets without having to involve a DEX or sending the tokens anywhere. Users can trustlessly share an imbalanced offer (e.g. 5 token A for 3 B) and it can be merged together with 1 or more other offers such that the end transaction is balanced. This process is non-interactive as the offers contain proofs for the partial transactions. As this is natively supported at the protocol level, having a standardized format for sharing this across a variety of mediums will help users take advantage of it with far less friction. This use-case is partially based on the offer files used on the Chia blockchain.

## Specification

Offer files are encoded using a [bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki) system. This must include a HRP of "zswapoffer", a separator "1", followed by the encoding of the binary format using bech32's data part encoding, and the checksum as defined in bech32. The binary format is simply the byte representation of the `zswap::Offer` type from the rust ledger implementation serialized using the `rust-serialize` crate from the ledger. The bech32 checksum uses the original bech32 checksum values.

## Rationale

Format requirements:
* Easily shareable across a wide variety of mediums
* Reasonably concise
* Not require execessive changes to the standard if zswap when avoidable
* Ideally human recognizable

// TODO: verify with final implementation - this is based off of the midnight_serialize::Serializable format (to_binary_repr returns 0 always?)

For these reasons it was decided to go with a bech32-style encoding. This allows it to be easily shareable in any ASCII text format (discord, twitter, etc) as well as having the potential to even be built into a QR code if that need ever arrises. For typical single input/output offers are less than 3000 alphanumeric characters. (TODO: verify this with real offers)

Base64 encoding was also considered the bloat from using bech32 over it is only around 20% (6 bits per byte vs 5) and includes the useful prefix to help identify it as well as a checksum. The checksum's fixed size degrades its usefulness compared to with short length addresses it was designed for, but users are also not going to be manually entering it like they might with an address, so our error detection requirements are much lower. Another option would be to add our own prefix and basic checksum to a base64 payload but then we lose on the abiltiy to double-click to select in most environments.

The binary format was chosen due to the lack of any existing binary spec for on-chain ledger types. While this isn't ideal, if there are any changes to the ledger implementation's serialization format, it will only invalidate offers between hard forks. This is due to the offers forming a part of transactions so any changes in this format would result in a fork.

We do not prepend a ledger version number to this binary data as in the worst case it simply will fail to parse if the format changed, but it might still parse after most updates.

## Backwards Compatibility Assessment

This MIP only specifies a standard for encoding offer files off-chain and does not require any changes to the core protocol. Changes in the protocol itself, specifically changes to the ledger's offer format, could however necesitate changes to this MIP.

## Security Considerations

There should not be any security issues introduced as any modification to any parts of the offer format will invalidate the proofs rendering them useless. Any human reading of the format will require parsing via a tool to see which assets are to be swapped as the raw text will only show that is an offer file and can't be misrepresented.

## Implementation

TODO

Sample rust implementation (with TODOs for now) (should this just be a link to a sample implementation?):
```rust
const OFFER_HRP: &str = "zswapoffer";

fn offer_to_bech32<P: Storable<InMemoryDB>>(offer: &Offer<P, InMemoryDB>) -> Result<String, bech32::EncodeError> {
    const OFFER_BYTES_REASONABLE_UPPER_BOUND: usize = 4096;
    use midnight_serialize::Serializable;
    
    let mut buf = BufWriter::new(Vec::with_capacity(OFFER_BYTES_REASONABLE_UPPER_BOUND));
    offer.serialize(&mut buf).unwrap(); // TODO: error conversion
    bech32::encode::<Bech32NoSizeCheck>(bech32::Hrp::parse_unchecked(OFFER_HRP), &buf.buffer())
}

fn offer_from_bech32<P: Storable<InMemoryDB>>(text: &str) -> Result<Offer<P, InMemoryDB>, bech32::DecodeError> {
    use midnight_serialize::Deserializable;

    let checked = bech32::primitives::decode::CheckedHrpstring::new::<Bech32NoSizeCheck>(text).unwrap(); // todo error handling
    if checked.hrp().as_str() != OFFER_HRP {
        // todo error
    }
    let mut read = std::io::Cursor::new(checked.data_part_ascii_no_checksum());
    let offer = Offer::deserialize(&mut read, midnight_serialize::RECURSION_LIMIT).unwrap(); // TODO error
    Ok(offer)
}
```

## Testing

TODO

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0
