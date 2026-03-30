---
MIP: 0002
Title: Acropolis Node - Custom Cardano Indexer Replacement
Authors: Chris Tilt (christopher-tilt)
Status: Proposed
Category: Core
Created: 2026-03-30
Requires: none
Replaces: none
License: Apache-2.0
---

<!--
 Copyright 2025 Midnight Foundation

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

This proposal introduces the Acropolis node, a modular Rust implementation of the Cardano node developed by IOG, as a replacement for the current Cardano Node and DBSync (db-sync) components used by Midnight for mainchain observation. The Acropolis node implements a custom Cardano blockchain indexer optimized specifically for Midnight's requirements, exposing indexed data through a gRPC interface. This targeted approach delivers substantial performance improvements: **48x faster boot time** (1-2 hours vs 6-24 hours to chain tip on mainnet, 12-30 minutes with Mithril snapshots), **3x faster Midnight sync time**, and **90+% reduced memory requirements** (7 GB vs 124+ GB represents a 94% reduction in indexer memory footprint, with additional optimizations easily achievable such as 50% further reduction). The custom indexer simplifies the Midnight node infrastructure by indexing only the data necessary for Midnight operations: cNIGHT token registrations, governance updates, UTXO state, and Ariadne validator selection data.

## Motivation

The current Midnight architecture relies on Cardano Node and DBSync to observe mainchain state, specifically to track cNIGHT token registrations and governance updates. This approach presents several challenges and accounts for **70% of Midnight's operational costs**, with the Midnight node itself comprising only 30%. DBSync fulfills its role as a general-purpose indexer, but it is fundamentally overkill for Midnight's specific requirements.

### Resource Inefficiency

DBSync indexes the entire Cardano blockchain into a PostgreSQL database (cexplorer), including vast amounts of data irrelevant to Midnight operations. Benchmarking demonstrates the severe resource penalty:

**Memory Requirements:**
- Cardano Node + DBSync: 124+ GB
- Acropolis custom indexer: 7 GB
- **Reduction: 94% (17.7x less memory)**

**Boot Time (Time to Chain Tip):**
- Preview testnet: 6 hours (DBSync) vs 1 hour (Acropolis genesis sync)
- Mainnet: 24 hours (DBSync) vs 2 hours (Acropolis genesis sync)
- **With Mithril snapshots:** 12 minutes (preview) / 30 minutes (mainnet)
- **Improvement: 48x faster**

**Query Performance:**
- Sequential operations: 11x faster with gRPC vs DBSync SQL queries (2.94s → 0.25s total per-block calls)
- Parallelized operations: 65x faster
- Midnight node sync time: 3x faster end-to-end
- Example: `cnight_observation_get_utxos_up_to_capacity` reduced from 2.8351 seconds to 24.034 milliseconds (118x improvement)
- Other performance improvements have already been done, but not fully quantified (examples: get_latest_stable_block_for reduced -96.014%, get_stable_block_for reduced -45.618%, and get_block_by_hash reduced by -36.276%).


### Operational Complexity

The current architecture requires running and maintaining multiple components:
- Full Cardano Node for blockchain synchronization
- DBSync service for indexing
- PostgreSQL database with SSL/TLS configuration
- Complex inter-component communication and monitoring

Acropolis simplifies this to **a single binary** with:
- Easy installation and configuration
- Integrated indexing and chain synchronization
- No external database dependencies
- Built-in Mithril support for rapid synchronization

### Tight Coupling
The dependency on DBSync creates coupling to:
- Cardano Node update cycles
- DBSync maintenance and compatibility
- PostgreSQL version requirements
- Generic indexing schemas not designed for Midnight

### Specific Midnight Requirements

Midnight requires only a subset of Cardano blockchain data:
- cNIGHT token registration events and associated UTXOs
- Governance transactions affecting Council and Technical Committee memberships
- Cardano Midnight System Transactions (CMST)
- DUST generation tracking
- Ariadne validator selection data (epoch nonces, candidates, parameters)

### Why Acropolis?

