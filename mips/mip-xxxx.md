---
MIP: "xxxx"
Title: Transaction Revalidation Cache
Authors:
- Dominik Zajkowski (@dzajkowski)
Status: Draft
Category: Core
Created: 2026-08-24
Requires: none
Replaces: none
MPS: ZK-Proof Verification Throughput Bottleneck, number pending
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

A node checks the zero-knowledge proofs in a transaction when the transaction enters the pool, and checks them again whenever ledger state changes, which happens when a block is produced and when a block is imported.
The only piece of ledger state a proof check reads is the verifying key, so the answer to that check stays the same until the key changes: at a protocol upgrade for the protocol's own proofs, and at a contract update for that contract's proofs.
On the performance network each transaction's proofs were checked about seven times across a three-node network, and proof checking was the largest single share of block import time.

This proposal keeps the result of a successful check and reuses it when state changes, re-running only the part of validation that does depend on state.
In a controlled comparison across eight validators it made block production 42.4% faster and block import 41.1% faster on blocks carrying transactions, a saving of about 22 ms per transaction on each path.

The change is local to a node: it changes how often validation work is repeated, not what a block contains or which blocks are valid.

## Motivation

[ZK-Proof Verification Throughput Bottleneck](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/82) sets the target this proposal serves: 1,000 transactions per second, against a block author with roughly 1,500 ms per 6 second slot.
That budget is what throughput is made of, and every millisecond of it spent re-checking a proof is a millisecond unavailable to another transaction.

Proof checking is a significant part of the block path.
In an instrumented run, ledger processing accounted for 91.5% of block import time, proof checking cost about 22 ms per transaction, and a full block spent 714 ms of 1183 ms there.

A key observation is that a lot of the time is spent in repeated work.
Each transaction's proofs were checked about seven times across a three-node network.

Every number above comes from one traffic shape: transfer transactions only (night and shielded), sent at a steady rate, on a small test network.
Per-transaction cost follows the number of proofs a transaction carries, so a different mix, contract calls included, would move all of these figures.
The repetition itself does not depend on the shape so it stands to reason that addressing it will result in improvements.
Another key observation is that the work is deterministic, so repeating it buys nothing. 
The cost of keeping results is memory, addressed under Security.

Reducing the cost of one check is a separate line of work, covered by batch verification and by proof aggregation.
This proposal removes checks that do not need to happen at all, and composes with both.

## Specification

"State-dependent" below means the part of validation that reads state transactions change.

1. When a transaction is admitted to the pool, validate it fully and keep the validated result.
2. When the transaction is considered (block creation or on block import) and the ledger state changed, and verifying keys did not change: keep the stored result, re-run only the state-dependent part of validation against the new state. Otherwise validate fully.
3. Discard the stored result when the transaction is no longer relevant (a cheap mechanism like TTL or similar should be sufficient).

The following are not settled and have to be specified before implementation is complete:

- **Lookup key.** What identifies a validated transaction so that a lookup succeeds across a state change. The measured implementation keys on a value tied to chain state; making the key tolerant of that value is the specific change the profiling pointed to. The exact form is open.
- **Bound and eviction.** The measured build retains 2000 entries. Whether that is the right bound, and what is evicted first, is open.
- **Lifetime.** When an entry must be discarded rather than revalidated. Two triggers are already known: a protocol upgrade invalidates every stored result at once, and a contract update invalidates stored results whose proofs were checked against that contract's keys. How the node notices the second is open.
- **Notifications.** Whether reuse changes what the node reports about a transaction's progress. This is an open correctness problem, not a design choice.

## Rationale

Validation asks two questions.
First: is the proof mathematically valid for what this transaction claims?
That check reads the transaction and one piece of ledger state, the verifying key; everything else it needs, including the state snapshot the proof was built against, travels inside the transaction.
Second: can the transaction be applied to the state we have now? Are its coins still unspent, is the snapshot it references one the ledger still accepts?
Only the first answer is stored and reused, and it stays correct until the key changes.
A double spend is not let through by reuse: it fails the second question, which re-runs if the state changed.
The protocol's own keys change only at a protocol upgrade.
A contract's keys can change when the contract is updated, and an update is an ordinary transaction, so reuse has to notice key changes at transaction-execution time rather than rely on upgrade events.
On the measured transfer traffic the second question was the cheaper one: roughly 13 ms per transaction against 22 ms for the proof check.
That ratio is a property of the traffic, not of the design; a transaction carrying more on-chain work makes the second question more expensive and shrinks the share reuse saves.
Keeping the first answer and re-running the second follows from the split either way, the traffic shape affects how much that buys.

