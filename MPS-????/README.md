---
MPS: ????
Title: Governance Observability for Builders
Status: Open
Category: Tools
Authors:
  - Noel Rimbert (noelrim)
Proposed Solutions: []
Discussions:
  - TBD
Created: 2026-03-27
License: Apache-2.0
---

## Abstract

Midnight governance operates across two execution surfaces. On Midnight, governance decisions are proposed, approved, and executed through a federated authority mechanism. On Cardano, authority membership and governed smart contracts hold authoritative state that the Midnight node observes via a mainchain follower. The node processes governance activity on both surfaces, but the midnight-indexer — the GraphQL API that wallet developers, block explorers, analytics platforms, and integrators rely on — does not capture it. Beyond D-parameter and Terms & Conditions history, governance data is inaccessible to builders.

This means builders cannot query the lifecycle of governance decisions, cannot determine who holds governance power at a given block height, and cannot track activity on governed Cardano smart contracts. There is no audit trail for governance actions that produce no state change, no categorization of governance event types, and no cross-chain governance view. As governance evolves toward participative voting and new action types, this observability gap will only widen.

This problem statement describes the gap, the builder use cases it blocks, and the goals any solution must meet to make governance queryable, subscribable, and auditable across both surfaces.

## Problem

### Governance spans two execution surfaces

Midnight governance is federated — Council, Technical Authority, and Federated Node Operators manage the protocol primarily through upgradability actions. All three bodies' membership is anchored on Cardano governance smart contracts.

On the **Midnight surface**, governance decisions flow through a federated authority mechanism: both Council and Technical Authority must independently reach a 2/3 approval threshold before a governance action can be dispatched. The lifecycle includes proposal, per-body approval, dispatch (execution), expiry, and revocation. These governance events are emitted by the node's `pallet-federated-authority` runtime but are not indexed.

On the **Cardano surface**, governed smart contracts (ICS, Reserve, Council Authority, TA Authority, FNO membership) follow an upgradable contract pattern with MAIN and STAGING governance UTxOs, Round-based versioning, and multi-signature authorization. Governance actions include logic/auth/mitigation script promotion, datum changes, and authorized contract invocations (e.g., Reserve releases to ICS). Authority membership changes are observed by Midnight via a mainchain follower and emitted as `CouncilMembersReset` and `TechnicalCommitteeMembersReset` events, but these too are not indexed. Broader governed SC events (promotions, invocations) are not currently surfaced to builders at all.

### What builders cannot do today

The midnight-indexer currently indexes only two governance-related data points: D-parameter history and Terms & Conditions history. Both follow a "store only on change" pattern via runtime storage queries. The remaining governance surface is unobservable:

- **Governance decision lifecycle is invisible.** Builders cannot determine which governance actions were approved, by whom, when they were dispatched, or whether they succeeded or expired. The five motion lifecycle events (`MotionApproved`, `MotionDispatched`, `MotionExpired`, `MotionRevoked`, `MotionRemoved`) are emitted by the node but never reach the indexer.

- **Authority membership state is opaque.** Builders cannot query who holds governance power (Council, TA, FNOs) at any historical block height, nor track when membership changed. The two membership observation events from Cardano governance smart contracts flow through the mainchain follower into the node but are not surfaced.

- **Governance action types are indistinguishable.** Runtime upgrades, parameter updates, Terms & Conditions updates, and governed SC upgrades are not categorized. Builders cannot filter or display governance events by what was actually governed.

- **The audit trail has gaps.** Existing D-parameter and T&C historization silently drops governance decisions that produce no state change ("no-ops"). A governance decision was made, but if the resulting values are unchanged, no record exists.

- **Cardano governed SC activity is invisible.** Promotions, datum changes, and authorized operations on governed Cardano smart contracts (ICS, Reserve) are not observable by builders. Non-governed contracts (e.g., cNIGHT-to-Dust generation SC) are out of scope.

- **No cross-chain governance view exists.** No tool — whether Polkadot-native (Polkassembly, SubSquare), Cardano-native (GovTool, Governance Health KPI Dashboard), or cross-chain (Tally MultiGov, Wormhole MultiGov) — observes governance across heterogeneous Substrate and Cardano architectures. Midnight's bidirectional observation pattern is architecturally novel with no competitor coverage.

### Why this matters now

As governance evolves toward participative voting and additional governance action types, observability becomes foundational. Building the governance data infrastructure now — while governance is federated and focused on upgradability — ensures that when governance expands, builders already have the tools to observe, query, and audit governance activity at scale.

## Use Cases

### 1. Wallet builder displaying governance status

A wallet developer building on Midnight wants to show users the current governance state — active governance decisions, pending upgrades, and authority membership. Today, they can display D-parameter and T&C values from the indexer, but cannot show governance decision lifecycle, who approved what, or whether a runtime upgrade is pending. Without governance observability, wallets present an incomplete picture of protocol health.

### 2. Block explorer showing governance history