Acropolis is a modular and extensible Rust implementation of the Cardano node developed by Input Output Global (IOG). Its architecture is specifically designed to support:
- Peer-to-peer networking, ledger validation, consensus, and chain synchronization
- **Configurable and custom indexers built directly into the node**
- Multiple use cases through its extensible architecture (e.g., Blockfrost API, DEX batchers)

The Acropolis team has already:
1. **Implemented a custom Midnight indexer** with all required functionality
2. **Developed the client-side gRPC logic** on a fork of midnight-node
3. **Demonstrated the performance improvements** with concrete benchmarks
4. **Committed to supporting integration and maintenance** with new funding

Reference implementation: https://github.com/whankinsiv/midnight-node-acropolis

The Acropolis approach addresses all the challenges above by:
- Indexing only data relevant to Midnight operations
- Exposing a modern gRPC interface optimized for Midnight's query patterns
- Drastically reducing resource requirements (94% memory reduction, 48x faster boot)
- Simplifying deployment to a single binary
- Enabling independent evolution of the Midnight-Cardano integration layer
- Leveraging IOG's continued development and support of the Acropolis platform

### Additional work needed
The Acropolis project has been funded by Intersect and the project team has met
all milestones to date and will meet the March '26 milestones that include 5.7
and 5.8: Plutus Script eval phase 2 and peer-to-peer networking and chain following.
However, the project has not undergone extensive testing on preview and is not yet
mainnet ready. This MIP must include additional work:
1. **Productization of the fundamental system**
2. **Extensive testing on Preview**
3. **Deployment on mainnet as a follower**
4. **Hardening of the peer-to-peer networking and chain following protocol**
These are just being completed and need testing and additional hardening.
5. **Phased delivery for Midnight verifies followed by Midnight block producers**
6. **Investigate if using a file-based db (library from the binary) can further reduce memory usage**
7. **Additional setup and maintenance documentation specically for MidNight deployment**
8. **Additional Monitoring and metrics** Product grade monitoring

## Specification

### Architecture Overview

The Acropolis node is a standalone service that:
1. Connects directly to Cardano network nodes via Ouroboros network protocol
2. Maintains a custom indexed view of the Cardano blockchain containing only Midnight-relevant data
3. Exposes indexed data through a gRPC API
4. Provides efficient query capabilities for Midnight node pallets

### Component Structure

```
┌─────────────────────────────────────────────────────────────┐
│                      Midnight Node                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Native Token Observation Pallet                      │  │
│  │  - cNIGHT token tracking                              │  │
│  │  - DUST generation management                         │  │
│  │  - UTXO state tracking                                │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │ gRPC                                │
│  ┌────────────────────▼─────────────────────────────────┐  │
│  │  Federated Authority Observation Pallet               │  │
│  │  - Authority change monitoring                        │  │
│  │  - Council membership updates                         │  │
│  │  - Technical Committee updates                        │  │
│  └────────────────────┬─────────────────────────────────┘  │
└────────────────────────┼─────────────────────────────────────┘
                         │ gRPC
                         │
           ┌─────────────▼──────────────┐
           │    Acropolis Node          │
           │  ┌──────────────────────┐  │
           │  │   gRPC API Server    │  │
           │  └──────────┬───────────┘  │
           │  ┌──────────▼───────────┐  │
           │  │  Custom Indexer      │  │
           │  │  - Token events      │  │
           │  │  - Governance txs    │  │
           │  │  - CMST tracking     │  │
           │  │  - UTXO management   │  │
           │  └──────────┬───────────┘  │
           │  ┌──────────▼───────────┐  │
           │  │  Optimized Storage   │  │
           │  │  (Embedded DB)       │  │
           │  └──────────┬───────────┘  │
           │  ┌──────────▼───────────┐  │
           │  │  Chain Sync Client   │  │
           │  └──────────┬───────────┘  │
           └─────────────┼────────────────┘
                         │ Ouroboros Protocol
                         │
                ┌────────▼─────────┐
                │  Cardano Network │
                │  (Mainnet/Testnet)│
                └──────────────────┘
```

