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
MPS: 0002
Title: Midnight Developer Tooling
Authors: Midnight Tooling Workforce
Status: Proposed
Category: Libraries and Tooling
Created: 23-JAN-2026
Requires: none
Replaces: none

---

## Abstract

The Midnight community needs a unified, developer-centric tooling layer on top of the Midnight SDK that makes building privacy-preserving applications intuitive, fast, and reliable. Midnight should offer developers a complete environment covering smart contract development, dApp integration, testing, debugging, deployment, and observability, so they can move from idea to production with clarity and confidence. This problem statement defines the current gaps and establishes goals for future Midnight Improvement Proposals (MIPs).

## Vision

The ideal Midnight developer journey is simple (easy installation, setup, integrations), fast (rapid iteration with minimal friction), predictable (consistent APIs, reliable behavior, meaningful errors), privacy-first (shielded and unshielded flows that are easy to build, test, and debug), and end-to-end (tooling that supports build, test, deploy, and operate).

## Problem

Developers building on Midnight currently face friction across the full development lifecycle. Feedback from the community of builders has identified gaps in cross-platform support, SDK consistency, state access patterns, testing workflows, error visibility, and documentation coherence. These gaps slow adoption and limit developers' ability to ship production applications.

The core opportunity is to provide **a unified, productive tooling layer that enables developers to build production applications on Midnight with confidence.**

Key areas where developers need stronger support:

*   Cross-Platform Experience: Developers need consistent behavior across web, mobile, and backend environments
*   SDK and Integration: Developers need unified patterns and clear abstractions across workflows
*   State and Observability: Developers need better visibility into state transitions and application behavior
*   Transaction and Privacy: Developers need simpler transaction construction and easier-to-understand privacy flows
*   Contract Lifecycle: Developers need streamlined processes and integrated tooling for faster iteration
*   Feedback and Debugging: Developers need clearer error messages and more actionable debugging tools
*   Documentation: Developers need consolidated information with cohesive guidance
*   End-to-End Workflow: Developers need a unified journey from prototype to production

## Use Cases

The following scenarios illustrate real developer needs that a unified tooling layer should address:

*   Web dApp Integration: A dApp developer integrating wallet signing, proof generation, and state queries into a browser-based application needs consistent SDK behavior and clear integration patterns.
*   Local Contract Testing: A smart contract developer testing Compact logic locally before deployment needs a reliable simulation environment with meaningful feedback on state transitions and privacy behavior.
*   Team Onboarding: A team onboarding new engineers to build on Midnight needs consolidated documentation, working examples, and a clear path from tutorial to production code.
*   Mobile Development: A mobile developer building a React Native application with privacy features needs first-class SDK support that handles cryptographic constraints without manual workarounds.
*   Production Debugging: A developer troubleshooting a deployed application needs observability tools that surface transaction outcomes, state changes, and errors without compromising user privacy.
*   Scalable State Access: An application handling high transaction volumes needs efficient patterns for querying and paginating state data as the application scales.

## Goals

*   Consistent, predictable SDK behavior across browser, JS frameworks, and server environments
*   Developer-friendly APIs that simplify wallet interactions, transaction construction, and state access
*   Mobile and browser extension development as a first-class capability
*   Streamlined project setup and development workflow from prototype to production
*   Complete lifecycle support for testing, simulation, deployment, and observability
*   Single source of truth for documentation, API conventions, and learning paths

## Expected Outcomes

By delivering this tooling layer, Midnight will unlock:

*   Faster onboarding: reduced time for new developers to begin building
*   Greater productivity: faster build-test-deploy cycles and more experimentation
*   Higher-quality applications: fewer errors through better testing and feedback
*   Broader ecosystem participation: lower barriers enabling more community contributions
*   More effective use of privacy features: making ZK-powered applications practical

## Open Questions

*  Is there a formal plan or timeline for a new implementation of the Midnight SDK and its associated end-user libraries?
*  Will all forthcoming documentation and formalization efforts be consolidated under the [Official Midnight Docs](https://docs.midnight.network/), or is a separate repository intended for community-led resources?
*  Does the current WASM strategy require a formal architectural review to address cross-platform limitations, or should all future tooling be developed to support the technology in its current state?
*  How should the categorization and listing of existing and future tools be managed? Should this be hosted within the official documentation or on a separate, collaborative platform? 

## Recommended MIPs

The following MIPs are recommended to address the problem domains:

1.  Midnight SDK Layer Architecture Core SDK structure, module boundaries, and compatibility requirements for browser, JS frameworks (React, Next.js, Vue), and server environments (Node.js). Versioning strategy and documentation standards. (Initial focus on JS/TS)
2.  Developer End User Libraries Higher-level abstraction layer focused on developer experience. Ergonomic APIs for wallet interactions, transaction construction, state queries, pagination, and indexer integration.
3.  Mobile Support Mobile compatibility, cryptographic library constraints, and platform-specific patterns for iOS and Android.
4.  Developer Workflow and Scaffolding Standard Project templates, CLI tooling, UI component libraries, and coding agent integration.
5.  Contract Testing/Simulation Framework and Deployment Tool Local simulation, testing harnesses for Compact contracts, deployment pipelines, and observability tools for debugging and transaction inspection.
6.  Unified API and Documentation for Libraries and Tooling Single source of truth for documentation, consistent API conventions, and structured learning paths.

## Acknowledgements

This MPS was developed through the collaborative efforts of the Midnight Tooling Workforce, incorporating feedback from partner workshops at the Midnight Summit and ongoing weekly sync sessions. Contributing partners include:

*   OpenZeppelin
*   Webisoft
*   Brick Towers
*   FiWork
*   Midnames
*   Vialabs
*   Balance
*   Mesh/Nebula (Edda Labs)
*   Shielded Technologies
*   Midnight Foundation

## Copyright

This MPS is licensed under CC-BY-4.0.