The proposal builds on a cache keyed on chain state which is already in use in the node.
The improvement is easy to trace: the current cache has low hit rate due to the state dependence.

Deferring proof checking off the block path entirely is a different proposal, not an alternative to this one.
It would move the work rather than remove the repeats, and it interacts with pool admission and spam resistance.
Since pool admission already verifies proofs, this caching approach would benefit from it during block creation and somewhat during block import (depending on the effectiveness of the mempool gossip protocol).

## Path to Active

### Acceptance Criteria

- Throughput improved against current benchmarks.
- A controlled A/B comparison run, with results documented.
- Transaction progress notifications shown to behave the same as an unmodified build.
- Memory held by the cache measured, and bounded.

### Implementation Plan

1. Prepare a production ready implementation of proof cache. Measure and report hit rate.
2. Define the properties of transaction validity on block inclusion and block import for live processing and catch-up.
3. Implement a test suite which verifies any implementation against the defined properties. Cover different shapes of traffic: load and tx diversity.
4. Show parity between the cached and raw implementation.
5. Run a comparison in a mixed testnet where cached and non-cached nodes behave the same way. Include catchup nodes, passive nodes.
6. Measure the memory the cache holds under sustained load, show bounded operation.
7. Roll out to a testnet, stress test.

## Backwards Compatibility Assessment

No hard fork is expected.
The change does not alter block contents, validation rules, or which blocks are valid, and nodes with and without it ran on the same chain at the same time throughout the comparison.

## Security Considerations

**Memory.** The cache holds validated transactions together with their proof data.
On the measured configuration that was estimated at 100 to 400 MB against 8 GiB of node memory, and it was not measured, because the network runs no host metrics collector.
This consideration needs to be included in the system requirements of the node.

**Soundness of reuse.** Reuse is only safe because the state-dependent part of validation re-runs on every state change.
A written argument for that, covering what the state-dependent part must cover, is open.
The argument must also cover verifying-key changes: reuse across a key change is unsound, and contract keys can change through an ordinary transaction, not only through a protocol upgrade.
Whether the measured implementation handles a contract update landing while a transaction is pooled is untested.

**Spam.** Admission to the cache is work an attacker can cause.
Whether it gives a cheap way to occupy node memory is open, and it overlaps with existing work on the cost of rejecting invalid transactions.

**Network fragmentation.** Mempool transactions reach only partitions of the network, causing some of the participants to see a tx for the first time during bock validation.

## Implementation

