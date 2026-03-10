# Midnight Problem Statements (MPS)

## Overview

Midnight Problem Statements (MPS) establish a formal process for documenting challenges in the Midnight ecosystem. MPSs complement Midnight Improvement Proposals (MIPs) by clearly articulating problems before solutions are proposed.

## Attribution

This MPS format is derived from [CIP-9999: Cardano Problem Statements](https://cips.cardano.org/cip/CIP-9999) (formerly CPS), created by the Cardano community. The structure, process, and documentation format have been adapted for the Midnight ecosystem while preserving the core principles of problem-first documentation.

**Original Work**: CIP-9999 / Cardano Problem Statements
**License**: CC-BY-4.0 (Creative Commons Attribution 4.0 International)
**Source**: https://github.com/cardano-foundation/CIPs

We extend our gratitude to the Cardano community and the authors of CIP-9999 for establishing this framework.

## Purpose

MPSs serve to:
- Document technical obstacles and ecosystem challenges
- Provide context for solution proposals
- Facilitate community discussion around problem scope
- Track the evolution from problem identification to solution implementation

## Relationship to MIPs

- **MPS**: Describes a problem, use cases, goals, and open questions
- **MIP**: Proposes a specific solution to an MPS or addresses an ecosystem need

MPSs are first-class documents that can exist independently or spawn multiple competing MIPs.

## Status Lifecycle

| Status | Definition |
|--------|-----------|
| **Open** | Well-formulated problem without a complete solution meeting all goals |
| **Solved** | Complete solution found and implemented (references MIPs) |
| **Inactive** | Problem is obsolete, withdrawn, or no longer relevant (with stated reason) |

## File Structure

Each MPS exists in its own directory:

```
MPS-XXXX/
├── README.md          # Main problem statement
└── [supporting files] # Optional diagrams, data, examples
```

Use `MPS-????` as a placeholder until a number is assigned.

## Document Template

See `MPS-TEMPLATE.md` for the complete structure.

## Required Sections

### 1. YAML Preamble
Metadata header with required fields:
- `MPS`: Number (or "?")
- `Title`: Succinct, descriptive heading
- `Category`: Ecosystem area (Node, Ledger, Indexer, SDK, Wallet, Compact, etc.)
- `Status`: Open | Solved | Inactive
- `Authors`: Names with contact information
- `Proposed Solutions`: List of related MIPs (if any)
- `Discussions`: Links to major technical discussions
- `Created`: ISO 8601 date format (YYYY-MM-DD)
- `License`: CC-BY-4.0 or Apache-2.0

### 2. Abstract
~200 word summary of target goals and technical obstacles.

### 3. Problem
Detailed description explaining:
- Current state
- Limitations or gaps
- Why existing approaches fail
- Context and motivation

### 4. Use Cases
Concrete examples from user perspective showing:
- What users attempt to do
- Why current alternatives fail
- Real-world scenarios requiring a solution

### 5. Goals
Ranked list including:
- Project objectives
- Requirements (performance, security, privacy, compatibility)
- Evaluation metrics for solutions
- Non-goals (explicit scope boundaries)

### 6. Open Questions
Questions that proposed solutions must address:
- Design considerations
- Trade-offs to evaluate
- Edge cases to handle
- Compatibility concerns

### 7. Copyright
Explicit license statement.

### Optional Sections
- **References**: Related documents, research papers, specifications
- **Appendices**: Technical details, data, benchmarks
- **Acknowledgements**: Contributors and reviewers

## Categories

MPSs are categorized by ecosystem component:

- **Node**: Substrate runtime, consensus, networking
- **Ledger**: Privacy ledger, ZK proofs, transaction processing
- **Indexer**: Data indexing, GraphQL API, wallet indexing
- **SDK**: Developer tools, midnight-js, contract interfaces
- **Wallet**: Wallet SDK, address format, HD wallets, coin selection
- **Compact**: Smart contract language, compiler, type system
- **Bridge**: Cardano integration, cNIGHT, partner chains
- **DApp**: DApp connector, browser integration
- **Tooling**: CLI tools, development workflow, testing
- **Documentation**: Docs, tutorials, examples
- **Operations**: Deployment, monitoring, node operations
- **Ecosystem**: Cross-component issues, governance, standards

## Licensing

**Recommended licenses:**
- Documentation: CC-BY-4.0
- Code/Software: Apache-2.0

All MPSs must include an explicit license statement in both the YAML preamble and the Copyright section.

### License Requirements

Per CC-BY-4.0 attribution requirements, MPSs must:
- Include original license information
- Indicate if modifications were made
- Provide a link to the license

## Submission Process

1. **Draft**: Author creates MPS in `MPS-????/` directory
2. **Discussion**: Seek feedback via GitHub issues, Discord, or community calls
3. **Refinement**: Iterate on problem formulation based on feedback
4. **Review**: Maintainers review for clarity, completeness, and relevance
5. **Merge**: Well-formulated MPSs are merged with assigned numbers
6. **Solution**: Community proposes MIPs addressing the MPS

## Merging Criteria

An MPS is ready to merge when:
- Problem is clearly articulated with concrete use cases
- Goals and evaluation metrics are well-defined
- Open questions identify solution requirements
- Problem is acknowledged by maintainers as legitimate
- No suitable alternatives exist in the ecosystem

## Rejection Reasons

An MPS may be rejected if:
- Problem scope is unclear or too broad
- Suitable solutions already exist
- Goals are unrealistic or contradictory
- Author has abandoned the proposal
- Problem is specific to one deployment (not ecosystem-wide)

## Solution Phase

Once an MPS is merged:
- Community members can propose MIPs addressing the problem
- Multiple competing solutions may be proposed
- MPS status changes to "Solved" when a complete solution is implemented
- Solved MPSs reference the implementing MIPs

## Resources

- **CIP-9999**: Original Cardano Problem Statement format
- **Midnight Docs**: https://docs.midnight.network
- **GitHub Discussions**: Community problem discussions
- **Discord**: Real-time feedback and refinement

## Quick Start

See `QUICK-START.md` for a 5-minute guide to creating your first MPS.

## Examples

- **MPS-0001**: Efficient Historical State Queries for Shielded Transactions (Indexer category)

## Copyright and License

This MPS process documentation is licensed under CC-BY-4.0.

**Derived from**: CIP-9999: Cardano Problem Statements
**Original Authors**: Cardano community
**Original License**: CC-BY-4.0
**Modifications**: Adapted for Midnight ecosystem with Midnight-specific categories, terminology, and examples

---

**Version**: 1.0.0
**Last Updated**: 2026-03-09