### Indexing Scope

The Acropolis node will index and store:

#### Token Operations
- cNIGHT token minting transactions
- cNIGHT token registration events
- UTXO outputs containing cNIGHT tokens
- UTXO spending that affects cNIGHT balances
- Token metadata relevant to Midnight operations

#### Governance Transactions
- Transactions updating Council membership
- Transactions updating Technical Committee membership
- Governance parameter changes affecting Midnight
- Authority delegation events

#### Cardano Midnight System Transactions (CMST)
- All CMST transaction types
- Associated metadata and parameters
- State transitions relevant to Midnight

### gRPC API Specification

The Acropolis node exposes the following gRPC services:

#### TokenObservationService
```protobuf
service TokenObservationService {
  // Query cNIGHT token registrations
  rpc GetTokenRegistrations(TokenRegistrationQuery) returns (stream TokenRegistration);

  // Query UTXOs containing cNIGHT tokens
  rpc GetTokenUTXOs(UTXOQuery) returns (stream TokenUTXO);

  // Monitor new token events
  rpc SubscribeTokenEvents(TokenEventSubscription) returns (stream TokenEvent);

  // Get DUST generation data
  rpc GetDustGeneration(DustQuery) returns (DustGenerationInfo);
}
```

#### GovernanceObservationService
```protobuf
service GovernanceObservationService {
  // Query current Council membership
  rpc GetCouncilMembers(MembershipQuery) returns (MembershipInfo);

  // Query current Technical Committee membership
  rpc GetTechnicalCommitteeMembers(MembershipQuery) returns (MembershipInfo);

  // Monitor governance changes
  rpc SubscribeGovernanceEvents(GovernanceSubscription) returns (stream GovernanceEvent);

  // Get authority delegation history
  rpc GetAuthorityHistory(AuthorityQuery) returns (stream AuthorityChange);

  // Get federated authority data
  rpc GetFederatedAuthorityData(AuthorityDataQuery) returns (FederatedAuthorityData);
}
```

#### AriadneValidatorSelectionService
```protobuf
service AriadneValidatorSelectionService {
  // Get epoch data for validator selection
  rpc GetEpochData(EpochQuery) returns (EpochData);

  // Get Ariadne parameters for an epoch
  rpc GetAriadneParameters(EpochQuery) returns (AriadneParameters);

  // Get validator candidates for an epoch
  rpc GetValidatorCandidates(EpochQuery) returns (stream ValidatorCandidate);

  // Get epoch nonce for randomness
  rpc GetEpochNonce(EpochQuery) returns (EpochNonce);
}
```

#### ChainSyncService
```protobuf
service ChainSyncService {
  // Get current chain tip
  rpc GetChainTip(ChainTipQuery) returns (ChainTip);

  // Query block by hash or height
  rpc GetBlock(BlockQuery) returns (Block);

  // Monitor new blocks
  rpc SubscribeBlocks(BlockSubscription) returns (stream Block);

  // Get sync status
  rpc GetSyncStatus(SyncStatusQuery) returns (SyncStatus);
}
```

### Storage Layer

The Acropolis node uses an embedded database (e.g., Fjall) for chain storage; it could
be extended to additionally store:
- Fast UTXO lookups by address and token policy
- Efficient range queries on block height and transaction index
- Compact storage of indexed data
- Quick synchronization from genesis or snapshot
- Point-in-time queries for historical state

### Configuration
Configuration is covered by the documentation on the Acropolis repository.

## Rationale

### Why a Custom Indexer?

**Alternative 1: Continue using DBSync**
- Rejected because it requires indexing the entire Cardano blockchain
- Performance and resource overhead cannot be eliminated (124+ GB vs 7 GB)
- Tight coupling to DBSync update cycles
- Generic schema not optimized for Midnight queries
- Accounts for 70% of Midnight operational costs

