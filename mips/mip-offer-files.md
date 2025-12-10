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

Offer files are encoded using a [bech32[(https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki) system. This must include a HRP of "zswapoffer", a separator "1", followed by the encoding of the binary format (see below) using bech32's data part encoding, and the checksum as defined in bech32.

### Binary Format

Discussion: How to specify the binary format? Easiest is the already-existing ledger definitions but that is specific to one implementation. There should be some kind of binary standard for on-chain types. To avoid changing formats breaking we could prepend a version number to the binary format exported.

## Rationale

Format requirements:
* Easily shareable across a wide variety of mediums
* Reasonably concise
* Not require execessive changes to the standard if zswap when avoidable
* Ideally human recognizable

// TODO: verify with final implementation - this is based off of the midnight_serialize::Serializable format (to_binary_repr returns 0 always?)

For these reasons it was decided to go with a bech32-style encoding. This allows it to be easily shareable in any ASCII text format (discord, twitter, etc) as well as having the potential to even be built into a QR code if that need ever arrises. For typical single input/output offers are less than 3000 alphanumeric characters. (TODO: verify this with real offers)

## Backwards Compatibility Assessment

This MIP only specifies a standard for encoding offer files off-chain and does not require any changes to the core protocol. Changes in the protocol itself, specifically changes to the ledger's offer format, could however necesitate changes to this MIP.

## Security Considerations

There should not be any security issues introduced as any modification to any parts of the offer format will invalidate the proofs rendering them useless. Any human reading of the format will require parsing via a tool to see which assets are to be swapped as the raw text will only show that is an offer file and can't be misrepresented.

## Implementation

TODO

## Testing

TODO

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0
