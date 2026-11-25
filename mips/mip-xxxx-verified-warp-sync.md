---
MIP: "xxxx"
Title: Verified Warp Sync for Midnight Full Nodes
Authors:
  - Justin Frevert (@justinfrevert)
Status: Draft
Category: Networking
Created: 2026-11-09
Requires: none
Replaces: none
MPS: MPS-0032, MPS-0048
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

This proposal establishes verified warp sync as a supported way to bootstrap a Midnight full node. A joining node verifies GRANDPA finality, downloads the runtime state at the verified block, and reconstructs the Midnight ledger arena committed to by that state. It can then validate new blocks without first executing the chain from genesis.

Midnight requires an extension to Substrate warp sync because its runtime trie contains a commitment to the ledger arena, rather than the arena itself. Recovering the trie alone leaves a node unable to execute the next block. The existing ledger-sync implementation supplies the missing data and verifies it against that commitment. This proposal formalizes that verification rule and the node behavior on which it depends.

It also specifies the improvements required for production support: coordinated checkpoint selection, bounded snapshot preparation and serving, retries that do not punish honest recovery, bounded resource use, durable import, recovery supervision, pruning integration, and end-to-end verification and performance evidence.

Warp sync retains Midnight's existing finality assumptions. It does not independently re-execute the skipped history, replace historical data availability, or provide a succinct or quantum-secure light-client proof. It contributes to the full-node bootstrap objectives of MPS-0032 and MPS-0048 while leaving their broader requirements to other proposals.

## Motivation

A node that must execute every historical block before participating pays an onboarding cost that grows with chain history. This affects new operators, replacement validators, RPC capacity, disaster recovery, and development networks. Restoring a database snapshot can reduce that cost, but a database archive alone does not demonstrate that its state belongs to the intended finalized chain.

Substrate already supports bootstrapping from finalized state. Midnight adds a second requirement: the state behind `pallet_midnight::StateKey` is stored as a content-addressed ledger graph outside the runtime trie. Putting those nodes in additional columns of the same physical database does not put them inside the trie. Both state domains must be recovered at the same block.

