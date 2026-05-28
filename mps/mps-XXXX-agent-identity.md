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

---
MPS: XXXX
Title: Agent Identity Gap on Midnight
Authors: Zidan (mzf11125)
Status: Proposed
Category: Standards
Created: 18-MAY-2026
Requires: none
Replaces: none

---

## Abstract

AI agents are arriving on Midnight. Midnight City already runs Gemini agents in simulation. AlphaTON is building privacy agents for Telegram's one billion users. But there is no standard way for these agents to say who they are, prove they can be trusted, or get validated by others. Each team building agents on Midnight invents its own identity scheme from scratch. Without a shared standard the ecosystem will fragment into incompatible identity systems.

Additionally, two complementary infrastructure layers are developing in parallel that would benefit from a shared agent identity standard. The 1AM wallet provides dust-free ZK proving via ProofStation, eliminating the need for users and agents to manage DUST balances for transactions. Midnight Passport, a seven-layer identity infrastructure currently in closed-source development, provides hardware-backed key protection, named accounts, and ZK credential issuance. These tools address the onboarding and credential layers respectively, but neither defines the agent-level identity, reputation, and validation primitives that MAIS addresses. This MPS describes the gap before any specific solution is proposed.

## Vision

A Midnight where any AI agent can register a verifiable identity, accumulate provable reputation from third party validators, and selectively disclose that information at Midnight's public, auditor, and private tiers. Developers should be able to query an agent's identity and trust level with one SDK call regardless of who built the agent. This standard should work seamlessly with existing Midnight wallet infrastructure, such as 1AM's dust-free proving model, and be compatible with emerging identity infrastructure like Midnight Passport's credential issuance and recovery systems. The standard should be native to Midnight's ZK architecture rather than adapted from transparent chain standards that would leak competitive or compliance sensitive data.

## Problem

Ethereum has ERC-8004 for agent identity. It provides agent identity and reputation registries on EVM chains. But those chains are transparent so every piece of agent metadata, every reputation score, and every validation record sits in public view. That creates a real problem for agents that operate in competitive or strategic domains. A trading agent's identity and reputation are visible to every competitor who can reverse engineer its strategy from the transaction trail.

Midnight's zero knowledge architecture was designed to solve exactly this tension. Selective disclosure means an agent can prove its identity and reputation without showing its full history to the network. But Midnight does not yet have a standard interface for agents to do this in any of the following areas. Identity registration, so there is no common way for an agent to say who it is or who operates it. Reputation, so there is no shared mechanism for tracking trust over time. Validation, so there is no ecosystem for third parties to attest to agent behaviour. Disclosure control, so there is no standard mapping of agent data onto Midnight's three tier privacy model.

Three things make this standard urgent. Midnight mainnet went live on 30 March 2026 and developer activity surged over 1,600 percent after the Summit in late 2025. The window to establish a foundational identity standard before ten incompatible schemes emerge is open right now. The AlphaTON partnership targets Telegram's one billion users with privacy preserving AI agents that need native Midnight identity. Midnight City Simulation uses Google Gemini agents as a live public testbed with no formal identity standard.

## Use Cases

A trading agent needs to prove its reputation score exceeds a threshold to a counterparty without revealing its full transaction history. Today this requires bilateral trust negotiation or sharing raw data over unverifiable channels.

A compliance agent handling regulated financial data needs to grant auditors access to its activity history while keeping its strategy hidden from competitors. Today there is no standard way to register an auditor key with granular permission levels.

A DAO governance agent needs a public identity so delegates can discover it but must keep its voting logic private. Today the agent builder must invent both the discovery mechanism and the privacy layer from scratch.

A validator wants to attest to an agent's behaviour by submitting on chain evidence. Today there is no registry for validators and no incentive structure to keep them honest through staking and slashing.

## Goals

1. A standard identity registration mechanism that works for both publicly discoverable agents and agents that need operational privacy, with both modes exposing the same interface.

2. A reputation system where trust scores are publicly readable but the underlying evidence stays private, verifiable through ZK proofs when deeper inspection is needed.

3. A validation framework with tiered validators that have anti Sybil protections through staking, accuracy tracking, and slashing.

4. A disclosure mapping that lets agents assign data to Midnight's public, auditor, and private tiers.

5. Optional interoperability with ERC-8004 on Ethereum for agents that operate across both chains.

6. An SDK that lets developers register an agent on Midnight with one function call and query identity and reputation in standard ways.

## Expected Outcomes

After the gap is addressed developers building agents on Midnight will use a shared identity standard instead of ad hoc schemes. Agents will be discoverable by other contracts and applications through a common interface. Trust decisions will be verifiable through ZK proofs rather than off chain promises. Validators will have economic incentives to provide honest attestations. Midnight agents will have a path to interoperability with the broader EVM agent ecosystem. The Midnight City and AlphaTON teams will be able to adopt a standard rather than building their own.

## Open Questions

1. Should the identity standard ship with both public and private identity modes at launch or start with public and add private later?

2. Is a public reputation score backed by private evidence the right balance or should scores also be fully private and accessed only through ZK proofs?

3. What staking amounts and slashing curves make the validator economics resistant to Sybil attacks without being too expensive for honest validators?

4. Should agent metadata include a field for payment information such as what the agent charges and how to pay it?

5. How should MAIS interact with the existing MPS standards, particularly MPS-0004 for Trusted Proof Serving and MPS-0005 for Events?

6. Should MAIS integrate with 1AM wallet's dust-free proving model so that agent registration and reputation attestation can be submitted without requiring agents to manage DUST balances?

7. How should MAIS credentials interoperate with Midnight Passport's credential layer (zkMe zkKYC) once Passport's API becomes publicly available? Should agent identities be resolvable through Passport's naming layer (alice.midnight)?

## Recommended MIPs

**MIP: Midnight Agent Identity Standard (MAIS).** A Compact contract framework defining four registries (Identity, Reputation, Validation, Disclosure Tier) with dual mode identity (public and private), public scores with private evidence, tiered validators with anti Sybil protections, disclosure tier mapping onto Midnight's three tier privacy model, and an optional ERC-8004 cross chain bridge. This MIP addresses all the use cases and goals described above.

## References

- ERC-8004: AI Agent Identity, Reputation, and Validation Registries. https://eips.ethereum.org/EIPS/eip-8004
- Midnight Network Documentation. https://docs.midnight.network/
- AlphaTON and Midnight Partnership. Midnight Foundation Announcement, 2026
- Midnight City Simulation. Midnight Network Developer Portal
- 1AM Wallet: Dust-free ZK proving and wallet infrastructure. https://1am.xyz/developers
- Midnight Passport: Seven-layer identity infrastructure (closed-source, LFDT-Nightstream). https://midnight-passport.vercel.app

## Acknowledgements

This MPS was developed with input from the Midnight Foundation community and draws on discussion from the Midnight Discord developers channel.

## Copyright

This MPS is licensed under CC-BY-4.0.
