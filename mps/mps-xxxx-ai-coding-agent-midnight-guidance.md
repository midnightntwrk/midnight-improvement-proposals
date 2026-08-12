---
MPS: "xxxx"
Title: AI Coding Agents Lack Reliable Midnight Domain Guidance
Authors: Neeraj Choubisa @Kali-Decoder
Status: Draft
Category: Libraries and Tooling
Created: 12-AUG-2026
Requires: MPS-0002
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

Developers increasingly use AI coding agents (Cursor, Claude Code, Codex, Copilot, and similar tools) to learn Midnight and scaffold Compact contracts, wallets, and dApps. Those agents today rely on general training data and ad-hoc web search. They often invent Compact APIs, misstate privacy boundaries, skip Dust and proof-server requirements, and recommend patterns that work on transparent chains but break under Midnight’s shielded model.

Midnight already invests in SDKs, docs, and human-oriented developer tooling (see MPS-0002). What is missing is a clear problem definition for **agent-consumable** Midnight knowledge: how coding agents should discover trustworthy Midnight skills, packages, examples, and constraints so that AI-assisted development is accurate, privacy-safe, and reproducible across tools.

This MPS scopes that gap without prescribing a single product, registry host, or vendor. It establishes goals and recommended MIP directions so the ecosystem can standardize how Midnight knowledge is packaged for AI agents, complementary to the broader developer tooling agenda in MPS-0002.

## Vision

A developer can open any mainstream AI coding agent, ask to build or debug a Midnight application, and receive guidance that:

- reflects current Compact, SDK, wallet, indexer, and network realities;
- distinguishes shielded, unshielded, and Dust concerns correctly;
- points to installable, versioned skill or knowledge packages rather than hallucinated APIs;
- works consistently across agent products through shared conventions;
- degrades safely when uncertain — asking clarifying questions instead of inventing protocol behavior.

Success means AI-assisted Midnight development accelerates onboarding without becoming a source of incorrect privacy or security advice.

## Problem

MPS-0002 describes friction across the human developer lifecycle: SDK consistency, testing, docs, scaffolding, and “coding agent integration” as a recommended direction. Since that MPS was drafted, AI coding agents have become a primary onboarding path for many builders. The Midnight-specific problem for agents is distinct and acute:

### 1. Unreliable domain knowledge

General models lack durable, version-aligned knowledge of Compact, midnight.js, DApp Connector flows, proof servers, Dust, and selective disclosure. Agents invent function names, ledger ADTs, and wallet APIs, or copy Ethereum patterns that do not map to Midnight.

### 2. No shared packaging surface for agent skills

There is no ecosystem-agreed way to publish “skills,” prompts, or knowledge packs that coding agents can install and trust for Midnight. Each team invents private prompt libraries, Cursor rules, or one-off RAG corpora. Agents and humans cannot share a common, reviewable package format or discovery path.

### 3. Documentation is human-first, not agent-operable

Official docs and tutorials are essential but are optimized for reading in a browser. Agents need structured, task-oriented, machine-installable units (prerequisites, stack choices, required skills, known failure modes) that can be retrieved and applied during a coding session without scraping fragile HTML.

### 4. Privacy and security advice is easy to get wrong

Incorrect agent guidance has higher downside on Midnight than on transparent chains: leaking what should stay private, mishandling witnesses, or skipping Dust/proof constraints can produce apps that compile in demos but fail ethically or operationally in production.

### 5. Fragmentation across tools

Builders use different IDEs and agents. Without shared conventions, Midnight “support” for agents becomes N bespoke integrations instead of one problem definition with interoperable solutions.

### Relationship to existing MPS documents

- **MPS-0002 (Developer Tooling)** remains the parent problem space for SDKs, CLIs, templates, and docs. This MPS narrows the **AI coding-agent consumption** slice that MPS-0002 flags but does not fully specify.
- **MPS-0015 (Agent Identity)** addresses on-chain identity and reputation for autonomous agents operating *on* Midnight. This MPS addresses developer-facing coding agents that help humans *build* Midnight software. The two are complementary, not duplicates.

## Use Cases

### New builder onboarding via an AI IDE

A developer new to Midnight asks an agent: “Help me build a private voting dApp.” Today the agent may skip Compact prerequisites, invent contract APIs, or omit wallet and Dust constraints. The developer wastes days correcting hallucinations instead of learning the real stack.

### Experienced engineer accelerating a feature

A Compact-literate engineer asks an agent to wire midnight.js providers and an indexer query. Without grounded, versioned skill content, the agent mixes outdated SDK patterns with current docs, producing code that fails only at prove or submit time.

### Team standardizing AI-assisted workflows

A team wants every engineer’s agent (Cursor, Claude Code, Codex) to use the same Midnight skill set and learning paths. Today they manually sync private prompt files with no shared registry semantics, versioning, or review process.

