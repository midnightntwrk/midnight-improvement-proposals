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
MPS: 
- MPS-0032 (History Management for Midnight)
- MPS-xxxx (ZK-Proof Verification Throughput Bottleneck, number pending)
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
[MPS-0032: History Management for Midnight](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0032-storage-management.md) is where that concern is stated for the network: what a node has to keep on disk is what decides whether running one stays affordable as the chain ages, and MPS-0032 ties the urgency of that growth to the throughput targets of [MPS-xxxx: ZK-Proof Verification Throughput Bottleneck](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/82).
Interim states are the part of that growth nothing will ever read again, so they are the first part to go.

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

- [MPS-0032: History Management for Midnight](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0032-storage-management.md): storage growth as an operational and decentralisation concern.
- [MPS-xxxx: ZK-Proof Verification Throughput Bottleneck](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/82) (number pending): the throughput targets that set the rate at which this cost is paid.
- [midnight-node pull request #2050](https://github.com/midnightntwrk/midnight-node/pull/2050): the per-transaction form, and the source of the build that was measured.
- The measurements quoted in this proposal are summarised in Appendix A.

## Appendix A: Experimental Evidence

Every measurement quoted in this proposal comes from one experiment on the Midnight performance network (perfnet), run between 2026-08-18 and 2026-08-20.
It tested two builds that release interim states, one after the other, each against an unmodified control on the same chain.
The first, [midnight-node#1443](https://github.com/midnightntwrk/midnight-node/pull/1443), persists each new state and then releases the one it superseded.
The second, [midnight-node#2050](https://github.com/midnightntwrk/midnight-node/pull/2050), holds interim states in a keep-alive cache and persists only the block's final state.
The storage figures in the Abstract, 2.9 GB against 4.9 GB over 11,750 blocks in 20 hours, are from the first build.
The second build removed the same share of growth and was adopted in its place.
This appendix records how each run was set up, what was captured, and what was concluded.

### A.1 Common method

Both runs were an A/B of eight validators on one perfnet chain, four running the change and four running the build it was based on, plus one RPC node on the changed build.
Both arms processed the same blocks, so any difference in what they stored or how long they took is the change.

The quantity of interest is the size of the ledger's own store on disk, held separately from the Substrate chain database.
Nothing in the node reports it, so it was measured directly on the filesystem with `du`, per node, at the start and end of each load window.
The chain database was measured the same way as a control: it should grow identically on both arms, and did.
The store only grows under load; across 43 idle intervals both arms were flat to within a few megabytes, so every storage figure below is over a loaded window.

Timing came from the node's own Prometheus metrics, aggregated per arm in 30-minute buckets and expressed as the ratio of treatment to control, plotted against node uptime.
The metrics were ledger transaction validation time, ledger transaction processing time, block verification and import time, block proposal time, and host CPU.
Timing comparisons were conditioned on the number of extrinsics in a block, since an empty block takes about 5 ms to produce and a full one about 1,000 ms, and unconditioned comparisons invert.

The workload was transfer traffic, submitted at about 24 user transactions per block, so that blocks filled evenly across arms without hitting the weight limit.

### A.2 First build: release each superseded state, 2026-08-18 to 2026-08-19

**Setup.**
The treatment ran #1443.
The control was the same commit with exactly one behavioural statement removed, the call that releases the superseded state, confirmed by diffing the two builds.
Control flow, error paths, and computed state roots were therefore identical, and the arms could not fork.
The main window ran 19.6 hours, blocks 43,888 to 55,638, 11,750 blocks, about 333,000 transaction validations across the eight validators, with no node restarting.

**Captured.**
Store size and chain database size per node at the start and end of the window.
The five timing metrics in 40 consecutive 30-minute buckets, from 0.7 to 20.2 hours of node uptime.
Per-process block IO delay accounting, CPU, write bytes, and write syscalls during load rounds.
Host uptime and binary install time per node, after one window gave a contradictory result.

**Found.**

| Measure | Treatment | Control |
|---|---|---|
| Store growth over the window | 2,858 MB | 4,885 MB |
| Store growth per block | 249 KB | 426 KB |
| Chain database at end, all eight nodes | 2,113 to 2,116 MB | 2,113 to 2,116 MB |
| Spread within an arm | 5 to 6 MB | 5 to 6 MB |

The control grew 1.71 times faster; the treatment removed 41.5% of the store's growth.
The ratio was stable across every window measured, from a one-hour run to the full 19.6 hours, and the gap between arms was about 400 times the spread within an arm.

On timing, the treatment was faster on all three ledger metrics in every phase, by about 7% early in the run decaying to about 2% late, and slower in only 3 to 6 of 40 buckets on any metric.
The advantage narrowed towards parity around 15 to 17 hours and then widened again over the last five hours, significantly on all three metrics.
Both arms slowed substantially over the run, validation time roughly doubling, which is chain growth affecting everyone and not an arm effect.

One earlier window measured the treatment 3.5 to 5% slower by two independent instruments.
In that window the treatment hosts and binaries were about 26 hours older than the control's, and it was the only window in which the control's store was the smaller.
The measured sensitivity of timing to node age predicts an 8 to 10 percentage point handicap for a 26-hour gap, which covers the observed anomaly.
Arm age is therefore a first-order confound in this experiment, and the second run was designed to remove it.

**From the source.**
The release is one root-count decrement per transaction and involves no IO: block IO delay was exactly zero on both arms through every load round.
Nothing in the node calls the ledger's collection pass, so nothing is ever deleted on either arm; the saving comes entirely from what is written.
Only the final state of a block needs to stay rooted, so the intermediate releases could be deferred to block end, about 25 times fewer root-count writes at 25 transactions per block.
That observation is the origin of the block-end form in the Specification.

### A.3 Second build: persist only the block's final state, 2026-08-19 to 2026-08-20

**Setup.**
The treatment ran #2050, which reverts the storage migration and the four versioned host functions that #1443 added.
The control was the merge base on main.
A fresh chain was started, both arms deployed together, and all nine nodes rebooted immediately before load, so host age, page cache, and counters were identical at the start.
The main window ran 15.5 hours, blocks 2,065 to 11,365, 9,300 blocks.
A second load on the same node processes, without restart, ran for 813 blocks at 19 to 21 hours of uptime to test whether the trend continued.

**Captured.**
Store size and chain database size per node at the start and end of each window, cross-checked against filesystem series in Prometheus, which agreed to within 2 MB.
The five timing metrics in 30-minute buckets against uptime, 32 buckets in the main window.
The treatment build's own cache-size metric at every post-block flush, split into transient and anchored entries, as a correctness signal: a non-zero transient count means an interim state leaked past the block boundary.

**Found.**

| Measure | Treatment | Control |
|---|---|---|
| Store growth, main window | 2,607 MB | 4,545 MB |
| Store growth per block, main window | 287 KB | 500 KB |
| Store growth per block, 19 to 21 hours | 375 KB | 729 KB |
| Chain database growth, main window | 606 MB | 606 MB |
| Spread within an arm | 2 to 3 MB | 2 to 3 MB |

The treatment removed 42.7% of the store's growth in the main window and 47.7% in the later one.

| Timing metric, treatment against control | At 6 hours or more | At 12 hours or more | At 19 to 21 hours |
|---|---|---|---|
| Ledger transaction processing | 12.8% faster | 17.3% faster | 25.3% faster |
| Block verification and import | 9.5% faster | 14.6% faster | 26.6% faster |
| Block proposal | 13.3% faster | 18.3% faster | 8.8% faster |
| Host CPU | 5.7% lower | 7.9% lower | 9.7% lower |
| Ledger transaction validation | 2.7% faster | 7.7% faster | 16.4% faster |

The treatment was faster on ledger transaction processing in every one of the 32 buckets.
Two of the fitted slopes, block import and host CPU, were significantly negative, meaning the treatment pulled further ahead the longer the nodes ran.
This is the opposite of the first build, whose advantage decayed towards parity, and it is the direction the mechanism predicts: the working set is rebuilt from the store once per block instead of about three times per transaction, while the control's store grows without bound.

The one cost was ledger transaction validation, 13 to 20% slower on the treatment for the first three hours before crossing over at about 6.5 hours.
The cause was not established; the pattern is consistent with the keep-alive cache paying a population cost before its saving dominates.

The transient entry count was zero at every one of the 32 post-block flushes over 15.5 hours of sustained load, and the anchored count held at its capacity of four.
The metric exists only on the treatment build, which independently confirmed the arm assignment.

### A.4 Conclusions

- Interim states are the majority of the ledger store's growth under transfer load. Releasing them removes about 42% of that growth, and the share is the same whether each state is released by its successor or never persisted at all.
- Nodes with and without the change compute identical states and stayed in agreement on one chain for the whole of both runs.
- Releasing costs no time. The first build was a few percent faster than its control throughout. The second was faster on every metric, by 25% or more on the ledger and import paths after 19 hours, with a margin that grew with uptime.
- Nothing is reclaimed from disk on either arm, because nothing in the node runs the ledger's collection pass. The saving is what is not written, so the footprint of an existing node does not shrink on adoption.
- The second build was adopted over the first: the same storage saving, a timing margin that grows rather than fades, and no storage migration or new host function.

### A.5 Limits of the evidence

- The longest window was about 20 hours of node uptime, short against production lifetimes.
- One load shape, transfer traffic at about 24 transactions per block. How the per-transaction release cost of the first build behaves at higher throughput was not measured.
- The early validation-time penalty of the second build is unexplained.
- What the collection pass would reclaim on top of the saving, once something calls it, is unmeasured.
- The first run's timing result was exposed to an arm-age confound and is explained rather than re-run; the second run controlled for it by design.

## Acknowledgements

- Oscar Bailey (@ozgb)
- Christos Palaskas (@chrispalaskas)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
