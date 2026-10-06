---
MIP: "0019"
Title: Multipart Event
Authors:
  - Edward Alvarado <edward.alvarado@midnight.foundation>
Status: Proposed
Category: Standards
Created: 2026-09-29
Requires: MIP-0002
Replaces: none
MPS: none
License: Apache-2.0
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

This proposal defines an opt-in rule for transporting a longer public payload in several `Misc` contract events, each with a 32-byte name and a 256-byte payload. After exact contract and event-name filtering, a reader groups all matching applied events from one physical intent of one included transaction and concatenates their payloads in ledger emission order.

The physical intent supplies the package boundary. No part number, count, identifier, checksum, registry, or persistent state is added on chain. Publishers are encouraged to emit a package in the guaranteed phase, but a package emitted entirely in one fallible phase is also atomic: on success all of its events are applied, and on failure none are. All events that form one package must use the same execution phase because a fallible failure can otherwise leave only the package's guaranteed portion applied.

## Motivation

One `Misc` payload is too small for some serialized transactions, attestations, and documents. Defining a new framing format for every protocol would duplicate part numbers, counts, identifiers, and integrity checks. Contract state would make assembly persistent and contract-specific, while increasing the event size would require a Midnight capability change.

Midnight transactions already provide an intent boundary and a deterministic order for applied contract events. This proposal standardizes how an adopting protocol uses that boundary without changing the ledger.

## Specification

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they appear in all capitals.

### 1. Scope and terminology

An **adopting protocol** opts an event name into this rule. A **publisher** emits the events. A **reader** reconstructs their **transport package**. A **part** is the 256-byte payload of one matching applied `Misc` event. A transport package is the ordered concatenation of one or more parts.

### 2. Opting in

The chain and contract address come from the deployed contract instance. The adopting protocol MUST declare:

1. **Event name:** the exact value of the existing `Misc` `name` field, shared by every part.
2. **Multipart rule:** the protocol's specification MUST state that it follows `mip-xxxx:multipart[v1]`.

This rule begins only after exact contract and event-name filtering of valid, decoded, applied events. If a protocol has not made that opt-in, this proposal has no effect on its events. Invalid envelopes, unsupported decoders, incomplete API responses, conflicting upstream deliveries, and unavailable history are outside this rule's input boundary.

For every nonempty group in scope, the only result defined here is an accepted transport package. This proposal does not define an application schema, application validity, authorization, signatures, truth, or semantic replay policy. An adopting protocol may consume arbitrary bytes and need not define any of those concepts.

### 3. Reconstructing packages

For a deployed contract instance and an opted-in event name, matching events from the same included transaction and the same physical intent form one group. Each nonempty group produces one package. A reader MUST follow ledger emission order, take each event's full 256-byte payload exactly once, and concatenate the payloads in that order. It MUST preserve every byte, including trailing zeros.

Events from different chains, contract addresses, event names, physical intents, or included transactions MUST NOT be joined, even when their caller or payload bytes are equal. Zero matching events produce no package. One matching event produces a one-part package. Several events that a publisher intended as separate messages still produce one package when they are in the same group; this rule carries no hidden sub-boundary. The package length is 256 times its number of events, and the original unpadded application length cannot be recovered from this transport alone.

### 4. Publisher requirements and atomicity

All matching events that a publisher places in the same physical intent share the same physical segment number within the transaction and would form one package when applied, even if the publisher regards them as separate logical messages. The publisher MUST place all of those events in one execution phase. It MUST preserve the intended byte order when it emits them. The parts MAY be produced by one call or several calls, and a call MAY emit more than one matching event.

A publisher SHOULD use the guaranteed phase. Guaranteed placement avoids a fallible execution failure that produces no package. Guaranteed placement is a recommendation, not a condition for this transport rule.

A publisher MAY instead place all of those events in the fallible phase of the physical intent. If that phase succeeds, all of its events are applied. If it fails, its state changes and locally accumulated events are discarded, so the reader receives no matching applied event and produces no package. The caller MUST verify that the transaction was included on chain and that this fallible phase succeeded before treating the package as published.

