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
MPS: none
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
- Throughput exploration report, entry 0003, Proof Verification Cache: the hypothesis, the runs, and the measurements quoted here.
- Throughput exploration report, block import profile of 2026-08-07: the repeated checking and the cache miss counts. //TODO add a link to the report when it is publicly available


## Acknowledgements

- Oscar Bailey (@ozgb)
- Christos Palaskas (@chrispalaskas)
- Michal Skowron (@mpskowron)

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