**Alternative 2: Use existing lightweight indexers (e.g., Ogmios, Scrolls)**
- Evaluated but found to still index more data than necessary
- May not provide the exact query patterns Midnight requires
- Still requires maintaining dependency on external projects
- Limited customization for Midnight-specific optimizations
- No demonstrated performance benefits comparable to Acropolis

**Alternative 3: Acropolis Custom Indexer (Selected)**
- **Proven approach:** Already implemented with demonstrated 48x boot time improvement and 94% memory reduction
- Complete control over indexed data scope
- Optimized storage and query patterns for Midnight (11x-65x faster queries)
- Independent evolution and maintenance
- Minimal resource footprint (7 GB vs 124+ GB)
- Purpose-built gRPC API for Midnight pallets
- Built-in Mithril support for 12-30 minute sync times
- **IOG backing:** Developed and supported by Input Output Global
- Single binary deployment vs multi-component architecture

### Why gRPC?

- Type-safe interface definitions via Protocol Buffers
- Efficient binary serialization
- Built-in streaming support for real-time updates
- Strong tooling and multi-language support
- Better performance than REST/JSON for inter-service communication
- Native support for authentication and TLS

### Why Embedded Database?

- Eliminates external database dependency
- Reduced operational complexity
- Better performance through local storage
- Simplified backup and snapshot management
- Lower resource requirements

### Design Principles

1. **Minimalism**: Index only what Midnight needs
2. **Performance**: Optimize for Midnight's specific query patterns
3. **Simplicity**: Reduce architectural complexity
4. **Independence**: Enable independent evolution
5. **Reliability**: Maintain data integrity and consistency

## Path to Active

### Acceptance Criteria

1. **Proof of Concept Implementation**: Successful implementation demonstrating:
   - Connection to Cardano network via Ouroboros protocol
   - Indexing of cNIGHT token events
   - Working gRPC API with at least token observation endpoints
   - Performance benchmarks showing resource reduction vs. DBSync

2. **Feature Completeness**: Full implementation including:
   - All gRPC services specified above
   - Complete indexing of governance events
   - CMST transaction support
   - Reliable chain synchronization

3. **Testing**: Comprehensive test coverage including:
   - Unit tests for indexing logic
   - Integration tests against Cardano testnet
   - Performance benchmarks
   - Stress testing under load

4. **Integration**: Successful integration with Midnight node:
   - Modified pallets using gRPC interface
   - Equivalent functionality to current DBSync integration
   - No regressions in token tracking or governance observation

5. **Documentation**: Complete documentation including:
   - API documentation
   - Deployment guide
   - Migration guide from DBSync
   - Operations manual

### Implementation Plan

**Current Status**: The Acropolis team has already developed a proof-of-concept implementation demonstrating all core functionality with proven performance metrics. The reference implementation is available at https://github.com/whankinsiv/midnight-node-acropolis

The implementation plan focuses on productization, hardening, and deployment:

#### Phase 1: Productization & Hardening (Months 1-2)
- Review and refine existing proof-of-concept code
- Resolve remaining rewards verification errors
- Optimize memory usage through file-based storage (embedded DB)
- Pre-decode blocks for additional 10x performance improvement
- Harden peer-to-peer networking and chain following consensus
- Set up production CI/CD pipelines

#### Phase 2: Testing & Documentation (Months 3-4)
- Comprehensive integration testing on preview testnet
- Security audit of the Acropolis indexer and gRPC interface
- Load testing and stress testing under production scenarios
- Complete installation and operation documentation
- Create migration guides from DBSync
- Performance benchmarking and validation

#### Phase 3: Integration & Staging (Months 5-6)
- Finalize Midnight node pallet modifications for gRPC
- Deploy to staging environments (preview testnet)
- Parallel operation with DBSync for data validation
- Build monitoring dashboards and alerting
- Train operations teams on Acropolis deployment

#### Phase 4: Mainnet Migration (Months 7-8)
- Gradual rollout to mainnet with canary deployments
- Parallel operation period on mainnet
- Data validation and correctness verification
- Performance monitoring and optimization
- Cutover and DBSync decommissioning

