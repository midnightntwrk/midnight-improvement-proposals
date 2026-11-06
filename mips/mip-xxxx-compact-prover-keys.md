---
MIP: X
Title: Smaller prover key files
Authors:
  - Giles Cope (gilescope)
Status: Draft
Category: Standards
Created: 2026-09-28
Requires: none
Replaces: none
MPS: MPS-0039
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

## Abstract

To use a Midnight contract privately, your device needs a "prover key" file
for each action the contract offers. These files are large: from a few
megabytes up to hundreds of megabytes each. This makes wallets and apps slow
to download and hard to run on phones and in browsers.

Most of each file is written in a wasteful way. This MIP defines a new way to
write the same file that is 10 to 77 times smaller on the keys we tested. It
turns back into the exact original file, byte for byte, in a few milliseconds.
Nothing on the chain changes.

## Terms

- **Proof:** a short piece of data that shows a transaction follows the
  contract's rules without showing the private details.
- **Prover key:** the file needed to make a proof for one contract action.
  Anyone can rebuild it from the contract, but that takes time and memory.
- **Verifier key:** the small file the chain uses to check a proof.
- **Encoding:** a way of writing data down as bytes. Two encodings can hold
  exactly the same information at very different sizes.

## Motivation

MPS-0039 describes the problem: to call a contract, a client must hold its
prover keys. The chain only stores the verifier key (kilobytes). The prover
key can be tens of thousands of times bigger.

Real sizes today:

| Where                                   | Prover key size        |
| --------------------------------------- | ---------------------- |
| Built-in shielded token actions         | 2 to 11 MB each        |
| Ten small `midnight-js` test contracts  | 384 MB in total        |
| A bridge and exchange deployment        | 45 to 570 MB each      |
| The same deployment, all 60 keys        | 10.4 GB                |

The files are already compressed with gzip. That does not help much, because
the waste is not the kind gzip can see.

The bridge and exchange figures come from a user report:
<https://github.com/midnightntwrk/servicedesk/issues/203>.

## Specification

### What is inside a prover key

Almost all of a prover key is two long tables of numbers. Every number is
stored as 32 bytes.

1. **The wiring table.** A proof works on a large grid of boxes. Some boxes
   must hold the same value as another box. For every box, this table says
   which box it is linked to. Most boxes are linked only to themselves.
   Each entry is stored as a 32-byte number that looks random. In fact it is
   just a scrambled form of a box position, which needs about 3 bytes.
2. **The constants table.** Fixed values that are part of the contract's
   rules. Most of them repeat: a table of hundreds of thousands of entries
   holds only a few thousand different values.

### The new encoding

The new encoding keeps the same information, written more simply:

1. **Wiring table:** unscramble each entry back to the box position it
   stands for. Store only how far it is from its own position. For a box
   linked to itself, that is 0.
2. **Constants table:** store each different value once, in a list. Then
   store, for each entry, its place in that list.
3. **Everything else** (the small header and contract description) is kept
   exactly as it is.

This MIP does not choose a compressor. The new file can be sent as it is,
or packed with any standard compressor, such as gzip (used today) or zstd.
Those compressors mark their own output, so a reader can always tell which
one was used. Inside, the new layout starts with its own marker, so a reader
can tell it apart from an old key.

Reading the file does the same steps in reverse. The result is the original
prover key, identical byte for byte. Anything that uses prover keys today
keeps working without change.

> Reviewer note: a byte-level layout (field order, number formats, version
> tag) must be added here before this MIP leaves Draft. The working spike in
> the References section defines one.

### Measured results

Tested on every built-in prover key for ledger versions 9 and 10. The new
layout is smaller than today's file even with no compressor at all:

| Key             | Today (gzip) | New, not packed | New + gzip | New + zstd | Decode time |
| --------------- | ------------ | --------------- | ---------- | ---------- | ----------- |
| Dust spend      | 2.18 MB      | 0.26 MB         | 0.031 MB   | 0.028 MB   | 3 ms        |
| Shielded sign   | 2.81 MB      | 0.64 MB         | 0.27 MB    | 0.25 MB    | 4 ms        |
| Shielded output | 5.73 MB      | 1.10 MB         | 0.28 MB    | 0.26 MB    | 10 ms       |
| Shielded spend  | 11.02 MB     | 1.91 MB         | 0.28 MB    | 0.26 MB    | 16 ms       |

With a compressor, the files are 10 to 77 times smaller than today.

Every file turned back into the exact original, byte for byte. Keys from
ledger version 6 use an older layout and were not tested.

> Reviewer note: these keys are 2 to 11 MB. The problem is worst for contract
> keys of 90 to 570 MB. Add measurements for large contract keys before this
> draft is shared.

## Rationale

