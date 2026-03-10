---
MPS: ?
Title: [Succinct, Descriptive Title]
Category: [Node | Ledger | Indexer | SDK | Wallet | Compact | Bridge | DApp | Tooling | Documentation | Operations | Ecosystem]
Status: Open
Authors:
  - [Author Name] <author@example.com>
Proposed Solutions: []
Discussions:
  - [Link to GitHub Discussion]
  - [Link to Discord thread]
Created: YYYY-MM-DD
License: CC-BY-4.0
---

## Abstract

[A short (~200 word) description of the target goals and the technical obstacles to those goals. This should be accessible to someone familiar with Midnight but not necessarily an expert in this specific area.]

## Problem

[Detailed description and context explaining the motivation for this problem statement. Include:
- Current state of the ecosystem
- What limitations or gaps exist
- Why existing approaches are insufficient
- Background context needed to understand the problem
- Impact on users, developers, or the network]

## Use Cases

[Concrete, user-perspective examples showing what users attempt to do and why current alternatives fail. Each use case should be realistic and demonstrate the problem clearly.]

### UC1: [Use Case Title]

**Scenario**: [Describe the concrete scenario]

**Current Approach**: [How users currently try to solve this]

**Limitations**: [Why the current approach fails or is inadequate]

**Desired Outcome**: [What the user needs to accomplish]

### UC2: [Additional Use Case]

[Repeat structure for multiple use cases]

## Goals

[Ranked importance list of objectives, requirements, and evaluation metrics. Be specific and measurable where possible.]

### Primary Goals

1. [Most important objective with measurable success criteria]
2. [Second priority with clear requirements]
3. [Additional goals...]

### Requirements

- **Performance**: [Specific performance requirements, e.g., "proof generation < 5s"]
- **Security**: [Security concerns or threat model considerations]
- **Privacy**: [Privacy guarantees or constraints]
- **Compatibility**: [Backward compatibility or interoperability requirements]
- **Usability**: [Developer or user experience requirements]

### Success Metrics

[How will solutions be evaluated? What measurements determine success?]

- [Metric 1]: [Target value or threshold]
- [Metric 2]: [Evaluation criteria]

### Non-Goals

[Explicitly state what is OUT of scope to prevent scope creep]

- [Thing that is not a goal]
- [Boundary of the problem space]

## Open Questions

[Questions that proposed solutions must address. These help identify potential pitfalls and design considerations.]

1. [Question about design approach or trade-offs]
2. [Question about edge cases or failure modes]
3. [Question about compatibility or migration]
4. [Question about performance or scalability]
5. [Question about security or privacy implications]

## References

[Optional section for related documents, research papers, specifications, or external resources]

- [Reference 1]: [Link or citation]
- [Reference 2]: [Link or citation]

## Appendices

[Optional section for technical details, benchmarks, data, or supporting material]

### Appendix A: [Title]

[Content]

### Appendix B: [Title]

[Content]

## Acknowledgements

[Optional section to thank contributors, reviewers, or inspirations]

## Copyright

This MPS is licensed under [CC-BY-4.0 | Apache-2.0].

---

## Template Notes (Remove before submission)

**Filing Your MPS:**

1. Create a new directory: `MPS-????/`
2. Copy this template to `MPS-????/README.md`
3. Fill in all sections (remove placeholders)
4. Replace `?` in YAML with `????` until number assigned
5. Update `Created` date to ISO 8601 format (YYYY-MM-DD)
6. Choose appropriate `Category` from the list
7. Add any supporting files (diagrams, code examples) to the directory

**Writing Guidelines:**

- **Be Specific**: Vague problems lead to vague solutions
- **Show, Don't Tell**: Use concrete examples and use cases
- **Define Success**: Make goals measurable when possible
- **Acknowledge Constraints**: Be realistic about requirements
- **Ask Hard Questions**: Open questions should probe solution design
- **Stay Focused**: Use non-goals to maintain scope

**Category Selection Guide:**

- **Node**: Substrate runtime, pallets, consensus (AURA/GRANDPA/BEEFY), P2P networking
- **Ledger**: Privacy ledger, ZK proofs (zkir), transaction processing, coin structures
- **Indexer**: Chain indexer, wallet indexer, GraphQL API, data persistence
- **SDK**: midnight-js, contract interfaces, provider APIs, type safety
- **Wallet**: Wallet SDK, HD wallets, coin selection, address format, key management
- **Compact**: Smart contract language, compiler (compactc), type system, circuit generation
- **Bridge**: Cardano integration, cNIGHT tokens, partner chains, cross-chain operations
- **DApp**: DApp connector, browser integration, wallet connections
- **Tooling**: CLI tools, development workflow, testing frameworks, debugging
- **Documentation**: User docs, API docs, tutorials, examples, migration guides
- **Operations**: Node operations, deployment, monitoring, SRE tooling
- **Ecosystem**: Cross-component issues, governance, standards, processes

**Discussion Venues:**

- GitHub Discussions: Long-form technical discussions
- GitHub Issues: Specific bugs or feature requests related to the problem
- Discord: Real-time feedback and community input
- Community Calls: Present problem for synchronous discussion

**Review Process:**

1. Self-review: Ensure all sections are complete and clear
2. Peer review: Share with colleagues familiar with the problem space
3. Community review: Post for broader feedback
4. Maintainer review: Submit PR for official review
5. Iteration: Refine based on feedback
6. Merge: Assigned MPS number and merged when ready

**Common Mistakes to Avoid:**

- Proposing solutions instead of describing problems
- Being too vague or too specific
- Ignoring existing ecosystem components
- Unrealistic performance requirements
- Missing key use cases
- Not defining success criteria
- Skipping open questions

**Good Examples:**

- Problem: "Shielded transactions require users to maintain full private state history, consuming unbounded storage"
- Use Case: "A mobile wallet user with 10,000 historical transactions requires 2GB of local storage, exceeding typical mobile app limits"
- Goal: "Private state storage < 100MB for 10,000 transactions without compromising privacy"
- Open Question: "Can state pruning maintain sufficient privacy guarantees for transaction unlinkability?"

**Bad Examples:**

- Problem: "Midnight needs better performance" (too vague)
- Use Case: "Users want faster transactions" (not concrete)
- Goal: "Make everything work better" (not measurable)
- Open Question: "How do we implement this?" (assumes solution)

---

## Attribution

This template is part of the Midnight Problem Statements (MPS) process, derived from [CIP-9999: Cardano Problem Statements](https://cips.cardano.org/cip/CIP-9999).

**Original Work**: CIP-9999 / Cardano Problem Statements
**Original License**: CC-BY-4.0
**Modifications**: Adapted for Midnight ecosystem

## License

This template is licensed under CC-BY-4.0 (Creative Commons Attribution 4.0 International).

**Template Version**: 1.0.0
**Last Updated**: 2026-03-09
