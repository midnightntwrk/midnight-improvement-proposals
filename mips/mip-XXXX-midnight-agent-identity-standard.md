---
MIP: X
Title: Midnight Agent Identity Standard (MAIS)
Authors:
  - Zidan (mzf11125)
Status: Draft
Category: Standards
Created: 2026-05-18
Requires: none
Replaces: none
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

This MIP proposes the Midnight Agent Identity Standard, called MAIS for short. It is a framework for identity, reputation, and validation for autonomous AI agents that run on Midnight Network.

Agentic AI is spreading across DeFi, governance, and privacy applications. Midnight faces a new kind of on chain entity, the autonomous agent, that needs a way to prove who it is without breaking the privacy guarantees that define the network.

MAIS uses a three tier disclosure model for identity data. Public tier is for anyone to read. Auditor tier is for regulators and designated reviewers. Private tier is for the operator alone. This maps directly onto Midnight's existing privacy architecture.

The standard defines a Compact contract interface for agent registration, reputation building, and third party validation. Reputation claims are ZK provable, meaning an agent can prove its trust level without exposing the underlying data. The system supports two identity modes. A contract address mode for agents that want to be publicly discoverable. A ZK credential mode for agents that need operational privacy. An optional ERC-8004 bridge connects MAIS identities to the Ethereum agent identity ecosystem for agents that operate across both chains.

