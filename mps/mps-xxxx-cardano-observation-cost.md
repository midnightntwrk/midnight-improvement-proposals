---
MPS: xxxx  
Title: Operational Cost of Cardano Observation  
Authors: Santiago Carmuega @scarmuega  
Status: Draft  
Category: Core  
Created: 01-SEP-2026  
Requires: none  
Replaces: none  
MIP: none  

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

Every Midnight node reads a small, well-defined slice of Cardano state: the UTxOs at a handful of script addresses, block lookups by hash and timestamp, and two per-epoch values (nonce and pool stake). The only supported way to obtain that slice is a full Cardano archive: cardano-node, cardano-db-sync and a tuned PostgreSQL 17 instance. db-sync is a general-purpose indexer whose own documentation describes it as a one-size-fits-all backend. Its maintainers recommend 64 GB of RAM, 700 GB of disk and 60k IOPS for mainnet, and a mainnet sync takes about two days. Midnight's follower touches eleven of its roughly eighty tables and cannot use the footprint-reducing options db-sync offers, because the queries depend on ledger-derived tables and full output history.

The Cardano dependency is therefore the largest single item in a Midnight node's operating envelope, and it is paid by every node role (validator, full, boot, RPC) regardless of how little Cardano data that role consumes. This MPS documents the read surface, the required stack and its cost as stated by official sources, and the operational burden that follows. It asks for a Cardano-observation dependency whose cost is proportional to the data consumed, under an explicit freshness and trust contract, so that a lighter observer can be adopted without weakening consensus guarantees.

## Vision

A Midnight node's Cardano observation costs roughly what the observed data costs. The protocol states what the node reads from Cardano and under which freshness and trust assumptions, and any observer that meets that contract is acceptable. Operators choose an observer by that contract, not by the product named in a setup guide. A Cardano stake pool operator adds a Midnight validator to the hardware they already run without adding a database tier. Testnets, CI and local development bring up the Cardano dependency in minutes and single-digit gigabytes. New Cardano-side observation features (governance events, bridge, staking) extend the read surface without re-imposing an archive on every node.

## Problem

### What a Midnight node reads from Cardano

Midnight is a partner chain. Its node consumes what the Partner Chains toolkit calls "Cardano observability data" through six data sources: four defined by the toolkit — stable-block and block-by-hash lookups, the latest-block view for RPC, committee-selection inputs (D-parameter, candidate registrations, epoch nonce, pool stake), and token-bridge transfers — and two Midnight-specific ones, cNIGHT observation and federated-authority observation. Appendix A itemizes each source, its interface and the data it reads, at a pinned commit of midnight-node.

Everything the six sources read is one of three shapes: UTxOs at known addresses or under known policy ids, with datums — served either as current state or as an event stream in block order from a checkpoint (at genesis generation the cNIGHT scan starts from the beginning of the chain); block metadata by hash, number or time; and two per-epoch values. There is no transaction history, no address balance, no reward history, no metadata search. The consumers are inherent data providers, so every node must observe the same data ([`docs/c-to-m-bridge.md`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/docs/c-to-m-bridge.md): "There has to be a consensus across nodes regarding observed Cardano events and their reflection in the Midnight chain"). Committee selection reads Cardano data "offset by 2 mainchain epochs (finalized)" ([MIP-0010](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0010-aura-to-babe-migration.md)), so the freshness the protocol needs is measured in days.

In db-sync terms, the queries behind these sources touch eleven of the roughly eighty tables db-sync maintains (Appendix A lists them).

### What the node is required to run

Official sources make one observer mandatory for every node role:

- [docs.midnight.network/nodes](https://docs.midnight.network/nodes): "Requires persistent connection to Cardano via PostgreSQL database populated by Cardano-db-sync". The architecture diagram there and in the [midnight-node README](https://github.com/midnightntwrk/midnight-node#readme) is Cardano Mainchain → Cardano Indexer (db-sync) → PostgreSQL (cexplorer) → Midnight Node, annotated "Observes mainchain state / Queries Cardano data (cNIGHT, governance)".
- The [full node](https://docs.midnight.network/nodes/full-node), [boot node](https://docs.midnight.network/nodes/boot-node) and [RPC node](https://docs.midnight.network/nodes/rpc-node) pages each list "Cardano-db-sync instance set up with accessible PostgreSQL port" as a prerequisite and require `DB_SYNC_POSTGRES_CONNECTION_STRING`.
- [docs.midnight.network/nodes/cardano-node](https://docs.midnight.network/nodes/cardano-node): "To maintain synchronization with the Cardano blockchain, the Midnight node requires a persistent connection to a PostgreSQL database populated by Cardano-db-sync."
- Partner Chains [`docs/intro.md`](https://github.com/input-output-hk/partner-chains/blob/e514eb79/docs/intro.md): "Unlike db-sync, which is always mandatory, ogmios is only required when interacting with the smart contracts as chain builder, or when registering as SPO."
- The node binary itself admits only two source families: the db-sync/Postgres implementations, or a mock that reads fixed data from a JSON file for tests and local development. There is no other production path and no configuration point for selecting one (Appendix A).

The stack per node is therefore cardano-node 11.0.1 (bootstrapped from Mithril), cardano-db-sync 13.7.1.0 and PostgreSQL 17 with mandatory tuning ([docs.midnight.network/nodes/cardano-db-sync](https://docs.midnight.network/nodes/cardano-db-sync)), and then the Midnight node itself.

### What that stack costs, per official sources

| item | source | figure |
|---|---|---|
| cardano-node, minimum | [cardano-node 11.0.1 release notes](https://github.com/IntersectMBO/cardano-node/releases/tag/11.0.1), "Minimum System Requirements" | 2+ cores at 1.6 GHz (2 GHz for a pool or relay); 24 GB RAM with the `InMemory` UTxO backend, 8 GB with `OnDisk` ("pending confirmation"); 300 GB free storage (350 GB recommended) |
| Cardano side, as sized by Midnight | [docs.midnight.network/nodes/cardano-node](https://docs.midnight.network/nodes/cardano-node) | mainnet: 32 GB RAM or more, 4 cores, 60,000 IOPS or better, 320 GB NVMe SSD, 100 Mbps. preview/preprod: 16 GB, 4 cores, 30,000 IOPS, 40 GB NVMe |
| cardano-node + db-sync co-located, requirement | [cardano-db-sync README, System Requirements (May 2025)](https://github.com/IntersectMBO/cardano-db-sync#system-requirements) | 64 GB RAM or more (mainnet), 4 cores or more, 60k IOPS or better, 700 GB or more of disk (SSD) |
| cardano-node + db-sync, measured on mainnet (`mainnet_13.6.0.5`) | same | cardano-node DB 203 GB; db-sync ledger state files ~10 GB; Postgres DB 438 GB; cardano-node RAM 24 GB; db-sync RSS 21 GB |
| same, preprod | same | Postgres DB 16 GB; cardano-node DB 12 GB; RSS 5.5 GB (node) + 3.5 GB (db-sync) |
| same, preview | same | Postgres DB 21 GB; cardano-node DB 12 GB; RSS 2.8 GB (node) + 3.5 GB (db-sync) |
| PostgreSQL tuning | [docs.midnight.network/nodes/cardano-db-sync](https://docs.midnight.network/nodes/cardano-db-sync) | "PostgreSQL tuning is required when targeting Mainnet. Without it, Cardano-db-sync synchronization will take an extremely long time to complete." `shared_buffers = 16GB`, `effective_cache_size = 48GB`, `maintenance_work_mem = 4GB`, `max_parallel_maintenance_workers = 4` |
| db-sync UTxO backend | [db-sync `doc/configuration.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/configuration.md) | `inmemory` "~16GB on mainnet at tip"; `lsm` "~2-3GB" at the cost of more disk I/O |
| sync time | Partner Chains [`docs/intro.md`](https://github.com/input-output-hk/partner-chains/blob/e514eb79/docs/intro.md) | db-sync: preview "hours", preprod "up to a day", mainnet "~2 days". Midnight's docs give no db-sync figure; Mithril brings cardano-node itself to "roughly 20 minutes" |
| locality | cardano-db-sync README | "The recommended configuration is to have the db-sync and the PostgreSQL server on the same machine. During syncing ... there is a HUGE amount of data traffic between db-sync and the database." |

Two figures disagree: Midnight's page sizes the Cardano side at 32 GB, db-sync's README at 64 GB for the co-located pair, and Midnight's table does not say whether it includes db-sync. Midnight's own tuning values (`effective_cache_size = 48GB`) assume the larger host. An operator sizing from Midnight's page alone under-provisions relative to the indexer's guidance.

### Why db-sync's own footprint controls do not apply here

db-sync describes itself as general-purpose: "The initial design of db-sync was a one size fits all approach. It served as a general-purpose backend for Cardano applications, including light wallets, explorers, etc. ... Most application use only a small fraction of these features. Therefore, db-sync offers options that turn off some of these features, especially the most expensive ones in terms of performance." ([`doc/configuration.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/configuration.md)). The largest saving is `ledger: disable` ("significantly reduce memory usage (by up to 10GB on mainnet) and sync time"), followed by pruning consumed outputs (`tx_out: prune` or `bootstrap`) and disabling the multi-asset, plutus and metadata tables.

The Partner Chains db-sync data sources require the opposite: they refuse to start unless db-sync runs its heaviest documented profile — ledger state enabled, full transaction output history, and the multi-asset, Plutus and governance tables populated (Appendix A lists the exact configuration flags). The queries show why: the epoch nonce and the stake distribution come from ledger-derived tables that are left empty when the ledger is disabled; deciding what was unspent at a given block height joins spent inputs against the full output history; registrations and governance bodies are located through the multi-asset tables and decoded from datums.

A Midnight operator must therefore run db-sync in its heaviest documented profile: ledger replay on, full output history, multi-asset and plutus tables populated. The general-purpose indexer's cost is paid in full by a consumer that uses a small slice of it. That is a mismatch between tool and job, not a defect of the tool.

### Operational burden beyond hardware

The coupling to a large relational archive shows up as ongoing work, not only as a hardware line:

- **The node binary modifies the operator's database.** The follower creates an index at startup ("This may take a while") and warns when it cannot ("is your db-sync readonly? ... Performance may be degraded"). The genesis-generation subcommands of the same binary create nine more indexes and retune autovacuum on six tables to keep the Postgres planner off weeks-stale statistics (~430-second queries were observed otherwise), and the data-source crate documents two further indexes the operator's database must carry. Appendix A itemizes all of these.
- **Query performance is a standing engineering stream.** Release [node-1.0.1](https://github.com/midnightntwrk/midnight-node/releases/tag/node-1.0.1) includes "Speed up cNight db-sync observation queries" (#1365) and "Cache multi_asset.id to avoid excessive joins" (#934). The node runs a startup probe that warns when db-sync tip or block lookups exceed 500 ms.
- **Indexer lag is a consensus problem.** Partner Chains docs: "Attempting to run a partner chain node with a db-sync instance that lags behind will result in consensus errors." `BLOCK_STABILITY_MARGIN` exists to be raised "if the network experiences a high number of blocks rejected because of Db-Sync lag".
- **Version coupling.** Midnight's docs pin db-sync 13.7.1.0 against cardano-node 11.0.1 and refer operators to a per-release compatibility matrix. A db-sync schema change means a migration or a resync ([`doc/migrations.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/migrations.md), [`doc/schema-management.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/schema-management.md)), during which the Midnight node cannot follow Cardano.
- **A second stateful service.** A pool of 32 database connections per node, TLS mandatory, and a database with its own backup, upgrade and access-control story.

Each item is reasonable engineering for a db-sync-backed follower. Together they mean that operating a Midnight node requires PostgreSQL administration, a skill a Cardano stake pool operator does not otherwise need.

### Existing observers named by official sources

| observer | named in | role | status for Midnight |
|---|---|---|---|
| cardano-db-sync + PostgreSQL | Midnight docs, midnight-node README and code, Partner Chains docs | the node's data source for all Cardano observation | the only supported production path |
| cardano-node (with Mithril bootstrap) | Midnight docs, Partner Chains docs | the validating node db-sync follows over a Unix socket; "mandatory for running a partner chain" | required, but the follower does not read it directly; its own node-to-client interface is unused |
| `partner-chains-mock-data-sources` | midnight-node (`use_main_chain_follower_mock`), Partner Chains | fixed registrations from a JSON file, no Cardano at all | tests and local development only |
| Ogmios | Partner Chains docs (dependencies and SPO registration guides) | WebSocket JSON-RPC bridge to a local cardano-node, used by the offchain code to submit registration and smart-contract transactions; "only required ... when registering as SPO" | a transaction submitter, not an observer; not used by the follower |
| `partner-chains-dolos-data-sources` | Partner Chains repository | implementations of the toolkit's data sources over a Blockfrost-compatible HTTP API served by the Dolos data node | marked WIP ("Dolos data sources are still WIP and should not be used in production"); a parity-diff harness ([PR #1128](https://github.com/input-output-hk/partner-chains/pull/1128)) reports `stake_delegation` differences and is still open; not integrated by midnight-node (Appendix A) |

The toolkit's design anticipates alternatives: "For the sake of modularity and indexer-independence, each feature separately defines its data needs in the form of a Data Source API. This APIs serve as contracts for various concrete Data Source Implementations" ([`docs/intro.md`](https://github.com/input-output-hk/partner-chains/blob/e514eb79/docs/intro.md)). What is missing at the Midnight level is a statement of the contract an observer must meet, a way to show conformance, and node support for selecting one.

### Why now

[MPS-0030](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0030-consensus-decentralization.md) UC4 asks that running a Midnight validator "fits within the operating envelope of an existing Cardano stake pool" and lists validator resource requirements as an open question. The Cardano archive is the largest item in that envelope and the one an SPO does not already have. [MPS-0017](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/168) (open question 2) asks whether the mainchain follower should be extended to observe governed-contract events. MPS-0019, MPS-0033, MPS-0034 and the committee-bridge MIP each add Cardano-side observation. Every new consumer inherits the archive requirement unless the dependency is right-sized first. [MPS-0032](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0032-storage-management.md) makes the same distinction for Midnight's own history ("data needed for consensus and data needed for historical queries"); this MPS applies it to the Cardano side.

## Use Cases

**UC1: Cardano SPO adding a Midnight validator**

* **Scenario:** A stake pool operator already runs a cardano-node relay and block producer and wants to run a Midnight validator on the same operation (MPS-0030 UC4).
* **Limitations:** They must add cardano-db-sync and a tuned PostgreSQL 17, sized by db-sync's guidance at 64 GB RAM and 700 GB disk with 60k IOPS, wait about two days for a mainnet sync, and take on database administration. None of this exists in their current stack; their relay already validates and holds the chain the follower needs.
* **Desired Outcome:** The Cardano observation the validator needs runs on hardware of the class the SPO already operates, sourced from the node they already run, without a database tier.

**UC2: Non-validating node operator**

* **Scenario:** A team runs a Midnight RPC node, boot node or full node for an application, an indexer or a wallet backend. Their node produces no blocks.
* **Limitations:** The documented prerequisites are the same full archive stack as for a validator. The Cardano dependency dominates the cost of a node whose consensus role is nil.
* **Desired Outcome:** The Cardano dependency of a non-validating node is bounded by what that node reads, and the docs say what that is.

**UC3: Testnets, CI and local development**

* **Scenario:** A developer or a CI pipeline brings up a Midnight node against preview or preprod.
* **Limitations:** Each environment replicates cardano-node, db-sync, PostgreSQL and (for registration) Ogmios; db-sync alone is 16 to 21 GB of Postgres plus 3.5 GB RSS on a testnet and takes hours to a day to sync. The Partner Chains local environments ship all four as containers.
* **Desired Outcome:** A testnet-connected node, including its Cardano observation, comes up in minutes on a laptop-class machine.

**UC4: db-sync upgrade or failure on a producing node**

* **Scenario:** A validator's db-sync falls behind, hits a schema migration on a pinned version bump, or needs a resync after a disk failure.
* **Limitations:** A lagging indexer produces consensus errors (Partner Chains docs); a mainnet resync is about two days, during which the node cannot follow Cardano. The operator's mitigation is a second archive, doubling the cost.
* **Desired Outcome:** Recovery of the Cardano observation is measured in minutes, and redundancy costs a fraction of a node, not a multiple.

**UC5: A new Cardano-side observation feature**

* **Scenario:** The follower is extended to observe governed-contract events (MPS-0017), native staking (MPS-0034) or the committee bridge (MPS-0033).
* **Limitations:** Each feature is written against db-sync's schema, which fixes the observer for every node and grows the set of tables an operator must maintain.
* **Desired Outcome:** New observation features are specified against the data-source contract, so they are implementable on any conforming observer and do not by themselves raise the operating envelope.

## Goals

### Primary Goal

1. **Proportional cost.** The resources a Midnight node spends on Cardano observation (memory, disk, IOPS, sync time, operator skills) should be proportional to the data the node reads, not to the size of a general-purpose archive.

### Secondary Goals

1. **An explicit observation contract.** The protocol documentation states what the node reads from Cardano (addresses, policy ids, block and epoch data), how fresh it must be, and what trust it assumes, independently of any product.
2. **Observer choice at the node boundary.** The interface boundary that already fronts all six of the node's Cardano data sources admits production implementations other than db-sync, selected by configuration rather than by the mock flag alone.
3. **A measured baseline.** Official documentation reports the measured cost of the Cardano dependency on the recommended hardware (steady-state RSS, disk, IOPS, sync time) for mainnet, preprod and preview, so that alternatives can be compared and the two hardware tables reconciled.
4. **No forced archive.** Operators who need a Cardano archive for other reasons keep running one; operators who do not are not required to.

## Requirements

Any solution that changes how a Midnight node observes Cardano must meet the following.

* **Freshness.** Delivers Cardano data at least as fresh as the protocol consumes it today: finalized data offset by two mainchain epochs for committee selection (MIP-0010), and stable-block selection governed by the security parameter `k`, `f` and `BLOCK_STABILITY_MARGIN` for the block hash inherent.
* **Determinism.** Every node observing the same Cardano chain obtains identical results for the same query at the same block, including across rollbacks within `k`. Observation is a consensus input.
* **Parity.** Results are identical to the current follower's on the full read surface documented in Appendix A, demonstrated on preview, preprod and mainnet fixtures, before the observer is accepted for validators.
* **Stated trust.** The observer documents its trust assumption relative to a fully validating cardano-node, and can be deployed behind the operator's own cardano-node so that an SPO's existing relay is the root of trust.
* **Bounded, documented resources.** The observer's own documentation states its memory, disk, IOPS and bootstrap time for each network, and the Midnight node does not need to tune or index the observer's storage at runtime.
* **No new hardware class.** A validator's Cardano observation fits on hardware of the class a Cardano stake pool already runs.

## Expected Outcomes

* The Cardano observation dependency stops being the largest item in a Midnight node's operating envelope, and MPS-0030 UC4 becomes achievable for existing SPOs.
* Node roles that read little Cardano data pay little for it; the candidate pool and the RPC/boot infrastructure both grow.
* Testnet, CI and local development environments start in minutes instead of hours to days.
* Cardano observation is specified as a contract, so features that add observation (MPS-0017, MPS-0033, MPS-0034) are written once against the contract and do not re-impose an archive.
* Operators retain db-sync where its breadth is wanted; it is no longer the only option where it is not.

## Open Questions

* **What does the dependency cost today, measured?** Midnight's docs give no db-sync sync time or steady-state figures on the recommended hardware; the two hardware tables (32 GB vs 64 GB) are unreconciled. A measured baseline is needed before any goal here is testable.
* **Where does the observation contract live?** As a protocol document in this repository, as the data-source trait documentation in midnight-node, or both?
* **What conformance evidence should be required** before an observer other than db-sync is accepted for validators, and who runs it? The Partner Chains diff harness (PR #1128) is a starting point.
* **Should the two Midnight-native sources (cNIGHT observation, federated authority) move behind the Partner Chains data-source boundary**, or does Midnight define its own boundary that covers all six?
* **Is a shared observer acceptable for validators**, or must each validator observe Cardano independently? A shared database is possible today (`POSTGRES_HOST`) but concentrates trust and availability.
* **How does the read surface grow** with MPS-0017, MPS-0033 and MPS-0034, and does that growth change the minimal footprint?
* **What is the trust difference, concretely,** between reading Cardano through a validating node's local socket and reading it from a non-validating observer, for each consumer of observation data?

## Recommended MIPs

1. **Cardano observation contract.** A specification of what the Midnight node reads from Cardano (addresses, policy ids, block and epoch data, query shapes), the freshness and determinism requirements per consumer, and the trust assumption. It turns the read surface documented in this MPS into a normative document that observers are written against. Prerequisite for the other two.
2. **Pluggable main-chain data source in midnight-node.** Make the interface boundary that already fronts all six data sources, including cNIGHT and federated-authority observation, selectable by configuration (the Partner Chains toolkit already selects among `db-sync`, `mock` and others by `CARDANO_DATA_SOURCE`), with db-sync remaining the default. Removes the hard Postgres dependency from the node binary without changing what is observed.
3. **Data-source conformance suite.** Fixtures and a diff harness that run any two observers side by side on preview, preprod and mainnet and report divergence on the contract's read surface, including rollback cases. Defines the acceptance bar an observer must pass before validators use it, and gives the Foundation a measured baseline for the cost figures this MPS asks for.

## References

* midnight-node, `main` @ [`4d5eb6dd`](https://github.com/midnightntwrk/midnight-node/tree/4d5eb6dd) (2026-09-01) — all code references are pinned to this commit in Appendix A. [Release node-1.0.1](https://github.com/midnightntwrk/midnight-node/releases/tag/node-1.0.1).
* Midnight docs (read 2026-09-01): [Nodes](https://docs.midnight.network/nodes), [Set up Cardano node](https://docs.midnight.network/nodes/cardano-node), [Set up Cardano-db-sync](https://docs.midnight.network/nodes/cardano-db-sync), [Set up full node](https://docs.midnight.network/nodes/full-node), [Set up boot node](https://docs.midnight.network/nodes/boot-node), [Set up RPC node](https://docs.midnight.network/nodes/rpc-node).
* IntersectMBO/cardano-db-sync (latest release 13.7.2.1, 2026-06-17): [README, System Requirements](https://github.com/IntersectMBO/cardano-db-sync#system-requirements), [`doc/configuration.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/configuration.md), [`doc/schema.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/schema.md).
* IntersectMBO/cardano-node: [11.0.1 release notes, Minimum System Requirements](https://github.com/IntersectMBO/cardano-node/releases/tag/11.0.1).
* input-output-hk/partner-chains, `master` @ [`e514eb79`](https://github.com/input-output-hk/partner-chains/tree/e514eb79) (read 2026-09-01): [`docs/intro.md`](https://github.com/input-output-hk/partner-chains/blob/e514eb79/docs/intro.md), [`docs/developer-guides/dependencies.md`](https://github.com/input-output-hk/partner-chains/blob/e514eb79/docs/developer-guides/dependencies.md), [`toolkit/data-sources/dolos/`](https://github.com/input-output-hk/partner-chains/tree/e514eb79/toolkit/data-sources/dolos), [PR #1128](https://github.com/input-output-hk/partner-chains/pull/1128).
* Related proposals: [MPS-0030](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0030-consensus-decentralization.md) (UC4, open question on validator resources), [MPS-0032](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0032-storage-management.md) (consensus data vs. query data), [MPS-0017, PR #168](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/168) (open question 2), [MIP-0010](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-0010-aura-to-babe-migration.md) (two-epoch finalized offset).

## Appendix A: Implementation Snapshot

This appendix records the implementation state backing the claims in the Problem section, as evidence rather than as a proposed design. All midnight-node references are pinned to `main` @ [`4d5eb6dd`](https://github.com/midnightntwrk/midnight-node/tree/4d5eb6dd) (2026-09-01) and all partner-chains references to `master` @ [`e514eb79`](https://github.com/input-output-hk/partner-chains/tree/e514eb79) (read 2026-09-01). Later drift in either repository does not alter what was observed at these commits.

### A.1 The read surface

The complete read surface of the running node is the `DataSources` struct in [`node/src/main_chain_follower.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/node/src/main_chain_follower.rs):

| data source | trait and methods | what it reads from Cardano |
|---|---|---|
| `mc_hash` | `McHashDataSource::get_latest_stable_block_for(timestamp)`, `get_stable_block_for(hash, timestamp)`, `get_block_by_hash`, `is_cardano_tip_fresh`, `is_cardano_ok` | the latest stable block inside a time window derived from `k` and `f`; a stable block by hash and timestamp; a block by hash; two health checks that ask the observer whether its view of Cardano is fresh |
| `sidechain_rpc` | `SidechainRpcDataSource::get_latest_block_info` | the latest Cardano block |
| `authority_selection` | `AuthoritySelectionDataSource::get_ariadne_parameters(epoch, policies)`, `get_candidates(epoch, address)`, `get_epoch_nonce(epoch)`, `data_epoch(epoch)` (epoch arithmetic, no data) | the D-parameter and permissioned-candidate UTxOs (by policy id), registration UTxOs at the committee-candidate address with their datums, per-pool stake for the epoch, the epoch nonce |
| `cnight_observation` | `MidnightCNightObservationDataSource::get_utxos_up_to_capacity(config, position, tip, capacity)` | cNIGHT registration and deregistration UTxOs at the registrants address, and asset create/spend UTxOs under the cNIGHT policy, in block order from a checkpoint |
| `federated_authority_observation` | `FederatedAuthorityObservationDataSource::get_federated_authority_data(config, block_hash)` | the council and technical-authority membership UTxOs, one per body, located by policy id and decoded from the datum |
| `bridge` | `TokenBridgeDataSource::get_transfers(scripts, checkpoint, max, block)` | transfer UTxOs at the bridge address after a checkpoint, with a lookahead |

Trait sources: [`sidechain-mc-hash/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/partner-chains/toolkit/sidechain/sidechain-mc-hash/src/lib.rs), [`sidechain/rpc/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/partner-chains/toolkit/sidechain/rpc/src/lib.rs), [`authority-selection-inherents/src/authority_selection_inputs.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/partner-chains/toolkit/committee-selection/authority-selection-inherents/src/authority_selection_inputs.rs), [`bridge/primitives/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/partner-chains/toolkit/bridge/primitives/src/lib.rs), [`primitives/mainchain-follower/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/primitives/mainchain-follower/src/lib.rs).

### A.2 db-sync tables touched

The queries behind these sources touch eleven tables: `block`, `tx`, `tx_in`, `tx_out`, `ma_tx_out`, `multi_asset`, `datum`, `tx_metadata`, `epoch_param`, `epoch_stake`, `pool_hash` ([`data-sources/db-sync/src/db_model.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/partner-chains/toolkit/data-sources/db-sync/src/db_model.rs), [`primitives/mainchain-follower/src/db/queries/`](https://github.com/midnightntwrk/midnight-node/tree/4d5eb6dd/primitives/mainchain-follower/src/db/queries)). db-sync's schema documents about eighty ([`doc/schema.md`](https://github.com/IntersectMBO/cardano-db-sync/blob/master/doc/schema.md)).

### A.3 Source selection in the node binary

`create_cached_data_sources` fails with `Missing db_sync_postgres_connection_string` unless `use_main_chain_follower_mock` is set. [`node/Cargo.toml`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/node/Cargo.toml) depends on `partner-chains-db-sync-data-sources` and `partner-chains-mock-data-sources` only. All six sources, including the two Midnight-native ones, are trait objects in the `DataSources` struct — but every trait has exactly two implementations, a db-sync/Postgres one and a mock, and the only selection mechanism is the `use_main_chain_follower_mock` boolean. The node has no counterpart to the toolkit's `CARDANO_DATA_SOURCE` selector.

### A.4 Required db-sync configuration

The Partner Chains db-sync data sources validate db-sync's `insert_options` at startup ([`data-sources/db-sync/src/lib.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/partner-chains/toolkit/data-sources/db-sync/src/lib.rs)): `insert_options.ledger` must be `"enable"`, `tx_out.value` must be `"enable"` or `"consumed"`, `multi_asset` must be `true`, `plutus` `"enable"`, `governance` `"enable"`, `use_address_table` `false`. The queries show why: `get_epoch_nonce` reads `epoch_param` and `get_stake_distribution` reads `epoch_stake`, both left empty when the ledger is disabled; the UTxO-at-address query joins `tx_in` against `tx_out` history to decide what was unspent at a given block height; registrations and governance bodies are located through `multi_asset` and `ma_tx_out` and decoded from `datum`.

### A.5 Modifications to the operator's database

- The follower creates `idx_multi_asset_policy_name_hex` at startup ("This may take a while") and warns when it cannot ("is your db-sync readonly? ... Performance may be degraded").
- The genesis-generation subcommands of the same binary create nine more indexes across `tx_out`, `tx_in`, `ma_tx_out`, `multi_asset`, `block` and `tx` — described in code as "critical for genesis generation performance when scanning the full Cardano blockchain", one of them avoiding "a full scan over ~1 billion rows" on `ma_tx_out` — and alter autovacuum settings on six tables because the Postgres planner "runs on weeks-stale statistics and picks bad join orders for high-cardinality lookups (observed ~430s queries on a preview/preprod cnight observation lookup against an otherwise idle DB)" ([`db/queries/cnight_observation.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/primitives/mainchain-follower/src/db/queries/cnight_observation.rs)).
- The db-sync data-source crate's documentation lists two further indexes the operator's database must carry (`idx_ma_tx_out_ident`, `idx_tx_out_address`).
- The node opens six connection pools totaling 32 connections against the db-sync database, requires TLS on all of them, and runs a startup probe that warns when tip or block lookups exceed 500 ms ([`node/src/main_chain_follower.rs`](https://github.com/midnightntwrk/midnight-node/blob/4d5eb6dd/node/src/main_chain_follower.rs)).

### A.6 Status of the Dolos data sources

[`toolkit/data-sources/dolos`](https://github.com/input-output-hk/partner-chains/tree/e514eb79/toolkit/data-sources/dolos) in the partner-chains repository implements the toolkit's data sources over a Blockfrost-compatible HTTP API served by the Dolos data node, selectable as `CARDANO_DATA_SOURCE=dolos` in the toolkit's demo node alongside `db-sync` and `mock`. It is marked WIP ("Dolos data sources are still WIP and should not be used in production"); its last change was 2025-12-11; a parity-diff harness ([PR #1128](https://github.com/input-output-hk/partner-chains/pull/1128)) reports `stake_delegation` differences and is still open. midnight-node's vendored copy of the toolkit contains only the `cli`, `db-sync` and `mock` data-source crates ([`partner-chains/toolkit/data-sources/`](https://github.com/midnightntwrk/midnight-node/tree/4d5eb6dd/partner-chains/toolkit/data-sources)), and the two Midnight-native sources have no implementation outside db-sync and the mock.

## Acknowledgements

Contributors and reviewers to be listed.

## Copyright

This MPS is licensed under CC-BY-4.0.