The change sits in the node's transaction pool and validation path, with the revalidation reference on the ledger side.
An implementation exists as [midnight-node#744](https://github.com/midnightntwrk/midnight-node/pull/744).

**Open:** The expected behavior of this code path is not documented beyond code.
A recommended approach is to capture it as the testable properties of transaction admission.

## Testing

The properties defined in step 2 of the implementation plan are the reference.
Each mode in which a node validates transactions is property tested against them, with the requirement that a cached and an uncached build give the same verdict on the same transaction in the same state:

- **Producing blocks.** Transactions taken from the pool for inclusion. The stored result is reused and the state-dependent part re-runs against the head being built on.
- **Importing blocks.** Transactions arriving in a block from a peer, both when the node already pooled and validated the transaction, and when the block is the first time the node sees it.
- **Catching up from behind.** A node importing a long run of blocks, where most transactions were never in its pool, and where verifying keys may change part way through the run, through a protocol upgrade
or a contract update in an imported block.
- **Switching between forks.** A node abandoning one head for another. Transactions from the abandoned blocks are handled properly, and the state-dependent part must re-run against the new
head. A stored result checked against a contract key that exists only on the abandoned fork must not be reused.

In addition:

- A controlled comparison on the performance network, with arms matched on transactions per block, and on node and host age.
- Transaction progress notifications, against an unmodified build.
- Memory held by the cache, under sustained load.
- A transaction invalidated by an earlier transaction is still rejected.
- Contract upgrades with contract calls in one block.

## References

- [midnight-node#744](https://github.com/midnightntwrk/midnight-node/pull/744)
- [MPS-xxxx: ZK-Proof Verification Throughput Bottleneck](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/82) (number pending)
- The measurements quoted in this proposal are summarised in Appendix A.

## Appendix A: Experimental Evidence

Every measurement quoted in this proposal comes from one line of work on the Midnight performance network (perfnet) between 2026-07-29 and 2026-08-13, testing the implementation in [midnight-node#744](https://github.com/midnightntwrk/midnight-node/pull/744).
The work had three parts: a profile of block import that located the repeated checking, a controlled comparison that measured what removing it saves, and two earlier runs of the same branch that point the same way.
This appendix records how each was run, what was captured, and what was concluded.

### A.1 Common method

All timings were read from the journald logs of every node, not from aggregated dashboards.
Block authoring is the proposer's own timing of preparing a block.
Block import is the interval from the first announcement of a block to its import on the receiving node.
Blocks are paired by hash across nodes, so a block is compared against itself wherever it was imported, forks never cross-contaminate a measurement, and a node's import of its own block is excluded.
Blocks are then binned by the exact number of user transactions they carry, which removes block content as a variable, and a linear fit of time against transaction count gives the marginal cost per transaction directly.
Where two arms are compared, significance is an exact permutation test over the per-node means.

The workload throughout was transfer traffic only: NIGHT (unshielded) transfers and shielded token transfers, each consuming one UTxO and producing two.
Per-transaction figures are averages over that mix and would move under a different mix.

### A.2 Block import profile, 2026-08-07

**Setup.**
A three-validator mock network on one host, 4 vCPU and 8 GiB per node, running main plus the transaction timing instrumentation of [midnight-node#2003](https://github.com/midnightntwrk/midnight-node/pull/2003).
Load ran for three hours at about one transaction per second, 10,806 unique transactions, none invalid or dropped.

**Captured.**
Per-phase wall time of every block import, split into pre-dispatch validation, transaction application, post-block update, deserialisation, and the remaining Substrate machinery.
Hit and miss counts of the node's strict validation cache on each lookup path.
Per-transaction cost of validation at mempool admission and of ledger application.

**Found.**

| Measure | Value |
|---|---|
| Block imports observed | 3,664 |
| Ledger share of import wall time | 91.5% (260 ms of 284 ms average) |
| Proof check at mempool admission | about 29 ms per transaction |
| Proof check again at pre-dispatch | about 22 ms per transaction |
| Strict cache hits at pre-dispatch | 0 of 32,418 lookups |
| Strict cache hits at application | 32,272 of 32,418 lookups |
| Ledger application | about 13 ms per transaction |
| One full block import | 1,183 ms, of which 714 ms pre-dispatch and 318 ms application |

The cache missed on every pre-dispatch lookup because its key included the block timestamp, which differs between the mempool's validation and the block's.
Counting admission, pool revalidation (about 0.34 extra runs per transaction), and pre-dispatch on each of three nodes, every proof was checked about seven times across the network.
The conclusion was that making the cache tolerant of the state it is keyed on would roughly halve block execution cost on every node.
This is the observation the proposal acts on.

### A.3 Controlled comparison, 2026-08-13

**Setup.**
Eight validators and one RPC node on perfnet, all in one region, on identical hardware (4 vCPU, 8 GiB), two nodes of each arm in each availability zone.
Both arms ran on the same chain at the same time; every node authored its share of slots and imported every other node's blocks.
The window was 11.5 hours, blocks 17,743 to 24,649, with 21,717 transactions applied identically on all nine nodes.

| Arm | Nodes | Build |
|---|---|---|
| Cache | 4 validators | #744 branch with main merged on 2026-08-11 |
| Baseline | 4 validators | The fleet's previous build, a feature branch at main of 2026-08-05 |

The baseline was chosen because it was already deployed and resumed the existing chain database without a wipe.
It is not plain main: the arms differ by #744 plus one week of main drift.
Neither of the two changes in that drift plausibly produces a uniform per-transaction saving on two independent paths, but the confound is not zero and a rerun against main is listed under limits.

**Captured.**
Authoring time and import time for every block, with the user transaction count of each.
Critical-path CPU per node, P2P bandwidth in and out, peer count, finality lag, and announcement wire time.
Every validation run, rejection, error, and weight-limit hit logged by every node.

**Found.**

| Measure | Cache | Baseline | Change |
|---|---|---|---|
| Block authoring, loaded blocks | 585 ms | 1,015 ms | -42.4% |
| Block import, loaded blocks | 605 ms | 1,027 ms | -41.1% |
| Marginal cost per transaction, authoring | 29.3 ms | 51.5 ms | -22 ms |
| Marginal cost per transaction, import | 31.0 ms | 53.1 ms | -22 ms |
| Fixed cost per block, authoring | 8.1 ms | 8.3 ms | none |
| Fixed cost per block, import | 16.9 ms | 16.8 ms | none |
| Empty block authoring | 7.3 ms | 6.3 ms | +1 ms |
| Critical-path CPU per node over the window | 778 s | 1,258 s | -38% |

The fit had r squared of about 0.97 on both paths.
The entire gain is per transaction and it is the same 22 ms on two independent paths, which is the signature of one removed per-transaction operation rather than a diffuse speedup.
All four cache nodes beat all four baseline nodes on both metrics, giving p = 0.014, the smallest value the design allows.
The one cost is about one millisecond of cache bookkeeping on empty blocks.

**Controls.**
Splitting import time by both the receiving arm and the producing arm shows the gain is receive-side: what the importing node runs decides the cost, and who produced the block changes it by under 5 ms.
Announcement wire time was 1.8 ms on both arms, peers held at eight of eight, finality lag at 2.2 blocks, and no competing blocks appeared at any height.
The transaction-count bins were near-identically populated in both arms, so neither arm received easier work.
Rejections were equal across arms and all of one kind, the pool noticing a transaction already on chain.
No node logged an error, and the watcher failures seen in the July run of this branch did not recur.

**Throughput.**
Network throughput did not move.
Blocks capped at 24 user transactions on the block weight limit, hit 445 times on the cache arm and 447 on the baseline, so the limit filled first and identically.
Authoring consumed about 0.6 s of a 6 s slot on the cache arm against 1.1 s on the baseline.
Extrapolating the marginal cost, authoring alone would saturate a slot at about 205 transactions per block on the cache arm against 116 on the baseline.
At this load the change is not a throughput gain but a 2.4x rise in the per-transaction ceiling of the two hottest paths.
It converts to throughput once the weight limit is raised.

### A.4 Earlier runs of the same branch

**Multi-region comparison, 2026-07-29.**
Four validators on #744 against four on plain main, spread over four regions, 11 hours, 28,869 transactions, blocks carrying 7 to 19 user transactions.
Block authoring fell from 186.9 ms to 139.5 ms on average, a 13 ms saving per transaction, and the two distributions did not overlap.
Same-region import fell from about 180 ms to about 118 ms median.
Main nodes also paid a burst of about 80 ms of full revalidation after every import, visible as roughly 8,400 pool rejection logs per node against none on the cache arm.
Two findings limited this run.
The arms were region-confounded, with each arm holding one near and one far region.
The cache build showed failures in an external transaction watcher check that plain main did not, which is the origin of the open notification question in this proposal.

**Fleet rollout, 2026-07-31.**
All eight validators were moved to #744 to break the region confound.
Fleet block production fell from 1,018 ms to 893 ms.
Netting out drift between runs, the difference-in-differences effect of the code was 6.3 percentage points on production and 9.2 on import.
The former arm split collapsed and the remaining split in import time followed region, not build.

### A.5 Conclusions

- Proof verification is checked once per transaction per block on every node because the existing cache key depends on chain state, and the check is the largest single component of block import.
- Reusing the verified transaction and re-running only the state-dependent checks removes about one proof verification, 22 ms, from every transaction on both the authoring and the import path.
- The effect is per transaction, receive-side, and consistent across two runs with different baselines and different topologies.
- Consensus behaviour did not change: finality, peer counts, propagation, and block contents were the same on both arms.
- At the measured load the saving is headroom, not throughput, because the block weight limit binds first.

### A.6 Limits of the evidence

- The controlled comparison's baseline was a feature branch one week behind the cache arm's main, not main itself.
- Cache memory was not measured; perfnet ships no host metrics collector. The 100 to 400 MB figure in Security Considerations is an estimate from 2,000 entries at 50 to 200 KiB each.
- The cache hit and miss counters that would turn "one proof verification removed" from an inference into a measurement exist in the node but were not split by arm in the analysis.
- Mempool behaviour was not compared: pool tracing ran on one cache-arm node only.
- One load shape, one region. The results say nothing about behaviour where propagation rather than validation dominates, nor about contract-call traffic.

## Acknowledgements

- Oscar Bailey (@ozgb)
- Christos Palaskas (@chrispalaskas)
- Michal Skowron (@mpskowron)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