**Responsible Teams**:
- Core development: Acropolis team (IOG) + Midnight engineering team
- Integration: Midnight engineering team
- Testing: QA and DevOps teams
- Documentation: Technical writing team + Acropolis team
- Deployment: Operations team

## Backwards Compatibility Assessment

### Breaking Changes

This proposal introduces a breaking change to the Midnight node architecture, however each
node can be replaced one at a time with no inter-node breakage. A Midnight node would have to
be rebooted and would take approximately 30 minutes before the chain data is available for
query. Taking nodes down 1 by 1 and upgrading them will not affect the Midnight network in
any harmful way; there is no time requirement by which all nodes must be upgraded.

1. **Pallet Interface Changes**: The Native Token Observation Pallet and Federated Authority Observation Pallet must be modified to communicate via gRPC instead of querying PostgreSQL directly

2. **Deployment Architecture**: The DBSync + PostgreSQL components are replaced with the Acropolis node

### Migration Strategy

#### For Node Operators

1. **Parallel Operation Period**: Run Acropolis alongside DBSync during transition
2. **Data Validation**: Verify Acropolis provides equivalent data to DBSync
3. **Cutover**: Switch Midnight node configuration to use Acropolis nodes on select few
4. **Decommission**: Retire DBSync and PostgreSQL after validation period
5. **Node Types**: Initial replacement can happen on validators and once further hardened can be extended to Midnight block producers

#### For Developers

1. **Pallet Updates**: Modified pallets will be released with Acropolis support
2. **Configuration Changes**: Update node configuration to point to Acropolis gRPC endpoint
3. **Testing**: Comprehensive testnet validation before mainnet deployment

#### For Applications

Applications consuming Midnight node data should not be affected as the change is internal to the node infrastructure. However:

1. **Deployment Dependencies**: Applications deploying their own Midnight nodes will need to deploy Acropolis instead of DBSync
2. **Configuration**: Update deployment configurations and documentation

### Compatibility Timeline

Given the existing proof-of-concept implementation:

- **Months 1-3**: Productization, hardening, and testing on testnet (no impact on users)
- **Months 4-5**: Preview testnet deployment and validation (selected preview testnet users must upgrade)
- **Month 6**: Parallel operation on mainnet begins (both DBSync and Acropolis supported)
- **Month 7**: Acropolis becomes primary, DBSync deprecated
- **Month 8+**: DBSync support removed, Acropolis is the only supported indexer

### Rollback Plan

If critical issues are discovered:
1. Midnight nodes can be reconfigured to use DBSync
2. Acropolis deployment can be paused or rolled back
3. Investigation and fixes applied before retry

## Security Considerations

### Threat Analysis

#### 1. Data Integrity
**Threat**: Acropolis node provides incorrect or manipulated blockchain data
**Mitigation**:
- Cryptographic verification of all Cardano blocks and transactions
- Validation against Cardano consensus rules
- Checkpointing against known block hashes
- Extensive testing against canonical Cardano data

#### 2. Denial of Service
**Threat**: Acropolis node becomes unavailable, preventing Midnight operation
**Mitigation**:
- Redundant Acropolis node deployment
- Connection pooling and failover in Midnight pallets
- Resource limits and rate limiting in gRPC API
- Monitoring and alerting for availability

#### 3. Network-Level Attacks
**Threat**: Man-in-the-middle attacks on Cardano network communication
**Mitigation**:
- TLS for gRPC communication
- Verification of Cardano network peer certificates
- Multiple Cardano node connections for cross-validation
- Network segmentation and firewall rules

#### 4. API Security
**Threat**: Unauthorized access to Acropolis gRPC API
**Mitigation**:
- Mutual TLS authentication between Midnight node and Acropolis
- API token authentication
- Authorization policies for sensitive endpoints
- Audit logging of all API requests

#### 5. Storage Security
**Threat**: Unauthorized access to indexed blockchain data
**Mitigation**:
- File system encryption for data directory
- Restricted file permissions
- Regular security updates of embedded database
- Backup encryption