A publisher MUST NOT split those events between the guaranteed and fallible phases. The ledger applies guaranteed events before fallible segments. If the fallible phase then fails, its events are discarded while the guaranteed events may remain applied in a partially successful transaction. A reader still groups the matching events that were actually applied; it does not infer an unemitted part or introduce another result for the publisher's mistake.

Canonical-chain and reorganization handling remain part of the underlying event processing. This proposal assumes its input contains valid, decoded, applied events with known ledger emission order; it does not claim that an arbitrary endpoint is honest, available, complete, or permanently retaining history.

## Rationale

### Requirement

Applications need messages larger than the fixed 256-byte payload of one `Misc` event. Splitting a message across events supports application-defined message lengths within transaction limits.

### Transport

The physical intent is the smallest existing boundary that can group several circuit calls without allowing a third party to add calls during transaction composition. Using it as the package boundary avoids new on-chain part numbers, counts, or package identifiers, persistent contract state, and changes to the event size.

### Security

When all parts use one execution phase of the same physical intent, their events are applied together or not at all, including when several circuit calls emit them. A third party cannot inject additional parts into the sealed intent; another intent forms a separate package. This MIP does not define who may emit data from a contract or the mechanism that enforces that permission.

### Rebuilding

Calls within that phase execute in the intent's defined order, and matching events retain their emission order. Concatenating their full payloads in that order reconstructs the published message bytes, including any padding. Recovering the original unpadded length remains the adopting protocol's responsibility.

## Path to Active

This document is a Draft and makes no claim of Acceptance, Implementation, or network activation.

### Acceptance Criteria

Before this proposal can be considered Active:

1. MIP editors assign its number and accept the normative rule through the MIP process;
2. at least two independently implemented readers reproduce every normative vector below;
3. guaranteed and fallible-only publications are exercised against an implementation of the prerequisite, including a failed fallible phase with no applied parts;
4. a public activation record identifies the exact chain, implementation, and observation artifacts; and
5. no unresolved security or interoperability issue changes the specified behavior.

### Implementation Plan

Publish the vectors with a small reference reader, update the existing reference publisher and reader to the accepted text, test another independent reader, and record activation only after the specified phase behavior and public-network examples are reproduced. Failures keep the proposal at its current stage until the specification or implementation is corrected.

## Backwards Compatibility Assessment

This proposal changes no ledger, VM, node, compiler, contract event, or indexer API. Protocols that do not opt in are unaffected. A one-part package has the same 256 payload bytes as the original event. Older readers continue to expose separate events; readers for an adopting protocol combine them first.

An opt-in has no inherent start height. Applying it to an event name already in use reinterprets all matching history. If one old intent contains several independent events with that name, the rule combines them. A new event name avoids this ambiguity. This proposal warns about the risk but does not forbid retroactive opt-in.

## Security Considerations

Transaction composition cannot add calls to an already sealed physical intent; a colliding physical segment is refused. A composer may add another intent, which forms a separate package.

A publisher can still publish false or misleading bytes, place two intended messages in one package, or split a message across different intents or transactions. It can also place matching events for one package across execution phases, contrary to the publisher rule. This transport convention does not establish truth, authorship, authorization, or an application-level boundary. The package-level phase rule prevents a fallible failure from leaving only the guaranteed prefix of one transport package.

Every part and package is public. The contract, name, transaction, intent, payload length, bytes, timing, and frequency may be visible to ledger readers, nodes, indexers, wallets, frontends, and proof services. Application encryption may hide payload plaintext but does not hide that metadata or retract disclosed data.

Repeated equal bytes in a new intent or transaction form a new package. The transport defines no semantic replay handling. Consumers must also respect the chain and reorganization policy of their event processing. Passing malformed or incomplete upstream data into this algorithm violates its input premise; this proposal does not add a second acquisition or endpoint-security protocol.

## Implementation

No Midnight component change is required. Implementation belongs in the adopting protocol and its contract, publisher, and reader tooling:

1. **Protocol declaration:** identify the existing event name and declare adoption as specified under Opting in. Any application encoding or interpretation of padding remains the protocol's responsibility.
2. **Publisher and contract:** prepare the message as 256-byte parts and emit them under the opted-in name, in the intended order, within one execution phase of one physical intent. One or several circuit calls can emit the parts. For fallible publication, the caller confirms transaction inclusion and success of that phase before treating the package as published.
3. **Reader:** obtain valid, decoded, applied events for the deployed contract and opted-in name. Group them by included transaction and physical intent, concatenate each full payload exactly once in ledger emission order, and deliver the resulting bytes to the application without trimming zeros.

The [reference implementation](https://github.com/acedward/mip-multipart-event) provides example publisher and reader tooling. Its publisher supports guaranteed-only publication.

## Testing

The following vectors are normative. They begin after opt-in and exact contract and event-name filtering. Inputs are valid, decoded, applied events with a known ledger emission order. Unless stated otherwise, all events use one chain, contract, name, transaction `T1`, physical intent 7, and the guaranteed phase.

Each test payload is exactly 256 bytes:

- `A`: 256 bytes of `0xaa`.
- `B`: 255 bytes of `0xbb`, followed by one `0x00` byte.
- `C`: 256 bytes of `0xcc`.
- `Z`: 256 bytes of `0x00`.

A conforming reader MUST reproduce the package groups, part counts, part orders, lengths, and bytes specified below. These vectors test reconstruction from applied events; they do not execute ledger state transitions or prove phase atomicity.

### Reconstruction and ordering

1. **Empty filtered input:** no matching events. Expected: no package.
2. **All-zero part:** one event with payload `Z`. Expected: one single-part, 256-byte package containing `Z`.
3. **Trailing zero:** one event with payload `B`. Expected: one single-part, 256-byte package containing `B`, including its final zero.
4. **Upstream order:** events arrive as `B`, then `A`, but their ledger emission positions are 1 and 0 respectively. Expected: one two-part, 512-byte package containing `A` followed by `B`.
5. **Equal distinct events:** two events at distinct ledger positions each contain `A`. Expected: one two-part, 512-byte package containing `A` followed by `A`; equal bytes do not remove an event.
6. **Multiple logs per call:** one call emits `A`, then `C`. Expected: one two-part, 512-byte package containing `A` followed by `C`.

### Package boundaries

1. **Two intents:** intent 7 emits `A`; intent 8 emits `B` in the same transaction. Expected: two packages of one part and 256 bytes each, containing `A` and `B` respectively; never join them.
2. **Repeated publication:** transaction `T1`, intent 7 and transaction `T2`, intent 7 each emit `A`. Expected: two packages of one part and 256 bytes each, despite equal payloads and intent numbers.
3. **No hidden framing:** `A` and `C` were intended as separate messages but were emitted in that order in one group. Expected: one two-part, 512-byte package containing `A` followed by `C`.

### Publication phases

1. **Guaranteed multipart:** guaranteed events contain `A`, then `B`. Expected: one two-part, 512-byte package containing `A` followed by `B`.
2. **Fallible success:** fallible events contain `C`, then `B`, and both are applied. Expected: one two-part, 512-byte package containing `C` followed by `B`.
3. **Fallible failure:** the fallible phase failed, leaving no applied matching events. Expected: no package.
4. **Same-group separate intentions:** guaranteed `A` was intended as message 1; applied fallible `B` was intended as message 2 in the same group. Expected: one two-part, 512-byte package containing `A` followed by `B`. The publisher violated the package-level single-phase requirement despite intending separate messages.
5. **Mixed-phase failure:** guaranteed `A` was applied; fallible `B` was discarded and is absent. Expected: one single-part, 256-byte package containing `A`. The publisher violated the package-level single-phase requirement; the reader does not infer `B`.

### Publisher execution checks

A publisher implementation MUST test the phase it supports against its supported event implementation. An implementation that supports fallible publication MUST also test fallible success, fallible failure, and the mixed-phase counterexample. These execution checks establish that failed phases expose no applied parts; the reader vectors alone cannot establish that behavior.

## Acknowledgements

The multipart use case originated in work on SIG Network. Dominik Zajkowski authored the public-event proposal on which this proposal depends.

## Copyright Waiver

All code and text contributed through this proposal are to be licensed under the Apache License, Version 2.0. The operative Contributor License Agreement text and link are editor-owned publication metadata and remain to be supplied by the MIP repository before submission.