An existing open-source implementation by apestchanker at [midnight-agent-did-manager](https://github.com/apestchanker/midnight-agent-did-manager) already provides a working DID registry on Midnight Preprod with W3C-aligned `did:midnight` identifiers, Compact contracts, 1AM wallet integration, selective disclosure Verifiable Credentials, and an MCP server for AI agent interaction. The MAIS spec and this implementation are being aligned to ensure the standard reflects what is already proven to work on-chain.

## Motivation

### The Gap

ERC-8004 launched on Ethereum mainnet in January 2026. It provides agent identity and reputation registries on EVM chains. But those chains are transparent, so every piece of agent metadata, every reputation score, and every validation record sits in public view. That creates a real problem for agents that operate in competitive or strategic domains. A trading agent's identity and reputation are visible to every competitor who can reverse engineer its strategy from the transaction trail.

Midnight's zero knowledge architecture was designed to solve exactly this tension. Selective disclosure means an agent can prove its identity and reputation without showing its full history to the network. But Midnight does not yet have a standard interface for agents to do this. There is no MIP that defines what an agent identity means on Midnight, how reputation gets built up and proven, or how validators can vouch for agent behaviour without exposing private transaction data.

### Why This Matters Now

Three things are converging to make this standard urgent.

First, Midnight mainnet launched on 30 March 2026. The network is live and has Google Cloud running as a validator. Developer activity surged over 1,600 percent after the Summit in late 2025. The window to establish a foundational identity standard before the ecosystem fragments into incompatible schemes is open right now.

Second, the AlphaTON partnership is targeting Telegram's one billion users with privacy preserving AI agents. Those agents need an identity system that is native to Midnight. They cannot lean on EVM based ERC-8004 for identity when their entire value proposition rests on Midnight's privacy guarantees.

Third, Midnight City Simulation uses Google Gemini agents as a live public testbed. Those agents have no formal identity standard. Each simulator invents its own ad hoc scheme. A standard would make Midnight City agents composable with real world applications instead of isolated inside the simulation.

### Design Principles

MAIS is built on five principles.

Privacy first means identity and reputation claims are verifiable without revealing the private data underneath. This is not an adaptation of a transparent chain standard. It was designed from scratch for Midnight's ZK architecture.

Disclosure graded means different stakeholders see different levels. A regulator needs auditor access. A counterparty needs reputation verification. The operator needs full private access. MAIS maps identity data to Midnight's three tier model.

Composable means other Midnight standards can consume MAIS data as standard inputs. This includes x402 for payments, Bastion for security enforcement, and governance contracts.

Ecosystem bridging means MAIS should work with the broader AI agent identity world, especially ERC-8004, without giving up Midnight's privacy advantage. Cross chain identity linking stays optional, never mandatory.

Incremental adoption means the simplest mode works with one contract call. Advanced modes like ZK credentials, auditor access, and cross chain bridging stack on top without breaking the base.

## Specification

### 1. Core Concepts

MAIS defines four registries. Each one is a Compact contract.

The Identity Registry handles agent registration, metadata storage, and operator binding. It defaults to public visibility for basic mode and private visibility for advanced mode.

The Reputation Registry tracks accumulated trust scores built from historical behaviour. Scores are public. Evidence is private.

The Validation Registry manages third party attestations about agent integrity. It operates at auditor tier visibility.

The Disclosure Tier Registry maps each agent's data into Midnight's public, auditor, and private tiers. It is configurable per agent.

An agent identity can take one of two forms.

Mode A, public identity, uses the agent's Compact contract address as the identity. Registration is public. Metadata including name, capabilities, and operator address lives in public state. This suits agents that operate in open marketplaces.

Mode B, private identity, uses a ZK credential where the agent holds a commitment to a secret. The commitment stays in private state. The operator proves knowledge of the credential through a ZK proof without putting the identity on chain. This suits agents that need operational privacy.

Both modes expose the same standard interface. The mode choice is abstracted away by the SDK. Consuming contracts do not need to know which mode is underneath.

### 2. Identity Registry

#### 2.1 Public Mode Registration

An agent registers by calling `registerPublic()` on the Identity Registry contract. The call provides three things. An `operator` address for the human or organisation controlling the agent. A `metadata` struct with name, description, capabilities, and version. And an `ownershipProof` that proves the operator controls the agent address through a simple signature check.

On success the contract stores the identity in public state and emits a `RegistrationEvent` event. The agent gets a unique numeric `AgentId`. Both the Compact address and the `AgentId` are valid identifiers. Contracts can reference agents by either one.

#### 2.2 Private Mode Registration

An agent registers by calling `registerPrivate()` on the Identity Registry contract. The call provides an `operator` address, the same `metadata` struct as public mode, a `credentialCommitment` hash, and a `credentialProof` ZK proof that the operator knows the preimage of that commitment.

On success the contract stores the credential commitment in private state. It is not readable by other contracts or external watchers. No public event fires for private registrations. The agent gets a unique `AgentId` but that ID also lives in private state. External observers cannot enumerate private agents.

#### 2.3 Standard Interface

All identity operations work through the same interface regardless of mode.

```
// Query whether an agent exists, works for both modes
function exists(agentId: AgentId): bool

// Get public metadata, returns null for private agents
function getMetadata(agentId: AgentId): Metadata?

// Prove agent identity to a verifier
// Public mode returns the agent address from public state
// Private mode returns a ZK proof of credential knowledge
function proveIdentity(agentId: AgentId): IdentityProof

// Update metadata, operator only
function updateMetadata(agentId: AgentId, newMetadata: Metadata)

// Transfer operator control, needs multi sig or ZK proof
function transferOperator(agentId: AgentId, newOperator: CompactAddress, transferProof: TransferProof)

// Deactivate agent, operator only
function deactivate(agentId: AgentId)
```

#### 2.4 Metadata Schema

```
struct Metadata {
    name: string,           // Human readable agent name
    description: string,    // What the agent does
    capabilities: [Capability],  // PAYMENT, TRADING, GOVERNANCE, COMPLIANCE, DATA
    version: string,        // Semantic version
    homepage: string?,      // Optional URL
    erc8004TokenId: u256?,  // Optional cross chain link to ERC-8004 NFT
}
```

### 3. Reputation Registry

#### 3.1 Public Scores with Private Evidence

The reputation system stores public scores backed by private evidence.

The score is a number from 0 to 100 kept in public state. A tier label sits on top of it. Anyone can read an agent's current reputation tier without any authorisation. The evidence that produced the score lives as hash commitments in private state. The score itself is verifiable because it sits on chain. But the reasons behind the score stay hidden. Competitors cannot reverse engineer an agent's strategy from its reputation history.

An agent can generate a ZK proof that says "my reputation score is at least X" without giving away the exact number. This lets two counterparties negotiate trust at whatever threshold matters to them without either side learning more than needed.

#### 3.2 Reputation Tiers

| Tier | Score Range | What it Unlocks | Audit Access | Validator Rule |
|------|-------------|----------------|--------------|----------------|
| **Basic** | 0 to 59 | Low volume and low value | Public only | Needs 1 or more open validators |
| **Proven** | 60 to 89 | Standard commerce access | Auditor tier available | Needs 3 or more validators, at least 1 trusted |
| **Institutional** | 90 to 100 | Full access, less friction | Full auditor tier | Needs 5 or more validators, at least 1 institutional |

#### 3.3 Score Calculation

Reputation scores use a weighted moving average over a 90 day evidence window.

```
newScore = 0.7 * currentScore + 0.3 * weightedAverage(evidenceWindow)

weightedAverage = sum(evidence.weight * validatorReputation) / sum(validatorReputation)
```

Evidence from the last 7 days carries double weight. Evidence from 7 to 30 days carries normal weight. Evidence from 30 to 90 days carries half weight.

Evidence from institutional validators carries triple weight. Trusted validators carry double. Open validators carry normal weight.

One negative validation takes three positive validations to cancel out the score impact.

If no new evidence arrives for 30 days the score decays by one point each week.

#### 3.4 Evidence Types

Validators add evidence after watching agent behaviour. The evidence types are transaction success, counterparty dispute, audit pass, TEE attestation (optional extension), and manual HITL review.

```
struct Evidence {
    evidenceType: EvidenceType,
    commitment: Hash,          // Hash of the transaction data underneath
    validatorId: ValidatorId,
    timestamp: u64,
    positive: bool,
}
```

#### 3.5 ZK Reputation Proof Interface

```
// Standard interface for reputation gated transactions
function proveReputation(agentId: AgentId, minScore: u8): ReputationProof

// ReputationProof contains:
//   agentId as a public input
//   proof that agent's score meets or exceeds minScore
//   timestamp for proof freshness
//   does NOT reveal exact score or evidence history
```

### 4. Validation Registry

#### 4.1 Hybrid Validator Model

The validation system uses three tiers with rising requirements.

Open validators can be any address that stakes NIGHT. It takes 10,000 NIGHT. Accuracy is tracked. Funds are slashable.

Trusted validators must complete a Foundation audit. It takes 50,000 NIGHT. These validators can validate Proven tier agents.

Institutional validators must prove they completed KYC through an accredited provider. It takes 100,000 NIGHT. These validators can validate any agent and produce compliance reports.

#### 4.2 Validator Registration

Validators register through `registerValidator()`.

```
function registerValidator(
    operator: CompactAddress,
    tier: ValidatorTier,
    stake: u64,
    proof: ValidatorProof  // ZK proof of qualification for the tier
)
```

Open validators need the stake, nothing more. Trusted validators must prove Foundation audit completion. Institutional validators must prove KYC completion through a ZK KYC proof.

#### 4.3 Validation Operation

A validator adds evidence by calling `addEvidence()`.

```
function addEvidence(
    agentId: AgentId,
    evidence: Evidence,
    observationProof: ObservationProof
)
```

The validator's stake is on the line. If a validation gets challenged later and turns out to be wrong, the validator's accuracy score drops and stake may get slashed.

#### 4.4 Validator Accuracy Tracking

```
function reportOutcome(agentId: AgentId, validatorId: ValidatorId, outcome: Outcome)

// SUCCESS adds 1 to validator accuracyScore
// VIOLATION_MISSED subtracts 5, stake gets slashed by SLASH_AMOUNT
// FALSE_POSITIVE subtracts 3, stake gets slashed by half
```

Validators whose accuracy falls below a threshold get demoted or deactivated. The threshold is configurable per tier.

### 5. Disclosure Tier Registry

#### 5.1 Mapping to Midnight's Three Tier Model

MAIS maps agent data to Midnight's disclosure tiers as follows.

| Data Category | Public | Auditor | Private |
|--------------|--------|---------|---------|
| Agent existence | Yes | Yes | Yes |
| Agent metadata | Yes | Yes | Yes |
| Reputation score | Yes | Yes | Yes |
| Current validator set | Yes | Yes | Yes |
| Aggregated activity stats | No | Yes | Yes |
| Validator evidence history | No | Yes | No |
| Transaction contents | No | No | Yes |
| Agent strategy and config | No | No | Yes |
| Operator identity (private agents) | No | Yes* | Yes |

Auditors registered by the agent operator can see operator identity at auditor tier. Unregistered auditors see nothing.

#### 5.2 Auditor Registration

An agent operator can designate auditor keys.

```
function registerAuditor(
    agentId: AgentId,
    auditorAddress: CompactAddress,
    permissions: AuditorPermissions
)
```

`AuditorPermissions` specifies which data categories the auditor can access. Options are ACTIVITY_STATS for aggregate data, EVIDENCE_HISTORY for individual validation records, and OPERATOR_IDENTITY for the operator's address (relevant for private agents).

#### 5.3 Regulator Disclosure

For regulated use cases MAIS defines a standard regulator disclosure flow.

A regulator submits a disclosure request backed by a ZK KYC proof of regulatory authority. The agent operator gets notified off chain. If the operator approves, or if a timeout passes and the agent's policy allows auto disclosure, the firewall produces a structured compliance report at auditor tier. The report is a ZK proof that says "this agent's activity during this period stayed within its declared policy set" without showing individual transaction contents.

### 6. ERC-8004 Cross Chain Identity Bridge

#### 6.1 Optional Integration

ERC-8004 integration is optional. Agents that do not need cross chain identity can register without it. The bridge is a separate Compact contract that agents opt into.

#### 6.2 Bridge Registration

An agent links its Midnight identity to an ERC-8004 identity by calling `linkIdentity()`.

```
function linkIdentity(
    agentId: AgentId,
    erc8004TokenId: u256,
    crossChainProof: CrossChainProof
)
```

The cross chain proof shows three things. The submitter controls the Midnight agent identity, verified through the Identity Registry. The submitter controls the ERC-8004 NFT, verified by a signature from the EVM address holding that NFT checked against a state proof of the ERC-8004 Identity Registry on Ethereum. And both proofs tie back to the same root of trust, the operator's key.

#### 6.3 Cross Chain Reputation

Once linked, an agent's Midnight reputation score can feed back into the ERC-8004 ecosystem. MAIS contracts can read ERC-8004 reputation data as one input to Midnight reputation calculation, weighted alongside Midnight native evidence. MAIS audit results can also be written to ERC-8004's Validation Registry on Ethereum through a cross chain message. This makes Midnight sourced trust data visible in the broader ERC-8004 ecosystem.

#### 6.4 Security Boundary

The bridge contract enforces freshness constraints on cross chain proofs, within one hour of the bridge call. It requires a threshold of confirmations on the source chain before accepting a proof. And it rate limits cross chain state updates to stop flooding attacks.

### 7. Standard Interface Summary

All MAIS compliant contracts must implement this query interface.

```
interface MAISAgent {
    // Identity
    function getAgentId(): AgentId
    function getOperator(): CompactAddress
    function getMetadata(): Metadata
    function getIdentityMode(): IdentityMode  // PUBLIC or PRIVATE

    // Reputation
    function getReputationScore(): u8
    function getReputationTier(): ReputationTier
    function proveReputation(minScore: u8): ReputationProof

    // Validation
    function getValidators(): [ValidatorId]
    function getValidatorCount(): u16

    // Disclosure
    function getDisclosureTier(): DisclosureTier
    function isAuditor(auditorAddress: CompactAddress): bool

    // Cross chain (optional)
    function getErc8004TokenId(): u256?
    function isErc8004Linked(): bool
}
```

## Rationale

### Why a Three Tier Privacy Model

Midnight's core difference from transparent blockchains is its three tier disclosure model covering public, auditor, and private views. MAIS maps directly onto this model rather than hiding it behind an abstraction.

The public tier enables open agent marketplaces. Agents can be discovered without giving up anything sensitive. The auditor tier is the compliance layer. Regulated AI agents in finance, healthcare, and legal domains cannot operate without an auditable trail. Midnight's auditor tier gives them that without exposing strategic data to competitors. The private tier protects competitive advantage. A trading agent's strategy is its moat. A governance agent's voting logic is sensitive. Private tier keeps these behind ZK proofs.

A purely public model like ERC-8004 would throw away Midnight's privacy advantage. A purely private model would prevent verifiable trust. The three tier balance sits in the right place.

### Why Dual Mode Identity

Single mode identity fails one side. Public only forces every agent to be traceable to its operator, which is unacceptable for agents handling sensitive data. Private only stops open agent commerce because you cannot build a marketplace if you cannot find agents.

Dual mode lets the ecosystem choose per use case. Public mode is the default for open ecosystems like marketplaces, DAOs, and public services. Private mode is the option for sensitive applications like institutional trading, healthcare agents, and legal document agents. Both modes expose the same `MAISAgent` interface so consuming contracts and SDKs work identically regardless.

### Why Public Scores with Private Evidence

Three alternatives were considered.

Fully public scores and evidence gives maximum verifiability but lets competitors reverse engineer agent strategy from the evidence trail. This violates Midnight's privacy thesis.

Fully private scores and evidence gives maximum privacy but destroys verifiable trust. Every transaction would need bilateral trust negotiation.

Public scores with private evidence gives verifiable trust without strategy leaks. Anyone reads a score in O(1). When deeper verification is needed a ZK proof can verify the score without exposing the evidence. Evidence commitments prevent reverse engineering. And auditors can access evidence history at auditor tier.

ZK provable scores only, with no public score at all, gives maximum flexibility but adds ZK proof latency to every transaction and removes the public signal needed for quick decisions. This is deferred to a possible extension.

### Why Hybrid Validator Tiers

Pure permissionless validation lets anyone validate. That invites Sybil attacks where an agent operator spins up fake validators to inflate their own score. Pure permissioned validation needs heavy governance and creates a bottleneck for ecosystem growth.

The hybrid model gives permissionless entry with anti Sybil mechanisms through staking, accuracy tracking, and slashing. The trusted tier provides a credibility upgrade for serious validators. The institutional tier gives enterprise customers the regulatory credibility they need. This mirrors how certificate authorities work in TLS, multiple tiers with rising requirements and corresponding trust.

### Why Optional ERC-8004 Bridge

ERC-8004 is the main agent identity standard on EVM chains. Midnight agents that also touch Ethereum or work with Ethereum based agents benefit from cross chain identity linking. But not every Midnight agent needs EVM interoperability. Cross chain bridges add attack surface. Making the bridge optional keeps the core standard simpler and the risk smaller. The bridge is a separate contract. The core identity, reputation, and validation registries work without it.

## Path to Active

### Acceptance Criteria

MIP-X should be considered for Accepted status when three things are true.

First, a working reference implementation of the Identity Registry, Reputation Registry, and Disclosure Tier Registry Compact contracts exists and passes a test suite on Midnight testnet.

Second, the proposal has been presented at a Midnight community workshop and feedback has been incorporated.

Third, at least two Midnight ecosystem projects have said they intend to adopt the standard. This could be Midnight City, AlphaTON, an Aliit Fellowship project, or similar.

MIP-X should be considered for Active status when five things are true.

First, all core contracts are deployed on Midnight testnet and have operated for at least 4 weeks without critical issues. Second, the `@midnight-agent/mais-sdk` TypeScript package is published and documented. Third, at least 5 unique agent identities are registered on testnet with at least 20 reputation evidence operations completed. Fourth, complete developer documentation exists including ARCHITECTURE.md, CONTRIBUTING.md, and a quickstart that gets a test agent registered in under 10 minutes. Fifth, a mainnet deployment plan has been prepared and reviewed.

### Implementation Plan

| Phase | Component | Timeline |
|-------|-----------|----------|
| 1 | Identity Registry contract | Weeks 1 to 4 |
| 2 | Reputation Registry contract | Weeks 4 to 8 |
| 3 | Disclosure Tier Registry contract | Weeks 8 to 10 |
| 4 | Validation Registry contract | Weeks 10 to 14 |
| 5 | ERC-8004 Bridge contract | Weeks 14 to 16 |
| 6 | TypeScript SDK | Weeks 16 to 20 |
| 7 | Testnet deployment, ecosystem onboarding | Weeks 20 to 24 |
| 8 | Security audit, mainnet preparation | Weeks 24 to 28 |

## Backwards Compatibility Assessment

### Impact on Existing Systems

MAIS is a new standard. It does not change any existing Midnight protocol component. No hard fork is needed. Everything lives at the application layer through Compact contracts.

### Compatibility with Existing Midnight Standards

MPS-0003 (CAIP Support) has no conflict. MAIS uses native Midnight identifiers. CAIP chain identifiers can help with cross chain bridge operations but are not needed for core functionality.

MPS-0004 (Trusted Proof Serving) is complementary. MAIS agents that need delegated proof generation can use the TEE attestation flow from MPS-0004 as a validator evidence type.

MPS-0005 (Events) has no conflict. MAIS emits standard Midnight events for public operations. Private operations do not emit events by design.

Midnight ZSwap has no conflict. MAIS agents can hold and transact ZSwap shielded tokens as usual.

### Compatibility with ERC-8004

MAIS is designed to work alongside ERC-8004, not to compete with it. The optional bridge enables interoperability. ERC-8004 agents that link to MAIS keep their ERC-8004 identity. MAIS agents that do not link are unaffected.

### Migration Path

No migration is needed since this is a new standard. Existing ad hoc agent identity implementations on Midnight can adopt MAIS by registering with the Identity Registry, putting the `MAISAgent` interface on their existing contract as a thin wrapper, and optionally moving any existing reputation data into the Reputation Registry.

## Security Considerations

### Threat Model

**Threat Actor 1, the Sybil validator.** An agent operator creates many fake validator identities to inflate their own reputation score.

Mitigation. Open validators must stake 10,000 NIGHT. Accuracy tracking with slashing makes Sybil validation cost more than it earns. The stake cost multiplied by the number of validators needed to meaningfully move a score exceeds the benefit of a higher reputation tier.

**Threat Actor 2, reputation gaming.** An agent runs a series of small safe transactions to pile up positive evidence, then pivots to high value malicious transactions once the reputation is high.

Mitigation. The 90 day sliding window with recency weighting makes reputation a perishable asset. Score decay of one point per week without fresh evidence prevents long term hoarding. Evidence type weighting gives lower weight to low value transactions. The system can be set up to require high value successful transactions before tier upgrades.

**Threat Actor 3, private agent deanonymization.** An observer correlates private agent activity with public metadata patterns like timing, transaction sizes, and counterparty patterns to figure out who runs the agent.

Mitigation. This is a metadata analysis problem, not unique to MAIS. Midnight's privacy layer already gives baseline protection. MAIS adds several things. Private agent IDs are not emitted in events. Credential commitments use salted hashes. The SDK documentation recommends randomised transaction timing and amount rounding for privacy sensitive agents.

**Threat Actor 4, bridge oracle manipulation.** An attacker feeds false ERC-8004 data through the cross chain bridge to mess with Midnight reputation scores.

Mitigation. The bridge requires confirmed source chain state proofs, not oracle reports. Freshness constraints stop stale data. Rate limiting stops flooding. And ERC-8004 data is weighted alongside Midnight native evidence, so even a fully compromised bridge cannot override what Midnight validators say.

**Threat Actor 5, compromised operator key.** If an attacker steals the agent operator's key they could deactivate the agent, transfer control, or change metadata.

Mitigation. Operator transfer needs multi factor proof, either multi sig or a ZK proof of current operator consent. Deactivation is irreversible so an attacker gains nothing from it. Metadata changes are not critical because they do not control transaction authorisation. Key rotation is recommended through Midnight's wallet SDK.

### What MAIS Does Not Protect Against

These boundaries matter for credibility and for understanding where responsibility sits.

MAIS does not guarantee what an agent actually does. It only guarantees that identity, reputation, and validation are verifiable. A highly reputed agent can still make a bad trade. The reputation downgrade that follows is the mechanism, not prevention.

MAIS does not protect against bugs in the Compact contracts that implement the standard itself. Those contracts need their own security audit.

MAIS does not protect against a quantum adversary breaking the underlying ZK proof system. Midnight's Plonk implementation inherits the cryptographic assumptions of BLS12-381.

## Implementation

### Existing Reference Implementation

The [midnight-agent-did-manager](https://github.com/apestchanker/midnight-agent-did-manager) project by apestchanker already provides a working MAIS-aligned Identity Registry on Midnight Preprod. This implementation demonstrates that the core concepts are viable:

- Compact DID registry contract deployed on Midnight Preprod
- W3C-aligned `did:midnight:<network>:<contract>:<agentKey>` identifiers
- 1AM wallet integration for dust-free proving and transaction submission
- Selective disclosure Verifiable Credentials via commitment-based Midnight proofs
- Full MCP server (stdio + HTTP) for AI agent interaction
- Postgres-backed service for DID requests, approvals, and credential issuance
- Browser-safe private state provider patch for Midnight SDK

This implementation serves as the starting point for the MAIS reference implementation. The Identity Registry component is already functional. The Reputation Registry, Validation Registry, and ERC-8004 bridge are the next components to be added, following the architecture described below.

### Compact Contract Structure

The reference implementation has four Compact contracts.

```
midnight-agent-identity/
├── contracts/
│   ├── identity-registry.compact
│   ├── reputation-registry.compact
│   ├── validation-registry.compact
│   ├── disclosure-tier-registry.compact
│   └── erc8004-bridge.compact
├── tests/
│   ├── identity-registry.test.ts
│   ├── reputation-registry.test.ts
│   ├── validation-registry.test.ts
│   ├── disclosure-tier-registry.test.ts
│   └── erc8004-bridge.test.ts
├── sdk/
│   └── mais-sdk/
│       ├── src/
│       │   ├── agent.ts
│       │   ├── identity.ts
│       │   ├── reputation.ts
│       │   ├── validation.ts
│       │   └── disclosure.ts
│       └── package.json
├── ARCHITECTURE.md
├── CONTRIBUTING.md
└── README.md
```

### Dependencies

The implementation depends on the Midnight Compact Runtime for ZK proof generation and verification, the Midnight.js SDK for wallet integration and transaction submission, and Midnight ZSwap for NIGHT staking by validators. ZK Kit is optional for credential commitment schemes in private identity mode. ERC-8004 contract ABIs are optional for the cross chain bridge.

### SDK Interface Design

```typescript
// Initialize agent identity
const agent = new MAISAgent({
  identityMode: 'public' | 'private',
  operator: midnightWallet,
  metadata: {
    name: 'TradingBot Alpha',
    description: 'Automated DEX arbitrage agent',
    capabilities: ['TRADING', 'PAYMENT'],
    version: '1.0.0',
  },
  disclosureTier: 'public',
  // Optional
  erc8004TokenId: 12345,
});

// Register on Midnight
await agent.register();

// Get reputation
const reputation = await agent.getReputation();
// => { score: 85, tier: 'PROVEN', evidenceCount: 42, lastUpdated: '2026-05-15' }

// Prove reputation, ZK proof that score meets threshold
const proof = await agent.proveReputation({ minScore: 60 });
// Submit proof to counterparty contract for verification

// Add evidence as validator
await validator.addEvidence({
  agentId: agent.id,
  evidence: { type: 'TRANSACTION_SUCCESS', positive: true },
  proof: observationProof,
});

// Read audit trail at auditor tier
const auditTrail = await agent.getAuditTrail({
  from: '2026-05-01',
  to: '2026-05-18',
  disclosureTier: 'auditor',
});
```

## Testing

### Unit Tests

Each Compact contract must reach 90 percent or higher branch coverage.

The Identity Registry tests must cover public and private registration, metadata updates, authorised and unauthorised operator transfers, deactivation, and prevention of reactivation after deactivation.

The Reputation Registry tests must cover score calculation edge cases like no evidence, single evidence, and many pieces of evidence. They must test negative penalties, decay over time, tier changes, and ZK proof generation with verification.

The Validation Registry tests must cover validator registration at all three tiers, evidence submission, accuracy tracking, stake slashing, and demotion.

The Disclosure Tier Registry tests must cover tier transitions, auditor registration, unauthorised access attempts, and the regulator disclosure flow.

The ERC-8004 Bridge tests must cover identity linking, cross chain proof verification, replay prevention, and rate limiting.

### Integration Tests

A full agent lifecycle test goes from registration through reputation building, tier upgrade, cross chain linking, and deactivation. A multi agent scenario runs ten agents with interleaved reputation operations. A validator collusion scenario tests multiple validators from the same operator attempting a Sybil attack.

### Stress Tests

Ten thousand concurrent agent registrations. One thousand evidence submissions per minute. One hundred reputation proof generations happening at the same time.

### Test Environment

Tests run against a local Midnight testnet set up through the Midnight.js SDK. A GitHub Actions CI pipeline runs the full test suite on every PR.

## References

- ERC-8004: AI Agent Identity, Reputation, and Validation Registries. https://eips.ethereum.org/EIPS/eip-8004
- Midnight Network Documentation. https://docs.midnight.network/
- Midnight Compact Language Reference. https://docs.midnight.network/develop/reference/compact/
- AlphaTON and Midnight Partnership. Midnight Foundation Announcement, 2026
- Midnight City Simulation. Midnight Network Developer Portal
- Bastion Agentic Defense. On chain security middleware for AI agents (complementary project)
- ZK KYC. Zero Knowledge Proofs for KYC Compliance, reference implementations by various providers
- 1AM Wallet: Dust-free ZK proving and wallet infrastructure. https://1am.xyz/developers
- Midnight Passport: Seven-layer identity infrastructure (LFDT-Nightstream). https://midnight-passport.vercel.app
- Midnight Agent DID Manager: Reference implementation of MAIS Identity Registry on Midnight Preprod. https://github.com/apestchanker/midnight-agent-did-manager

## Acknowledgements

This MIP draws inspiration from ERC-8004's agent identity model by the Ethereum Foundation dAI team, the Bastion Agentic Defense project's security middleware architecture, and Midnight's three tier disclosure privacy model by IOG and the Midnight Foundation. The dust-free transaction model is informed by the 1AM wallet's ProofStation architecture. The credential and recovery architecture is informed by Midnight Passport's seven-layer identity model by LFDT-Nightstream.

## Copyright Waiver

All contributions, both code and text, submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
