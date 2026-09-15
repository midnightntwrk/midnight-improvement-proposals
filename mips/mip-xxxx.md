---
MIP: "xxxx"
Title: Interim Ledger State Management
Authors:
- Dominik Zajkowski (@dzajkowski)
Status: Draft
Category: Core
Created: 2026-09-14
Requires: none
Replaces: none
MPS: ZK-Proof Verification Throughput Bottleneck (https://github.com/midnightntwrk/midnight-improvement-proposals/pull/82, number pending)
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

## Abstract

A node applies the transactions in a block one at a time, and each application produces a new ledger state.
Only the last of those states matters once the block is complete; the ones before it are interim, produced by one transaction and consumed by the next.
Yet the node keeps every one of them, in a store the ledger holds separately from the chain database, and nothing ever removes them.
That store grows with every transaction, forever.
This proposal changes the node to release interim states, so the store no longer has to keep what nothing will ever read again.

The proposed change to the node was measured in a controlled comparison: eight validators on one chain, four running the change and four running the same build without it, under load for 20 hours.
Over the same 11,750 blocks, the ledger's own on-disk storage grew by 2.9 GB on the nodes running the change and by 4.9 GB on the nodes without it, so the change removes about 42% of that growth.
The nodes running the change were also slightly faster, by a few percent on every timing measure, which is what doing less work per transaction buys.
Existing storage does not shrink on adoption; the gain is a halved growth rate.
To benefit from the change, a node will have to start from genesis or use an relevant snapshot.

## Motivation

The cost of keeping every interim state is paid per transaction, so it scales with throughput: any change that raises transactions per second also makes the store grow faster.
The growth never reverses, so the longer a node runs and the more traffic it carries, the more it holds, and the experiment's data is consistent with a larger store being slower to operate.
A network that intends to raise its throughput cannot keep a per-transaction storage cost that never falls.

The ledger already knows how to tell needed state from unneeded state.
The node just never tells the ledger that a superseded interim state is no longer needed.
The growth is not a missing capability in the ledger; it is the node not exercising the one it has.

## Specification

"Release", below, means the node telling the ledger's store that it no longer needs a state.
The store only has to keep states that something still needs, so releasing trims the growth down to the states the node actually has to keep.
Garbage collecting released states from disk is a separate mechanism, unused today, and this proposal leaves it that way.
This decision is motivated by the current behavior of the flushing mechanism.
An in-memory ledger-state representation will be persisted when a `flush` is called.
If a node is creating one block at a time, this happens at the end of the blocks creation flow, so there is no need to cleanup, as only referred states will be persisted.

Applying a transaction stores the new ledger state it produces.
That does not change.
What changes is that interim states get released, in one of two forms:

1. **Per transaction, as measured.** Each transaction's application releases the state it just superseded. This is the form the numbers in this document come from.
2. **At block end.** When a block completes, the node releases all of its interim states in one step, keeping only the final state.
   One release per block instead of one per transaction, but this form has not been implemented, measured, or shown correct.

This proposal recommends the block-end form, subject to the verification called for in the Implementation Plan.
The per-transaction form is the measured fallback if that verification fails.
The release work happens once per block instead of once per transaction, so committing a single state at block end may be faster outright, which the comparison in the Implementation Plan will measure.
At the same time it is the larger change to how the node processes a block, so its correctness has to be shown rather than assumed.

## Rationale

Today's behaviour is not maintainable over a long period: the store grows with every transaction and never shrinks, so the question is not whether to change something but which change carries the least risk.
The changes considered here are the lowest-risk kind: they use the ledger's own mechanisms better and leave how transactions are processed untouched.
Of the two forms, the per-transaction one was implemented and measured first because it is the simplest; the block-end form promises more, one release per block and possibly a faster commit, but is the more complex change.
Anything more fundamental, changing how transactions are processed at all, has not yet been researched and is beyond this proposal.

An interim state exists to be the input of the next transaction in the block. 
Once that transaction has run, nothing in normal block processing reads it again, and the ledger itself already classifies such states as transient.
Releasing changes only what the store keeps, not what any state contains, so nodes with and without the change compute identical states.
The measured comparison bears both points out: the two kinds of node ran the same chain, side by side, in agreement, for the whole 20-hour run.

The two forms release the same states, all but the final one of each block. The difference is timing only.
The storage saving is therefore expected to carry over unchanged: by the time a block is done, the same states are released either way.
The one way this breaks is if the store does work on a state in the gap between the two moments, so the expectation has been checked against the code but has to be confirmed by running it, which is the verification the Implementation Plan calls for.

Running the ledger's garbage collection pass on top of releasing was considered and left out.
Nothing in the node runs it today, what it would reclaim beyond releasing is unmeasured, and when to run it (block finalisation is the obvious candidate) is an open design question of its own.

## Path to Active

### Acceptance Criteria

- The written properties and their test suite exist, and the implementation in the released build passes it.
- The chosen form is implemented in midnight-node and part of a released build.
- Nodes running the change and nodes without it stay in agreement under sustained load.
- A controlled comparison of the final implementation shows ledger storage growing materially slower, on the order this document measured, with no timing cost.
- Disk usage for a from-genesis-synced node is reduced compared to a node synced without this change.

### Implementation Plan

1. Write down which states the node needs at each point of processing a block, and when a state may be released, covering every mode of block processing: producing blocks, importing them, catching up from behind, and switching between forks.
These properties are what the rest of the plan tests against.
2. Build a test suite that checks an implementation against those properties.
Run it against the unmodified node first: it must pass every property about what is kept, which validates the properties themselves before they judge anything new.
3. Verify the per-transaction form against the suite, then implement the block-end form and verify it the same way.
This is the verification the recommendation in this document is subject to.
If the block-end form fails, fall back to the per-transaction form, which has already passed.
4. Show that nodes running the chosen form stay in agreement with unmodified nodes.
5. Run a controlled comparison of the chosen form under sustained load, with the compared nodes matched on host and build age, measuring storage growth and timing.
6. Roll out to a testnet and observe under load.

## Backwards Compatibility Assessment

No hard fork is needed.
The change does not alter what a block contains or which blocks are valid, and the measured comparison is itself the compatibility evidence: nodes with and without the change ran the same chain, at the same time, for the whole run.

For a node operator, adoption changes the growth rate, not the footprint.
The store reuses space it has freed internally rather than returning it, so storage already on disk stays.
A node that needs a smaller footprint gets one only by syncing fresh or restoring from a snapshot.

## Security Considerations

The risk this change introduces is releasing a state the node still needs.
The saving comes precisely from the store no longer keeping what is released, so a wrong release means missing state, and the code being right is currently the only guard.
The written properties and the test suite in the Implementation Plan are the mitigation, and they must cover every mode in which a node processes blocks: producing them, importing them, catching up from behind, and switching between forks.

The change also narrows an exposure that exists today: every transaction permanently grows every node's store, so sustained traffic is also a sustained storage cost imposed on the whole network.
Releasing the needless states shrinks what a transaction's fee buys in permanent storage, though growth itself remains.

## Implementation

The change lives where the node applies a block's transactions to the ledger.
The per-transaction form exists as [midnight-node #2050](https://github.com/midnightntwrk/midnight-node/pull/2050): each transaction's application releases the state it supersedes, and that build is where this document's measurements come from.
The block-end form is not yet built; it restructures the same code path to release all of a block's interim states once, when the block completes.
Neither form adds a runtime component or a dependency: the release mechanism already exists in the ledger, and the change is the node calling it.
It is not entirely certain that the block-end form will not require a runtime upgrade alongside node changes.

## Testing

- Unit tests verify an implementation against the written properties: apply a block's worth of transactions in isolation, check that exactly the permitted states were released.
- End-to-end tests on a synthetic network force the modes unit tests cannot reach, catching up and fork switches, and check the store's contents against the same properties afterwards.
- A mixed network of changed and unchanged nodes, including nodes joining late and catching up, stays in agreement under sustained load.
- A controlled comparison under sustained load measures storage growth and timing, with the compared nodes matched on host and build age.
- Storage is measured under load. An idle chain's store barely grows, so an idle measurement shows nothing either way.

## References

- [midnight-node pull request #2050](https://github.com/midnightntwrk/midnight-node/pull/2050): the per-transaction form, and the source of the build that was measured.
- Throughput exploration report, entry 0023, Execution Interim Storage Management: the experiment and the measurements this document quotes. Private at the time of writing, pending publication. //TODO add a link to the report when it is publicly available

## Acknowledgements

- Oscar Bailey (@ozgb)
- Christos Palaskas (@chrispalaskas)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