[Issue #1648](https://github.com/midnightntwrk/midnight-node/issues/1648) recorded the failure of ordinary warp sync. [PR #1650](https://github.com/midnightntwrk/midnight-node/pull/1650) supplied ledger-arena recovery, import gating, compression, and root verification. That establishes a functional baseline, but successful bootstrap alone does not establish acceptable performance or resistance to hostile peers.

The review record identifies expensive serialization for arbitrary requested hashes, incomplete abuse controls, and a retry design that conflicts with proposed replay bans. A [perfnet report](https://github.com/midnightntwrk/midnight-node/pull/1650#issuecomment-5203998415) also recorded 4,256 seconds for verification and import of a 41,421,163-byte decompressed snapshot. That is a historical observation on a particular build and machine, not a current benchmark or a demonstrated root cause. It makes profiling and repeatable performance qualification prerequisites for production support.

This MIP makes complete warp sync a maintained node capability, with explicit safety, availability, and operational requirements. It does not change the default sync mode merely by being accepted.

## Specification

The terms MUST, MUST NOT, SHOULD, and MAY express requirements of this proposal. Sections describing the existing version-2 protocol record the baseline; the remaining requirements describe the proposed hardened implementation. Suggested numerical defaults below are design choices for review, not measured results.

### 1. State and verification model

Let `N` be a block hash and `H_N` its header. Let `T_N` be the runtime state authenticated by `H_N.state_root`, and `K_N` the complete typed ledger key stored in `Midnight.StateKey` in `T_N`.

A successful recovery MUST establish all of the following:

1. `H_N` belongs to the finalized chain selected by the node's configured trust anchor and verified GRANDPA authority transitions.
2. The downloaded runtime state, including child tries needed for execution, matches `H_N.state_root` through the normal verified state-sync path.
3. `K_N` is read from that verified state at exactly `N`. A key supplied by a snapshot peer MUST NOT replace it.
4. The received ledger graph is decoded using the supported format identified by `K_N`. Its structure and content addresses are independently verified, and its reconstructed typed root equals `K_N`.
5. The complete reachable ledger graph and its required persistence metadata are durably installed, so execution against `N` can resolve the state after restart.

These checks authenticate different layers:

```text
configured chain / finality anchor
  -> verified GRANDPA transitions and finalized header N
  -> verified runtime state rooted at H_N.state_root
  -> Midnight.StateKey at N
  -> verified, complete ledger arena rooted at that key
  -> ordinary block validation after N
```

The ledger root MUST NOT stand in for the whole runtime state. Warp sync MUST preserve the normal consensus, inherent, and transaction checks on subsequent executed blocks. It imports an authenticated checkpoint; it does not prove that the joining node re-executed the preceding transactions.

### 2. Supported node behavior

Full replay and verified warp bootstrap MUST remain supported choices. A warp-capable node MUST support ledger recovery as a client whether or not it serves snapshots.

Non-validator nodes SHOULD serve prepared snapshots by default, subject to local resource limits. Validators MUST require an explicit serving opt-in, preserving the protection supplied by `--serve-warp-ledger-sync`. A serving opt-out for other operators MAY be provided. Serving configuration MUST NOT disable client-side recovery or change the requirements for serving GRANDPA proofs.

A server MUST advertise a prepared snapshot only while it holds the associated runtime state, finality evidence, and complete ledger data. Clients MAY retrieve these state domains from different peers, subject to the same verification rules. An advertisement is a capability claim, not a trust anchor. An incomplete node MUST NOT serialize or advertise an arena it has not recovered.

### 3. Checkpoint selection and retention

The hardened design uses a small set of locally prepared, finalized session checkpoints. This is the proposed resolution of [#1723](https://github.com/midnightntwrk/midnight-node/issues/1723).

For a session transition, the checkpoint is the first canonical block whose post-state has the new session index. Session changes MUST be detected from authenticated runtime behavior or an appropriate runtime API, rather than assuming fixed block heights or a hard-coded runtime `spec_version`. A checkpoint becomes eligible only after finality and after the server has sufficient evidence to prove its finality. A later finalized descendant plus verified ancestry MAY supply that evidence when no justification targets the checkpoint itself.

The proposed default is to retain the latest three successfully prepared checkpoints. Servers MAY retain a smaller set when their storage budget cannot hold three; they MUST advertise the actual set. A fresh server MAY have an empty set until preparation completes. Building a replacement MUST NOT discard the last usable checkpoint before the replacement is ready.

Preparation MUST follow local finalized session transitions, with at most one successful serialization per checkpoint and a bounded retry schedule for local failures. Remote requests MUST NOT create preparation jobs, choose arbitrary historical roots, or evict prepared snapshots. A single preparation job per server is the default; queued work, retained bytes, and preparation duty cycle MUST also be bounded. If preparation falls behind, intermediate checkpoints MAY be skipped.

The server MUST verify canonical membership, not merely that a requested block's number is below the finalized height. It MUST pin the runtime state and ledger graph against pruning while preparing a snapshot. Advertised checkpoints MUST remain usable for the server's configured transfer grace period, including their runtime state and finality evidence. Pins, grace periods, and active transfers MUST have finite global limits; a client cannot extend retention indefinitely by sending another request. A server with insufficient retention capacity MUST stop advertising a checkpoint before accepting new transfers for it.

The client first discovers available checkpoints, verifies finality for its chosen `N`, and directs runtime state sync and ledger recovery to that same `N`. Selecting a session checkpoint only for the arena while state sync ends at a different block is invalid. Advancing to another checkpoint requires selecting and verifying a new complete recovery target; the client MUST NOT compare one block's arena with another block's trie key.

Using the SDK's `WarpSyncConfig::WithTarget` with a peer-supplied header is prohibited unless that header has already passed finality verification: this configuration skips proof download and assumes a trusted header. The implementation MUST retain verified authority-set context when handing an authenticated target to state sync.

### 4. Baseline transfer format

The existing protocol name is:

```text
/{lowercase_hex_genesis_hash}[/{fork_id}]/midnight-ledger-sync/2
```

Its SCALE fields are encoded in this order:

| Message | Fields |
| --- | --- |
| Request | `target_hash: Hash`, `offset: u64`, `max_len: u32` |
| Response | `compressed_total_len: u64`, `raw_total_len: u64`, `offset: u64`, `bytes: Vec<u8>` |

Offsets address a single raw-Snappy compressed stream, not decompressed bytes. The decompressed payload is a tagged serialization rooted at `Ledger`: `GLOBAL_TAG || Ledger::tag() || ':' || TopoSortedNodes`. It is not an unwrapped `LedgerState` export or a physical database backup. Child nodes precede parents. The ledger serializer defines graph encoding and normal-form validation; interoperable fixtures MUST identify its exact version.

Version 2 caps a response's data at 1 MiB and the decompressed payload at 1 GiB. It checks the compressed size against Snappy's maximum compressed length for the advertised raw size. A client requests full-size ranges, except for the final remainder, and rejects an undersized response. Its implementation returns empty data for requests at or beyond the end; this historical behavior is not an abuse-control mechanism.

The baseline is documented for compatibility. The hardened protocol below MUST use a new version, because discovery, explicit errors, and transfer identity change the wire contract.

### 5. Hardened protocol and transfer identity

The proposed protocol suffix is `midnight-ledger-sync/3`, using the same genesis and fork namespace. It retains raw-Snappy and the tagged ledger payload, while adding checkpoint discovery, target-specific finality retrieval, and an immutable transfer descriptor.

The following describes SCALE encoding. Enum discriminants are the listed `u8` indices; integers use the normal SCALE fixed-width encodings, vectors use SCALE length prefixes, and fields appear in listed order. `Hash` and `Header` use the network's block types. Decoders MUST consume the entire message and enforce length and allocation limits before materializing vectors.

`Descriptor` contains `target_hash: Hash`, `target_number: u32`, `session_index: u32`, `ledger_tag: Vec<u8>`, `raw_total_len: u64`, `compressed_total_len: u64`, and `blob_hash: [u8; 32]`, in that order. `blob_hash` is SHA-256 of the complete compressed stream. `descriptor_id` is SHA-256 of the bytes `midnight-ledger-sync/3/descriptor` followed by the SCALE-encoded descriptor. The tag is limited to 256 bytes. Transport version 3 fixes compression and payload framing as described above; a change to those conventions requires a new negotiated version.

| Request index | Fields | Successful response index and fields |
| --- | --- | --- |
| 0: ListTargets | none | 0: Targets, `descriptors: Vec<Descriptor>` |
| 1: Finality | `target_hash: Hash`, `begin_hash: Hash` | 1: Finality, `target_hash: Hash`, `warp_proof: Vec<u8>`, `ancestry: Vec<Header>` |
| 2: Range | `descriptor_id: [u8; 32]`, `offset: u64`, `max_len: u32` | 2: Range, `descriptor_id: [u8; 32]`, `offset: u64`, `bytes: Vec<u8>` |

Response index 3 is `Error`, containing `code: u8` and `retry_after_ms: u32`. Codes are 0 `UnknownTarget`, 1 `NotReady`, 2 `Busy`, 3 `InvalidRange`, 4 `UnsupportedFormat`, 5 `TooLarge`, 6 `ProofUnavailable`. The retry value is a bounded scheduling hint, not a commitment or a reason to exceed the client's deadline.

`Targets` contains at most three entries, newest first, with no duplicate target hashes. A server retaining additional checkpoints MUST still expose at most three through this message. Descriptor fields are untrusted hints until checked against verified headers, state, and payload. In particular, neither the transport digest nor the session index establishes chain membership.

`Finality` reuses the GRANDPA warp-proof encoding and verification rules of the SDK pinned by the implementation. Responses can be partial; the client retains the successfully verified authority transitions and requests the next fragment from the last verified hash, keeping the same selected target. Completed evidence MUST authenticate exactly the selected checkpoint. If the warp proof terminates at a later finalized header, `ancestry` supplies the contiguous headers back to the checkpoint, with hashes and numbers checked at each link. No authority transition needed to verify that later header may be omitted. Empty or non-advancing fragments MUST NOT reset progress deadlines. Proof serving and caching require their own resource bounds; moving serialization out of the range handler does not make proof generation free.

Range requests MUST refer to a retained prepared descriptor. Valid requests have `0 < max_len <= 1 MiB` and `offset < compressed_total_len`. Success returns exactly `min(max_len, compressed_total_len - offset)` bytes and echoes the requested identity and offset. Invalid ranges MUST be rejected before snapshot work or response-data allocation. All arithmetic MUST be checked before conversion to platform allocation sizes.

The initial profile retains the baseline 1 GiB raw ceiling and Snappy compressed ceiling. Requests are limited to 1 KiB. Targets and error responses are limited to 4 KiB; range responses to 1 MiB plus 128 bytes; finality responses, including ancestry and framing, to 8 MiB. Finality proofs too large for one response require verified continuation, rather than increasing the limit. Implementations MUST also bound decoded header count, nesting, graph node count, and aggregate memory; an encoded-byte limit alone is insufficient.

Clients MUST check descriptor stability, offsets, exact range lengths, total lengths, the completed compressed digest, Snappy's advertised decompressed length, the actual decompressed length, and ledger-root equality. The digest detects transfer inconsistency before expensive ledger reconstruction, but is not a substitute for `K_N`.

Clients MUST retain partial progress for retries of the same descriptor, including after a transient disconnection. They MAY resume with another peer only when its descriptor is identical. A changed compressed representation requires a new transfer even if its eventual ledger root would be the same. Multi-peer scheduling is optional. Without authenticated per-chunk commitments, a failed mixed-peer transfer does not establish which peer supplied the bad range; clients MUST NOT impose a cryptographic-misbehavior penalty on an arbitrary contributor. A bounded single-peer retry can establish attribution.

### 6. Resource limits, fairness, and retries

Prepared snapshots MUST be immutable while served. Range handlers MUST NOT serialize the ledger or hold the arena lock. Snapshot preparation, finality proof generation, range serving, decompression, and ledger import MUST run under separate bounded work budgets, with expensive synchronous work kept off shared async workers.

Serving MUST enforce both per-peer and global limits on request rate, concurrent work, queued bytes, response bytes per interval, and retained snapshot bytes. Limits MUST refill over time and charge actual work or egress. Cache eviction, offset churn, reconnects, and new peer identities MUST NOT reset the global budget. Cache replacement is a server decision and MUST NOT be blamed on an honest client as target cycling.

Repeated valid ranges, including offset zero, are idempotent requests. They consume the normal bandwidth and request budgets and MAY receive `Busy`; repetition alone MUST NOT cause a permanent ban. Invalid encodings and attributable invalid data SHOULD affect peer reputation. Ordinary pruning, unavailable targets, overload, network failure, and unsupported local formats MUST NOT be treated as evidence of malicious behavior.

Clients MUST bound discovery attempts, first-response time, per-range time, whole-transfer time, cumulative retry work, and verification work. A proposed initial transfer budget is `40 seconds + ceil(compressed_bytes / 256 KiB per second)`, with no individual range allowed more than 40 seconds. These values require qualification on the supported network profile. Deadline calculations MUST use checked or saturating durations. A zero remaining budget terminates the attempt normally.

Import has a separate finite work and stall budget informed by profiling. Dropping a future or timing out a blocking-task handle MUST NOT permit a still-running importer to race another importer, block execution, or shutdown cleanup. Cancellation requires cooperative cancellation or isolation with a safe completion boundary.

### 7. Recovery, persistence, and readiness

Recovery MUST have explicit states: target selection, finality verification, runtime-state download, ledger download, ledger verification, durable installation, ready, and failed. A shared gate MUST prevent authoring and ledger-dependent block execution before installation completes. Startup MUST establish the gate before those consumers can observe a recovered trie with a missing arena; polling alone MUST NOT leave an unprotected interval.

The import gate MUST allow the non-executing state-sync target and historical gap imports to complete. It MUST defer imports that would execute the runtime, including `ExecuteIfPossible` when parent state is present. Deferral MUST preserve the sync engine's queued-block bookkeeping and MUST NOT generate repeated identical block downloads as its retry mechanism. Ledger-dependent RPC and transaction-pool operations MUST report recovery as unavailable or use an equivalent safe gate.

Graph reconstruction MUST use a deserializer designed for untrusted input, including content-hash recomputation, child-reference validation, type checks, and normal-form checks. Unverified data MUST NOT become an accepted or retained chain-state root. Temporary writes, if used for bounded-memory import, MUST remain unreachable as accepted state and have bounded cleanup. After equality with `K_N` is established, the importer MUST persist the complete reachable graph, record the root according to the active storage/GC contract, and complete the required durability barrier before marking ready.

A durable recovery record SHOULD bind chain identity, target hash, typed root, storage format, and installation completion. Equivalent database invariants MAY replace that record if crash tests demonstrate the same guarantee. Finding only the root node is not sufficient evidence that all descendants survived a partial installation. Re-import after a crash MUST NOT accumulate duplicate permanent root references or leak incomplete attempts indefinitely.

Recovery MUST be supervised. An unexpected monitor or importer failure MUST produce an explicit recovery failure or stop the node; it MUST NOT leave an apparently healthy process with no task capable of completing recovery. A normally completed no-op monitor MUST NOT be mistaken for an essential-task failure.

The default policy after the configured overall recovery deadline is a visible failed state, continued gating, and a nonzero process exit with actionable diagnostics. An operator MAY explicitly choose continued waiting or a restart into full replay. Falling back MUST reconcile the installed trie and arena using supported database initialization; executing from a partially recovered target or silently lowering verification requirements is prohibited.

Readiness MUST distinguish networking availability, recovery completion, and subsequent catch-up. Metrics and logs MUST expose target, phase, transferred bytes, verified nodes or batches, persistence progress, last-progress time, retries, limit rejections, and terminal failure. A timer heartbeat alone MUST NOT claim progress. Warp completion does not by itself make an old checkpoint current.

### 8. Ledger upgrades and garbage collection

The supported decoder MUST be selected from the authenticated typed root and ledger-provided format metadata. Unknown versions MUST fail explicitly. Peer metadata and the newest ledger compiled into the node MUST NOT override the checkpoint's version. Manual substring parsing and a second independently maintained format table SHOULD be replaced with a ledger-owned capability registry and fixtures.

Any new runtime API used by this feature MUST be versioned, and node behavior MUST branch on its API version. Any new runtime dependency on node host behavior MUST use a versioned host function. A hard-coded `spec_version` threshold is not an acceptable capability gate.

The proposal MUST work with the storage model being deployed, including the pending unified-storage and GC changes. A GC wrapper such as `TaggedRoot` MUST NOT change the root compared with the on-chain `StateKey`; it is persistence metadata around that root. Its block or parent tag MUST be derived from the authenticated target header. Preparation pins, active recovery roots, and installation records MUST participate in GC liveness. Retirement of a transfer MUST release its pins without releasing a chain-live root.

Warp bootstrap supplies current execution state, not an archive. Capability reporting MUST distinguish retained runtime state, ledger checkpoints, and historical block bodies. Indexers and other consumers requiring arbitrary history must obtain it separately.

## Rationale

The existing commitment chain is sufficient to authenticate the recovered arena under Midnight's finality assumptions. Adding a trusted snapshot operator would introduce an unnecessary source of authority. Putting all ledger nodes into the runtime trie would be a much larger storage and runtime redesign; physical database unification does not itself solve authenticated transfer.

Session checkpoints make expensive work depend on local chain progress rather than arbitrary remote hash selection. Preparing them before advertising availability also keeps range requests short and predictable. Three retained targets and a bounded grace period provide room for ongoing transfers across a session change without allowing unbounded retention. The exact operating limits remain subject to the qualification required below.

A larger arbitrary-hash LRU and per-peer serialization counters do not remove the cache-busting attack: adversaries can use more targets or identities, while honest transfers can be charged for server-side eviction. Structural bounds come first; refillable budgets and reputation provide additional protection.

Retry-safe ranges match real network behavior. Banning a repeated request confuses packet loss and process restarts with abuse. Transfer identity prevents a resumed download from accidentally combining different compressed streams, while the authenticated ledger root remains the acceptance criterion.

Blocking every import would also block the state required to recover. Returning `MissingState` or repeatedly failing executable imports has already been documented to wedge sync bookkeeping or trigger duplicate-request bans. Holding only the imports that would execute retains the working baseline's essential property.

## Path to Active

### Acceptance Criteria

This MIP can be marked Implemented only when:

1. The baseline commitment chain and proposed checkpoint/transfer rules are implemented, documented, and covered by reproducible protocol vectors. Before transition to Proposed, maintainers review the version-3 encoding, session convention, finality continuation design, and numerical resource profile.
2. Real multi-node tests bootstrap from empty storage, verify both state domains, import subsequent blocks, and reach finality. A node with wrong finality, wrong trie state, or a mismatched arena never becomes ready.
3. Retry, overload, target-cycling, malformed-input, restart, disk-failure, and GC cases in Testing pass through the production handlers and recovery driver.
4. A repeatable performance report identifies the cause or disposition of the historical import stall and measures every recovery phase on representative mainnet-sized state and a larger growth fixture. It records hardware, build, state size and graph size, pruning configuration, CPU, RSS, I/O, and concurrent load. A proposed qualification target is verified durable installation of a 512 MiB raw arena within ten minutes on a 4-vCPU, 8-GiB reference node, with measured peak RSS below 6 GiB and a documented disk/network profile. This is an initial target to validate or revise before Proposed, not a claimed result.
5. Stress tests demonstrate bounded preparation count, memory, queues, and egress under multiple hostile identities while honest clients continue to progress within the declared service capacity. Concurrent serving does not prevent block import or finality duties.
6. Node maintainers complete security review of finality-to-checkpoint binding, untrusted decoding, gate ordering, crash durability, and pruning lifetimes. Outstanding relevant findings from #2014 are resolved or explicitly superseded with evidence.

Active additionally requires release and deployment of the qualified node capability on the intended Midnight network, working snapshot-serving capacity, published operator instructions, and verification on the released storage layout. Editors determine the status transition under MIP-0001. Acceptance alone does not certify an existing build as production ready.

### Implementation Plan

1. Document and pin the existing version-2 baseline, including tests of its current verification and import behavior.
2. Instrument and profile decompression, deserialization, allocation, root verification, persistence, flushing, and lock waits. Fix the measured bottleneck while retaining untrusted-input checks. Do not optimize by trusting remote hashes.
3. Implement prepared session checkpoints, retention pins, target-specific verified finality, discovery, and coordinated state-sync targeting. Review the new wire version before rollout.
4. Add bounded fair serving, retry-safe transfers, finite deadlines, supervised recovery, safe installation, and progress reporting. Reuse sound parts of #2014 where applicable; its proposed replay-ban behavior is superseded by this specification.
5. Integrate with the storage/GC release, execute the full fault and performance matrix, and deploy serving capacity before promoting warp sync as a supported production bootstrap mode.

## Backwards Compatibility Assessment

This proposal preserves block validity, ledger commitments, transaction semantics, and the existing finality model. The intended implementation is a node networking and storage change; no consensus hard fork is inherently required. If implementation introduces a runtime API, its versioning and metadata changes must be handled explicitly.

Version 3 is separately negotiated. Version-2 support MAY remain during transition, but legacy requests MUST NOT bypass the hardened server's prepared-target and resource limits. A version-2 client whose arbitrary target is unavailable may therefore be unable to complete warp sync against that server. Documentation must state that limit. A hardened client MUST NOT silently downgrade to a trusted-header path or an unsafe legacy serving policy; full replay remains a supported operator choice.

This MIP does not itself require erasing existing databases. The separate storage bundle in [#2332](https://github.com/midnightntwrk/midnight-node/pull/2332) may require a resync for its own format changes. Deployment must qualify warp sync on that resulting format rather than treating those changes as already merged or assuming physical layout compatibility from an earlier test.

## Security Considerations

**Finality and chain selection.** Authentic state depends on the correct chain configuration, authority-set transition verification, and GRANDPA's weighted supermajority safety assumptions. Retired-key compromise, long-range histories, forced authority changes, and community-authorized chain resets require the same explicit anchor and recovery policies as the node's finality verifier. Multiple peers do not constitute a proof of the correct history. Freshness and resistance to eclipse attacks are separate from a snapshot's root correctness.

**Untrusted data.** Every peer-supplied descriptor, proof, length, tag, and graph is untrusted until checked at its corresponding layer. Decoding, decompression, graph expansion, staging writes, and repeated failed verification all require budgets. Native hash verification protects integrity only if parsing and graph validation are correct; it does not by itself protect availability or memory.

**Availability and resource exhaustion.** Canonical prepared targets eliminate remote choice of serialization work. Global limits remain necessary because peer identities are cheap. Retention pins, proof requests, range retries, and temporary imports are all bounded. Fair scheduling and reserved capacity protect ordinary node duties. At least one reachable source of valid state is still necessary for progress; this MIP provides no new data-availability guarantee.

**Partial installation and concurrency.** The recovery gate remains closed until durable installation. An importer that has timed out but still holds storage access is not safely cancelled. Crash recovery and GC must preserve graph completeness and prevent use of a bare root whose descendants are absent. Local storage errors are reported as local failures, rather than punished as peer misconduct.

**History and privacy.** Warp sync accepts authenticated finalized state without locally checking each skipped transaction proof. It neither replaces historical auditing nor justifies deleting data needed by wallets or indexers. Only public node ledger state is transferred; implementations MUST NOT include wallet secrets, validator keys, private configuration, or a physical data-directory archive in the payload.

**Relationship to MPS-0048.** This MIP addresses the joining-node use case in part. It does not provide constant-size committee proofs, constant verification work over chain age, post-quantum signatures, a new whole-state signing duty, equivocation reporting/key banning, or node-free catch-up from an old tip. The existing runtime-state root plus `StateKey` binding can inform later designs, but those requirements remain open.

## Implementation

The reference baseline is `midnightntwrk/midnight-node` at `ab25b169a88ca96b5491119c201e7cb4f9ed572b`. Its `node/src/warp_ledger_sync/` module supplies protocol, server, client, monitor, oracle, and block-import gating. Ledger serialization and verified import live in `ledger/src/lib.rs` and the version-specific ledger modules. `node/src/service.rs` wires serving, warp proofs, and recovery into consensus and networking. Partner Chains verifier handling of non-executing state imports must be preserved.

Checkpoint selection and target-specific finality need integration with the SDK warp provider/state-sync machinery, not just a new ledger server cache. The implementation must retain GRANDPA justifications and authority-change evidence wherever warp proofs are served, independently of the ledger-serving flag. A node whose old pruning removed that evidence must report the capability gap; a new retention rule cannot recreate missing proofs.

The pending storage work in #1991, #2050, #2300, #2332, and #2333 must be reconciled with recovery's persistence and pruning contract. These are implementation dependencies to coordinate, not claims that the PRs have landed or additional MIPs this document supersedes.

## Testing

Tests MUST exercise production request handlers, peer selection, recovery, and block import. Helper round trips alone are insufficient.

| Area | Required evidence |
| --- | --- |
| Authenticity | Invalid finality; omitted authority transition; wrong ancestry; stale-fork hash below finalized height; wrong trie or child trie; valid graph with wrong root; type/tag mismatch; malformed or non-normal graph; trailing data; truncation. All rejected without readiness. |
| Complete bootstrap | Empty database, non-empty arena, state sync plus ledger recovery, next-block execution, transaction validation, and finalized catch-up; compare roots with a replayed control node at the same height. |
| Consensus compatibility | AURA, BABE, both sides of the migration and a recovery crossing it; Partner Chains state-import bypass; supported old/new ledger formats; unsupported format fails explicitly. |
| Gate ordering | Fast local state sync, task scheduling delays, restart with recovered trie/missing arena, gap imports, tx pool/RPC consumers, import backlog, and monitor failure. No execution before installation and no import dependency deadlock. |
| Retry and fairness | At least three honest offset-zero retries against a sole server; lost ranges; reconnect/resume; identical and changed descriptors; mixed-peer corruption without false attribution; checkpoint rotation during transfer; concurrent honest clients under overload. |
| Abuse bounds | Alternating arbitrary targets; many identities; out-of-range offsets; zero/tiny ranges; proof-query churn; malformed lengths; decompression bombs; graph allocation amplification. Assert preparation counts, memory, queues, and bandwidth remain within configured bounds. |
| Deadlines | First-response and whole-transfer expiry under a controlled clock; expiry while descheduled; non-advancing proofs; stalled import; cancellation while a blocking worker remains alive. No panic, unsafe retry, or silent indefinite gate. |
| Durability and GC | Kill/restart during every installation boundary; process-level reopen; disk full and write/flush failure; partial graph with root present; repeated import reference counts; GC during preparation/import and after pin release; canonical and stale roots. |
| Deployment | Full warp run on the released unified/GC layout, legacy layout where still supported, state/ledger pruning boundaries, validator-only network with and without a serving opt-in, and independently configured GRANDPA-proof serving/retention. |
| Performance | Cold/warm preparation, transfer, decompression, verification, durable import, catch-up, growth fixtures, and concurrent service. Publish measured progress and resource use, not just a periodic heartbeat. |

CI MUST run an actual multi-node warp bootstrap and subsequent-block test. Larger resource and fault tests MAY run on a scheduled qualification network, with reproducible commands and archived results required for release.

## References

- [MIP template](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-template.md) and [MIP-0001](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0001-mip-process.md).
- [MPS-0032: History Management for Midnight](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0032-storage-management.md).
- [MPS-0048: Enabling Light Clients on Midnight](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0048-enabling-light-clients.md).
- [MIP-0010: AURA to BABE](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0010-aura-to-babe-migration.md) and [MIP-0026: Interim Ledger State Management](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0026-interim-ledger-state.md).
- [#1648: Warp sync doesn't work](https://github.com/midnightntwrk/midnight-node/issues/1648), [#1650: Add ledger arena for warp sync](https://github.com/midnightntwrk/midnight-node/pull/1650), and [#2014: Warp sync hardening](https://github.com/midnightntwrk/midnight-node/pull/2014).
- [#1723: Session targets](https://github.com/midnightntwrk/midnight-node/issues/1723), [#1687: Snapshot size](https://github.com/midnightntwrk/midnight-node/issues/1687), and [#1816: Ledger format discovery](https://github.com/midnightntwrk/midnight-node/issues/1816).
- [Reference warp-sync implementation](https://github.com/midnightntwrk/midnight-node/tree/ab25b169a88ca96b5491119c201e7cb4f9ed572b/node/src/warp_ledger_sync) and [ledger dispatch](https://github.com/midnightntwrk/midnight-node/blob/ab25b169a88ca96b5491119c201e7cb4f9ed572b/ledger/src/lib.rs).
- [Pinned SDK warp configuration](https://github.com/shieldedtech/polkadot-sdk/blob/9c7455eabd0cc2e77da867e837dfb466b11fbcf6/substrate/client/network/sync/src/strategy/warp.rs) and [GRANDPA warp proofs](https://github.com/shieldedtech/polkadot-sdk/blob/9c7455eabd0cc2e77da867e837dfb466b11fbcf6/substrate/client/consensus/grandpa/src/warp_proof.rs).
- [#2332: Storage bundle](https://github.com/midnightntwrk/midnight-node/pull/2332), [#1991: Ledger GC](https://github.com/midnightntwrk/midnight-node/pull/1991), [#2050: Live intra-block ledger states](https://github.com/midnightntwrk/midnight-node/pull/2050), [#2300: Unified-storage ledger column options](https://github.com/midnightntwrk/midnight-node/pull/2300), and [#2333: Unified storage](https://github.com/midnightntwrk/midnight-node/pull/2333).

## Acknowledgements

The prior implementation and review discussions include contributions from Giles Cope (@gilescope), @Klapeyron, @chrispalaskas, @m2ux, and the Midnight node and ledger teams. Inclusion here records that prior work and does not imply endorsement of this draft. This text was drafted with AI assistance; the named human author remains responsible for review and submission.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for contributions to the Foundation.