### Ecosystem educator publishing reusable guidance

A docs or community contributor wants to publish a “wallet integration” skill that any compliant agent can install. There is no agreed package shape, metadata, or discovery convention for Midnight agent skills.

### Privacy-sensitive review

A security-minded reviewer asks an agent to audit a Compact contract for disclosure leaks. Without curated knowledge of Midnight privacy pitfalls, the agent gives generic Solidity-style advice that misses Midnight-specific failure modes.

## Goals

1. **Define the agent-guidance gap** as a first-class Midnight tooling concern, subordinate to but distinct from general SDK/docs work in MPS-0002.
2. **Enable trustworthy retrieval** of Midnight domain knowledge for coding agents (skills, examples, constraints, learning paths) with clear provenance and version alignment.
3. **Encourage interoperable packaging conventions** so the same Midnight skill content can be consumed by multiple AI coding tools without per-vendor forks.
4. **Prioritize privacy-correct defaults** in agent-consumable content: selective disclosure, Dust, proof generation, and wallet boundaries must be explicit.
5. **Support measurable onboarding improvement**: reduce time-to-first-correct Compact/dApp scaffold when using AI agents, and reduce incidence of invented APIs in agent output.
6. **Remain solution-agnostic**: allow official docs sites, community registries, IDE extensions, RAG services, or hybrid approaches — without mandating a single product.

## Expected Outcomes

- Faster, safer AI-assisted onboarding to Midnight for new developers.
- Fewer hallucinated Compact/SDK APIs in agent-generated code.
- Shared vocabulary for “Midnight skills / agent packages” that MIPs can standardize.
- Cleaner separation between: (a) human docs, (b) agent-consumable packages, and (c) on-chain agent identity (MPS-0015).
- Stronger alignment between Tooling Workforce efforts under MPS-0002 and the AI-assisted developer journey.

## Open Questions

1. Should agent-consumable Midnight skills live primarily under official Midnight docs, a community registry, IDE marketplaces, or a combination with clear provenance rules?
2. What minimum metadata (network/target SDK versions, privacy tier, skill level, license) must every Midnight agent skill declare?
3. How should skills be versioned and deprecated when Compact or SDK breaking changes land?
4. What trust model is required — signed packages, curated allowlists, reputation, or “install at your own risk” with clear warnings?
5. How much of this belongs as a Recommended MIP under MPS-0002 versus a standalone standards track (package format + discovery)?
6. How should agent-guidance work relate to existing CLI scaffolding and templates without duplicating them?

## Recommended MIPs

1. **Midnight Agent Skills Package Format**  
   Specify a portable package shape (manifest, skill body, metadata, license, target tool compatibility) for Midnight domain skills consumable by coding agents. Addresses packaging fragmentation.

2. **Midnight Agent Skills Discovery and Provenance**  
   Define how skills are listed, versioned, verified, and deprecated (registry semantics, checksums/signatures, network/SDK compatibility fields). Addresses trust and discovery.

3. **Agent-Oriented Midnight Knowledge Architecture**  
   Define how official docs, examples, templates, and skills relate for retrieval by agents (task-oriented indexing, required vs optional skills, privacy checklist hooks). Addresses grounding quality without prescribing a single RAG vendor.

4. **Coding Agent Integration Profiles (under MPS-0002)**  
   Extend MPS-0002’s “Developer Workflow and Scaffolding / coding agent integration” direction with concrete profiles for major agents (install commands, project hooks, evaluation fixtures). Addresses cross-tool consistency.

5. **Privacy-Safe Agent Guidance Checklist**  
   Standardize the non-negotiable privacy and ops checks agents must surface (Dust, proof server, disclose boundaries, wallet scopes). Addresses high-cost hallucination modes.

## References

- [MPS-0002: Midnight Developer Tooling](./mps-0002-developer-tooling.md) — parent tooling problem statement; Recommended MIP #4 mentions coding agent integration.
- [MPS-0015: Agent Identity Gap on Midnight](./mps-0015-agent-identity.md) — complementary problem for on-chain agent identity (not coding-agent DX).
- [Official Midnight Docs](https://docs.midnight.network/) — primary human documentation surface.
- Community experiments in agent-installable Midnight skills and registries (e.g. public Midnight skills packages and marketplace UIs) demonstrate demand; this MPS does not endorse any specific implementation.

## Acknowledgements

This draft builds on the Midnight Tooling Workforce framing in MPS-0002 and on feedback from builders using AI coding agents to learn Compact and midnight.js. Community discussion is invited via the Midnight Improvement Proposals Discussions tab and Discord before formal proposal.

## Copyright

This MPS is licensed under CC-BY-4.0.
