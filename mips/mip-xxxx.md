---
MIP: ?
Title: Node-Side Surfacing of Ledger Events via frame_system::Events
Authors:
  - Mike Clay (m2ux)
Status: Draft
Category: Core
Created: 2026-05-18
Requires:
Replaces:
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

_Draft scaffold — full content authored during the `implement` activity._

This MIP proposes a candidate solution to [MPS-0007: Node Event Visibility](../mps/mps-0007-node-event-visibility.md). The Midnight node currently receives a stream of per-transaction events from the ledger crate but discards them before they reach any external consumer. This proposal specifies the publication of those events on Substrate's existing `frame_system::Events` channel, wrapped in a stable, version-uniform pallet event so consumers can decode them with standard Substrate tooling regardless of which ledger crate version produced them.

## Motivation

_Section drafted from `.engineering/artifacts/planning/2026-05-11-node-expose-ledger-events/01-design-philosophy.md` and `02-kb-research.md` during the `implement` activity._

## Specification

_Section drafted from the work-package plan and assumptions log during the `implement` activity._

## Rationale

_Section drafted from the design philosophy and research artifacts during the `implement` activity._

## Path to Active

### Acceptance Criteria

_Drafted during `implement`._

### Implementation Plan

_Drafted during `implement` — references the upstream PR in `midnightntwrk/midnight-node` once opened._

## Backwards Compatibility Assessment

_Section drafted during `implement` — sources the backwards-compatibility analysis already completed in the implementation work package._

## Security Considerations

_Section drafted during `implement`._

## Implementation

_Drafted during `implement` — pointers to the upstream node PR and runtime changes._

## Testing

_Section drafted from `.engineering/artifacts/planning/2026-05-11-node-expose-ledger-events/05-test-plan.md` during `implement`._

## References

- [MPS-0007: Node Event Visibility](../mps/mps-0007-node-event-visibility.md)
- [Issue #1474 — Node should expose events emitted by the Ledger](https://github.com/midnightntwrk/midnight-node/issues/1474)

## Acknowledgements

_To be populated._

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