**Why not just use a better compressor?** We tried. A strong modern
compressor (zstd at its highest setting) saves only 2 to 4% more than the
gzip used today. The numbers look random at the byte level, so no general
compressor can find the pattern. You have to know what the numbers mean.

**Why not rebuild the key from the contract instead?** It is possible to send
only the contract description (a few kilobytes) and rebuild the prover key on
the receiving device. That makes the download even smaller. But rebuilding
costs far more time and memory:

| Approach                | Download        | Work on the device            |
| ----------------------- | --------------- | ----------------------------- |
| Today                   | full size       | none                          |
| This MIP                | 10 to 77 times smaller | a few milliseconds     |
| Rebuild from contract   | kilobytes       | 9 seconds and 2.3 GB of memory for a 224 MB key, more for bigger keys |

A phone or browser cannot afford that memory. Browsers also cannot rebuild
keys today, because the browser build of the tools does not include that
function. Rebuilding remains a good option for servers, and it can be added
later next to this MIP. The two do not conflict.

**Why not make the contracts smaller?** Other proposals (MPS-0010, MPS-0011,
MPS-0008) aim to make some contract actions cheaper by adding built-in
cryptography. That makes keys smaller at the source, and it is good work.
This MIP works on any key, today, and the savings stack: a smaller key also
gets smaller with this encoding.

**Why not keep the keys on the proof server?** Today a client sends the full
prover key to the proof server with every proof. A proof server that keeps
its own copy of the keys removes that upload (see `midnight-ledger` pull
request 770). That helps clients that use a proof server, and this MIP makes
the server's own copy smaller to fetch and store. Clients that make proofs
themselves still need the key, so the two work well together.

## Path to Active

### Acceptance Criteria

- The byte-level format is specified in this document.
- The proof server and `midnight-js` can read keys in the new encoding.
- The compiler can write keys in the new encoding.
- Round-trip tests pass on every built-in key and on a set of large contract
  keys (over 90 MB).

### Implementation Plan

1. Publish a small library that converts between the two encodings.
2. Teach readers (proof server, `midnight-js`, wallet SDK) to accept both.
3. Teach writers (the Compact compiler) to produce the new encoding.
4. After a transition period, make the new encoding the default.

## Backwards Compatibility Assessment

No hard fork. Prover keys are never stored on the chain, so the rules the
chain runs by do not change. Old files stay valid. Readers accept both
encodings and can tell them apart by a version tag at the start of the file.
The proof server's request format carries keys too, so it needs the same
version tag. Tools that have not been updated can convert a new file back to
the old one with the library.

The new encoding depends on how the proof system lays out a prover key. If a
future version of the proof system changes that layout, the encoder must
refuse the file with a clear error rather than guess. The version tag then
changes with it.

## Security Considerations

- **Privacy does not change.** A prover key holds no user secrets. It is
  public, and anyone can rebuild it from the public contract. The new
  encoding only rewrites what is already in the file, so it cannot reveal
  anything new. The private details of a transaction never enter the key.
- **Proofs do not change.** Decoding gives the exact original key, so proofs
  made with it are exactly as valid as proofs made with the old file.
- **A damaged or tampered file** decodes to a wrong key. A wrong key makes
  proofs that the chain rejects. That is the same outcome as a tampered file
  today, so no new risk is added.
- **Malformed input.** The decoder must check every size and position it reads
  and stop with a clear error, rather than crash or allocate huge amounts of
  memory.

## Implementation

Components to change:

- A new conversion library (Rust, with a WebAssembly build for browsers).
- The ledger's prover key reader (`transient-crypto`), to accept both
  encodings.
- The proof server and `midnight-js`, which read and send prover keys.
- The Compact compiler, which writes them.

The library needs only the basic number maths of the proof system, which
these components already include.

## Testing

- Round-trip test: encode and decode every built-in key, and a set of large
  contract keys. The output must match the input byte for byte.
- Proof test: make a proof with a decoded key and check it on a local node.
- Fuzz test: feed the decoder random and truncated input. It must return an
  error, never crash.

## References (Optional)

- MPS-0039: Calling a Contract Requires Its Full Compiled Artifacts.
- MPS-0004: Trustworthy Delegated Proof Generation.
- MPS-0008, MPS-0010, MPS-0011: built-in cryptography for Compact.
- Prover key format: `midnight-proofs` 0.7, `ProvingKey::write`
  (`src/plonk/mod.rs`), and the ledger wrapper in
  `midnight-ledger/transient-crypto/src/proofs.rs`.

- Working spike, with the measurements above:
  <https://github.com/gilescope/pk-codec>

## Acknowledgements

Hector and NicolasDP (GitHub), for highlighting the need:
https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0039-lightweight-contract-interaction.md

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under
the Apache License, Version 2.0. Submission requires agreement to the Midnight
Foundation Contributor License Agreement, which includes the assignment of
copyright for your contributions to the Foundation.