#### 6. Dependency Vulnerabilities
**Threat**: Vulnerabilities in Acropolis dependencies
**Mitigation**:
- Regular dependency updates
- Automated vulnerability scanning
- Minimal dependency footprint
- Security audit of critical dependencies

### Security Properties

The Acropolis node maintains the following security properties:

1. **Verifiable Data**: All indexed data is cryptographically verifiable against Cardano blockchain
2. **Integrity**: Data cannot be modified without detection
3. **Availability**: Redundant deployment prevents single points of failure
4. **Confidentiality**: TLS encryption for all network communication
5. **Auditability**: Comprehensive logging of operations and data access

### Security Best Practices

Operators should:
- Enable TLS for all gRPC communication
- Use mutual TLS authentication where possible
- Monitor for anomalous behavior
- Keep Acropolis software updated
- Follow principle of least privilege for service accounts
- Regularly backup indexed data (?? Maybe to a local fjall DB for speedy recovery boot)
- Implement disaster recovery procedures

## Implementation

### Components to Modify

#### New Components

1. **Acropolis Node** (existing codebase)
   - Cardano chain sync client using Ouroboros protocol
   - Custom indexer for token and governance events
   - Embedded database (Fjall)
   - gRPC API server
   - Monitoring and metrics

2. **gRPC Protocol Definitions**
   - TokenObservationService.proto
   - GovernanceObservationService.proto
   - ChainSyncService.proto
   - Common message types and enums

#### Modified Components

1. **Native Token Observation Pallet**
   - Replace PostgreSQL queries with gRPC calls to Acropolis
   - Implement gRPC client and connection management
   - Update error handling for gRPC communication
   - Add retry logic and failover

2. **Federated Authority Observation Pallet**
   - Replace PostgreSQL queries with gRPC calls
   - Implement gRPC client
   - Update governance event processing

3. **Midnight Node Configuration**
   - Add Acropolis endpoint configuration
   - Remove DBSync/PostgreSQL configuration
   - Add TLS certificate configuration for gRPC

### Dependencies

#### Runtime Dependencies
- Cardano network access (existing)
- gRPC libraries (new)
- Protocol Buffer compiler (development)
- Embedded database library (RocksDB or SQLite) (new)

#### Build Dependencies
- Rust toolchain (if implementing in Rust)
- Protocol Buffer compiler (protoc)
- gRPC code generators

### Technology Stack Recommendations

**Programming Language**: Rust
- Memory safety and performance
- Strong ecosystem for blockchain development
- Existing Cardano tooling (Pallas, Oura)
- Excellent gRPC support

**Cardano Integration**: Pallas library
- Comprehensive Cardano primitives
- Ouroboros mini-protocols implementation
- Block parsing and validation

**gRPC Framework**: tonic (Rust)
- High-performance gRPC implementation
- Good Protocol Buffer integration
- Streaming support

**Embedded Database**: RocksDB
- Proven performance for blockchain indexing
- Efficient key-value storage
- Good Rust bindings

### Repository Structure

```
acropolis/
├── proto/                  # Protocol Buffer definitions
│   ├── token.proto
│   ├── governance.proto
│   └── chain.proto
├── src/
│   ├── chain_sync/        # Cardano chain synchronization
│   ├── indexer/           # Indexing logic
│   ├── storage/           # Database abstraction
│   ├── grpc/              # gRPC server implementation
│   └── config/            # Configuration handling
├── tests/
│   ├── integration/       # Integration tests
│   └── performance/       # Performance benchmarks
├── docs/
│   ├── api/              # API documentation
│   ├── deployment/       # Deployment guides
│   └── migration/        # Migration guides
└── deployment/
    ├── docker/           # Docker configurations
    ├── kubernetes/       # Kubernetes manifests
    └── systemd/          # Systemd unit files
```

## Testing

### Test Strategy

#### Unit Tests
- Indexing logic for token events
- Indexing logic for governance transactions
- Storage layer operations
- gRPC service implementations
- Configuration parsing and validation