An explorer developer building a Midnight block explorer needs to display the full governance event timeline — every approval, dispatch, expiry, and membership change — with the ability to filter by governance action type and governance body. Today, they would need to scan the entire chain block-by-block to reconstruct this history. The indexer provides no governance event queries.

### 3. Analytics platform tracking governance health

An analytics operator wants to build a governance health dashboard showing participation rates, motion outcomes, membership change frequency, and governance action categorization trends. Today, the data to compute these metrics does not exist in any queryable form. The operator would need to run a custom chain scraper and decode Substrate pallet events manually.

### 4. Integrator monitoring cross-chain governance

A Cardano ecosystem integrator needs to monitor governance activity affecting the Midnight-Cardano bridge — governed SC promotions, membership changes, and contract invocations on ICS and Reserve. Today, this requires monitoring both chains independently with separate tooling and no unified view. The indexer provides no Cardano-side governance data.

### 5. Auditor reconstructing governance decisions

A compliance auditor needs to verify the complete chain of governance decisions from genesis — including who approved each action, what it enacted, and whether no-op decisions were made. Today, D-parameter and T&C history exist but silently drop no-op governance actions. Motion lifecycle events are not preserved in any queryable form. A complete audit trail is impossible without scanning raw block data.

### 6. Governance participant tracking real-time changes

A Council or TA member wants real-time notifications when governance events occur — a new motion approved by the other body, an expiring motion, a membership change observed from Cardano. Today, no subscription mechanism exists for governance events. The member must poll block data manually or rely on off-chain communication channels.

## Goals

Goals are listed in order of priority.

1. **Governance decision lifecycle must be queryable.** Builders must be able to query the full lifecycle of governance decisions on Midnight — approvals, execution outcomes, expiry, revocations — by motion hash, governance body, action category, and block height range.

2. **Authority membership must be queryable at any block height.** Builders must be able to determine who holds governance power (Council, TA, FNOs) at any historical block height, with both Midnight account IDs and Cardano mainchain member identifiers.

3. **Governance events must be categorized by action type.** An extensible taxonomy must distinguish runtime upgrades, parameter updates, T&C updates, governed SC upgrades, governed SC invocations, and governance membership updates — unified across surfaces, not surface-based.

4. **Cardano governed SC events must be observable.** Promotions, datum changes, and authorized operations on governed Cardano smart contracts (ICS, Reserve, membership SCs) must be accessible through the same API. Non-governed contracts are out of scope.

5. **Governance history must be complete and reconstructable from genesis.** Every governance event — including decisions that produce no state change — must be recorded. Builders must be able to replay governance history and reconstruct governance state at any point in time.

6. **Real-time governance subscriptions must be available.** Builders must be able to subscribe to governance events filtered by category, receiving push notifications when governance changes occur.

7. **Governance data consumption must be unified across surfaces.** Builders must be able to query a single cross-chain governance timeline as well as surface-specific views, with consistent schema patterns regardless of whether events originated on Midnight or Cardano.

8. **Governance indexing must be configurable and opt-in.** Indexer operators must be able to enable or disable governance indexing without impacting base indexer performance, with zero overhead when disabled.

9. **The governance data model must accommodate governance evolution.** Governance bodies must be modeled as generic collectives (not hardcoded as Council/TA), and the schema must support future governance structure changes (new bodies, participative voting, additional action types) without breaking historical data.

## Open Questions

1. **What is the minimum metadata the indexer must store per governance event?** Full event data maximizes query convenience but increases storage. Minimal metadata (event type, timestamp, block reference) with on-demand detail resolution from the chain reduces storage but requires node access for deep queries. What is the right trade-off for builder workflows?

2. **How should the indexer source Cardano governed SC events beyond membership?** Only membership observation events currently flow through the mainchain follower into Midnight blocks. Broader governed SC events (promotions, datum changes, invocations on ICS/Reserve) do not. Should the indexer query Cardano directly, should the mainchain follower be extended, or is there another approach?

3. **How should governance events be correlated across surfaces?** A hard fork may require both a Midnight runtime upgrade and a Cardano SC promotion. Today, there is no on-chain standard for linking these related actions. What governance metadata standards (inspired by Cardano's CIP-100/108 anchor pattern) could enable cross-event linkage in the future?

4. **Should governance indexing support configurable granularity modes?** Operators may want different levels of governance data retention — minimal (terminal events only), normal (full lifecycle with minimal metadata), or archive (full lifecycle with rich metadata). Should this be a product requirement or an implementation detail?

5. **What per-event-type context fields are sufficient for builders?** Runtime upgrades might store spec_version; parameter updates store actual values; SC promotions store Round number. What is the minimum context per event type that allows builders to understand what happened without requiring a block lookup?

6. **How should the GraphQL schema balance unified and surface-specific queries?** A three-layer approach has been proposed (value-level queries following existing patterns, surface-bound event queries, unified cross-chain timeline). Is this the right consumption model, or do builders prefer a different organization?

## Copyright

This MPS is licensed under the Apache License, Version 2.0.
