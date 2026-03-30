---
MIP: ?
Title: Midnight Problem Statements
Authors:
  - Noel Rimbert  (noelrim)
Status: Proposed
Category: Governance
Created: 2026-03-28
Requires: MIP-0001
Replaces: none
License: Apache-2.0
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

This MIP introduces Midnight Problem Statements (MPS) as a formal document type within the MIP process. An MPS captures a problem facing the Midnight ecosystem without prescribing a solution. It exists alongside MIPs in the same repository, giving the community a shared format to describe problems, articulate use cases, and define the goals that any solution must meet.

## Motivation

MIP-0001 already references MPS documents ("Proposals addressing a specific MPS should also be listed in the corresponding MPS header, in 'Proposed Solutions'"), but no formal definition exists. This gap matters for two reasons.

First, MIPs are designed around solutions. Their template asks for a specification, rationale, implementation plan, and backwards compatibility assessment. When a problem is not yet well understood or when multiple solution paths exist, forcing authors into the MIP format is premature. The result is either a MIP with a thin motivation section that undersells the problem, or a MIP that tries to be both problem statement and solution, doing neither well.

In practice, this has led to MIPs that read more like product requirements documents than technical specifications. Authors who want to capture use cases, prioritized goals, and success criteria have no other format available, so they put that content into a MIP. The MIP then carries both the problem framing and the solution, making it harder for reviewers to evaluate either on its own terms. MPS gives that product-level content a proper home, so MIPs can stay focused on technical design.

Second, the Midnight ecosystem is young. Many of the challenges facing builders, node operators, and governance participants are still being identified and scoped. A format for structured problem discovery invites broader participation. Contributors who understand a problem deeply but are not yet ready to propose a solution can still contribute meaningfully.

MPS documents fill this gap. They give problems a home, a structure, and a lifecycle. When solutions emerge, they are proposed as MIPs that reference the originating MPS.

## Specification

### What is an MPS

A Midnight Problem Statement is a document that describes a problem, gap, or challenge facing the Midnight ecosystem. It does not propose a solution. Instead, it defines the problem space clearly enough that the community can evaluate candidate solutions against a shared set of goals.

An MPS may be authored by anyone: protocol developers, application builders, node operators, governance participants, or community members.

### Document Structure

An MPS document consists of a YAML preamble followed by mandatory and optional sections, written in Markdown.

#### Preamble

```yaml
---
MPS: <number>
Title: <title>
Status: <Open | Solved | Inactive>
Category: <Core | Standards | Networking | Governance | Informational | Tools>
Authors:
  - <Name> (<github-username>)
Proposed Solutions:
  - <MIP-XXXX (if any)>
Discussions:
  - <link to GitHub discussion or PR>
Created: <YYYY-MM-DD>
License: Apache-2.0
---
```

**Field descriptions:**

| Field | Description |
|---|---|
| MPS | Number assigned by MIP Editors upon merge. Use `????` in draft submissions. |
| Title | Short, descriptive title of the problem. |
| Status | Current lifecycle status (see Lifecycle section). |
| Category | One of: Core, Standards, Networking, Governance, Informational, or Tools. |
| Authors | One or more authors with GitHub usernames. |
| Proposed Solutions | List of MIPs that propose solutions. Updated as MIPs are submitted. |
| Discussions | Links to related discussions, forum threads, or PRs. |
| Created | Date the MPS was first drafted. |
| License | Must be Apache-2.0. |

#### Mandatory Sections

**Abstract** (~200 words)

A plain summary of the problem: what is broken or missing, who is affected, and why it matters.

**Problem**

A detailed description of the problem, including context, root causes, and scope. This section should establish that the problem is real, significant, and not already addressed by existing tools or proposals. Authors should not prescribe solutions here. The goal is a shared understanding of the problem space.

**Use Cases**

Concrete scenarios, written from the perspective of affected users or builders. Each use case should describe what the user is trying to accomplish, what they can do today (if anything), and where they are blocked. Use cases help solution authors understand practical impact and prioritize accordingly.

**Goals**

A ranked list of requirements that any solution must satisfy. Goals should be specific enough to evaluate a proposed solution against. Where possible, goals should describe observable outcomes rather than implementation approaches.

**Open Questions**

Unresolved questions that the community or solution authors should consider. These may concern scope, technical trade-offs, dependencies, or areas where the problem is not yet fully understood.

**Copyright**

```
This MPS is licensed under the Apache License, Version 2.0.
```

#### Optional Sections

- **References**: External sources, related work, or prior art.
- **Appendices**: Supporting data, diagrams, or analysis.
- **Acknowledgements**: Contributors beyond the listed authors.

### Lifecycle

An MPS progresses through three statuses:

| Status | Meaning |
|---|---|
| **Open** | The problem is well-formulated but no complete solution exists yet. One or more MIPs may be in progress. |
| **Solved** | A MIP that addresses all stated goals has been accepted or implemented. Format: `Solved: by MIP-XXXX`. |
| **Inactive** | The problem is no longer relevant, has been withdrawn by the authors, or has been superseded. The reason must be stated. |

An MPS is merged as **Open**. It moves to **Solved** when the MIP Editors and authors agree that a proposed solution satisfies the stated goals. It may be marked **Inactive** at any time by the authors, or by the Editors if the problem no longer applies.

### Relationship to MIPs

MPS and MIP documents serve different purposes:

- An **MPS** describes a problem. It has no specification, no implementation plan, and no backwards compatibility section, because it does not propose a change.
- A **MIP** proposes a solution. Its motivation section may reference one or more MPS documents to establish context, and the MIP should be listed in the corresponding MPS header under Proposed Solutions.

A single MPS may have multiple proposed MIPs. A single MIP may address multiple MPS documents. The relationship is tracked through the Proposed Solutions field in the MPS preamble and through references in the MIP motivation section.

### Submission and Review

MPS documents follow the same submission process as MIPs, defined in MIP-0001:

1. Authors submit an MPS as a pull request. The PR title uses `MPS-????` as placeholder; Editors assign the number upon merge.
2. MPS documents are stored in their own directory at the repository root: `MPS-XXXX/README.md`, alongside `mips/`.
3. MIP Editors review for clarity, completeness, and whether the problem demonstrably exists. An MPS does not need to demonstrate consensus on a solution, only that the problem is real and worth solving.
4. Once merged, the MPS is listed in the repository README alongside MIPs.

### Merge Criteria

For an MPS to be merged, it must:

- Follow the structure defined in this MIP.
- Describe a problem that demonstrably exists (not hypothetical).
- Be written clearly enough for the community to evaluate proposed solutions against the stated goals.
- Not duplicate an existing Open MPS without meaningful new framing.

An MPS does **not** need to:

- Propose a solution.
- Demonstrate community consensus.
- Include implementation details.

## Rationale

### Why not extend MIPs?

The MIP template is built around solutions: specification, rationale, implementation, testing. Repurposing it for problem statements would mean leaving most sections empty or overloading the motivation section. A dedicated format keeps both document types focused.

### Why not use GitHub Issues?

Issues work well for bugs and feature requests in specific repositories. Problems that span multiple components, affect multiple stakeholders, or require architectural decisions benefit from the permanence, structure, and version history that a repository document provides. Issues are good for discussion; MPS documents are good for shared reference.

### Why add a Tools category?

The MIP-0001 categories (Core, Standards, Networking, Governance, Informational) cover protocol and process concerns well. However, problems affecting developer tooling, such as indexers, SDKs, wallets, and explorers, do not fit neatly into any existing category. The Tools category acknowledges that builder experience is a distinct concern for the ecosystem.

### Why Apache-2.0?

The Midnight Improvement Proposals repository is licensed under Apache-2.0, and MIP-0001 requires the same for all contributions. MPS documents follow this convention for consistency within the repository.

## Path to Active

### Acceptance Criteria

This MIP reaches Active status when:

- At least one MPS has been submitted, reviewed, and merged following the process defined here.
- The MPS directory structure is established in the repository.
- MIP Editors confirm the process works in practice and no blocking issues have been identified.

### Implementation Plan

1. Merge this MIP to establish the MPS process.
2. Submit the first MPS (Governance Observability for Builders) as a reference implementation of the format.
3. MIP Editors review the end-to-end workflow and confirm Active status.

## Backwards Compatibility Assessment

This MIP introduces a new document type. It does not modify any existing MIP, protocol component, or tooling. MIP-0001 already references MPS documents in its submission notes. No backwards compatibility concerns exist.

## Security Considerations

MPS documents describe problems. They do not introduce code, protocol changes, or new attack surfaces. No security implications arise from this proposal.

## Implementation

Implementation consists of:

- Updating the repository README to describe the MPS document type and link to this MIP.
- MIP Editors incorporating MPS review into their existing triage workflow.

No code changes, protocol modifications, or tooling updates are required.

## Testing

The process is validated by submitting the first MPS and verifying that:

- The document follows the structure defined here.
- MIP Editors can review and merge it using existing workflows.
- The Proposed Solutions field can be updated when a MIP references the MPS.

## References

- [MIP-0001: Midnight Improvement Proposal Process](./mip-0001-Midnight-Improvement-Proposal-Process.md)
- [CIP-9999: Cardano Problem Statements](https://cips.cardano.org/cip/CIP-9999)

## Acknowledgements

The MPS format is directly inspired by the Cardano Problem Statement (CPS) process, defined in CIP-9999 by Matthias Benkort and Michael Peyton Jones of the Cardano Foundation and IOG respectively.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement, which includes the assignment of copyright for your contributions to the Foundation.