**Coverage Target**: >80% code coverage

#### Integration Tests

1. **Chain Sync Tests**
   - Connect to Cardano testnet
   - Synchronize from genesis
   - Verify block processing
   - Test network interruption handling

2. **Indexing Tests**
   - Index known token transactions
   - Index governance updates
   - Verify indexed data correctness
   - Test edge cases and malformed data

3. **API Tests**
   - Test all gRPC endpoints
   - Verify query results
   - Test streaming/subscriptions
   - Test authentication and authorization

4. **Midnight Integration Tests**
   - Deploy Acropolis with Midnight node on testnet
   - Verify pallet functionality
   - Test token observation workflows
   - Test governance observation workflows

#### Performance Tests

1. **Synchronization Performance**
   - Measure time to sync from genesis
   - Measure sync speed (blocks/second)
   - Compare with DBSync performance

2. **Query Performance**
   - Benchmark common query patterns
   - Measure P50, P95, P99 latencies
   - Test under concurrent load

3. **Resource Usage**
   - Measure CPU usage during sync and queries
   - Measure memory footprint
   - Measure disk usage and growth rate
   - Compare with DBSync resource usage

**Performance Targets**:
- Sync speed: >100 blocks/second
- Query latency P95: <100ms
- Memory footprint: <2GB (if we use embedded DB to store encoded blocks and UTXOs)
- Storage size: <50% of DBSync equivalent (that will be easy!)

#### Security Tests

1. **Authentication Tests**
   - Test TLS certificate validation
   - Test mutual TLS authentication
   - Test token-based authentication

2. **Authorization Tests**
   - Verify access controls
   - Test unauthorized access attempts

3. **Penetration Testing**
   - Third-party security audit
   - Network security assessment
   - API security testing

#### Regression Tests

- Automated test suite run on every commit
- Comparison with DBSync data on testnet
- No functional regressions in Midnight node behavior

### Test Environments

1. **Development**: Local Cardano node (testnet)
2. **CI/CD**: Automated testing against testnet
3. **Staging**: Full deployment with Midnight node on preview testnet
4. **Production**: Gradual rollout on mainnet with monitoring

### Acceptance Testing

Before deployment to mainnet:
1. All tests pass on testnet
2. Performance targets met
3. Security audit completed and issues resolved
4. Successful parallel operation with DBSync
5. Sign-off from Midnight engineering team

## References

1. [Acropolis Proof-of-Concept Implementation](https://github.com/whankinsiv/midnight-node-acropolis) - Working implementation demonstrating Acropolis integration with Midnight
2. [Midnight at Acropolis - BuidlerFest 2026 Presentation](https://docs.google.com/presentation/d/1jdq4XBWLC_6qR4Ptp4bw8q0YEL7CrJePcAQX3yNfbwY/edit) - Performance benchmarks and architecture overview
3. [Midnight Node Architecture](https://github.com/midnightntwrk/midnight-node) - Current Midnight node implementation
4. [Cardano Node Repository](https://github.com/input-output-hk/cardano-node)
5. [DBSync Documentation](https://github.com/input-output-hk/cardano-db-sync)
6. [Ouroboros Network Protocol](https://ouroboros-network.cardano.intersectmbo.org/)
7. [Mithril Network](https://mithril.network/) - Fast Cardano snapshot synchronization
8. [gRPC Protocol Buffers](https://grpc.io/docs/what-is-grpc/introduction/)
9. [Pallas Library](https://github.com/txpipe/pallas) - Rust Cardano primitives
10. [RocksDB](https://rocksdb.org/)

## Acknowledgements

This proposal is based on the work of the Acropolis team at Input Output Global (IOG):
Will, Matt, Dmitry, Kris, Eric, Simon, Jon, Leonard, Paul, and Chris.

Thanks to all contributors and workshop participants who provided feedback on this proposal.

## Copyright Waiver

All contributions (code and text) submitted in this MIP are licensed under the Apache License, Version 2.0.
