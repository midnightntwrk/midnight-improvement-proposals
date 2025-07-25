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

---
MIP: ?  \
Title: Midnight Problem Statements (MPS)  \
Authors: Erick ErickRomeroDev \
Status: Proposed \
Category: Governance \
Created: 2025-07-25 \
Requires: none \
Replaces: none 
---

## Abstract

Midnight Problem Statement (MPS) is a structured document for articulating and registering problems within the Midnight ecosystem. It provides a standardized way to define complex challenges faced by developers, researchers, and governance actors. The MPS process complements the MIP system by clearly delineating problems independently from specific proposals, enabling solution-driven discussion, streamlined funding, and community collaboration.

## Motivation

In the Midnight ecosystem, which introduces privacy-preserving primitives, complex design and implementation challenges are often underexplored in MIPs due to insufficient problem framing. The _Motivation_ section of MIPs is sometimes not used thoroughly enough to reflect the multidimensional nature of Midnight-specific issues, such as (but not limited to): privacy implications, compliance requirements, zero-knowledge constraints, scalability implications, and development complexity.

The introduction of the **Midnight Problem Statements** (MPSs) addresses this gap by defining a formal template and structure around the description of problems. MPSs are meant to replace the more elaborate motivation of complex MIPs. However, they may also exist on their own as requests for proposals from ecosystem actors who've identified a problem but are yet to find any suitable solution.

## Specification

### MPS Document Structure

Each MPS follows a standardized Markdown format:

| Section            | Description |
|--------------------|-------------|
| **Preamble**        | YAML metadata block (MIP number, title, authors, etc.) |
| **Abstract**        | Brief description of the problem and technical challenges |
| **Problem**         | Thorough explanation of the issue, its context, and urgency |
| **Use Cases**       | Practical examples of what users are trying to do and what blocks them |
| **Goals**           | Requirements and non-goals that shape the design space |
| **Open Questions**  | Areas that need community or expert input before a solution is attempted |
| _optional sections_         | References, Appendices, Acknowledgements | 
| **Copyright**                                     | The MPS must be explicitly licensed under acceptable copyright terms

Each MPS is stored in a folder named `MPS-XXXX/README.md`. Supplementary files (e.g., diagrams or code samples) may be added. Before a number is assigned, use ? as a placeholder name

#### Example Directory

```
MPS-XXXX/
├── README.md
├── diagram.png
└── ...
```

##### Header preamble

Each MPS must begin with a YAML key:value style header preamble (also known as 'front matter data'), preceded and followed by three hyphens (`---`).

Field                | Description
---                  | ---
`MPS`                | MPS number, or "\?" before being assigned
`Title`              | A succinct and descriptive title
`Category`           | One registered or well-known category covering one area of the ecosystem.
`Status`             | Open \| Solved \| Inactive (..._reason_...)
`Authors`            | A list of authors' real names and email addresses 
`Proposed Solutions` | A list of MIPs addressing the problem, if any
`Discussions`        | A list of links where major technical discussions regarding this MPS happened. Links should include any discussion before submission, a link to the pull request that created the MPS, and any pull request that modifies it.
`Created`            | YYYY-MM-DD

#### Statuses

From its creation onwards, a problem statement evolves around the following statuses.

Status       | Description
---          | ---
Open         | Any problem statement that is fully formulated but for which there still exists no solution that meets its goals. Problems that are only partially solved shall remain _'Open'_ and list proposed solutions so far in their header's preamble.
Solved       | Problems for which a complete solution has been found and implemented. When solved via one or multiple MIPs, the solved status should indicate it as such: `Solved: by <MIP-XXXX>[,<MIP-YYYY>,...]`.
Inactive    | The statement is deemed obsolete or withdrawn for another reason. A short reason must be given between parentheses. For example: `Inactive (..._reason_...).

#### Categories

As defined in MIP-0001.

### The MPS Process

#### 1. Early Stages

##### 1.a. Authors open pull requests with their problem statement

MPS authors initiate the process by submitting a pull request (PR) to the Midnight MIPs GitHub repository, placing their problem statement in a folder named `MPS-?/README.md`. The placeholder `?` should be used until an MIP editor assigns an official number.

##### 1.b. Authors seek feedback

During the open PR stage, authors are encouraged to gather community feedback via:

- GitHub comments on the PR
- [Midnight Community Forum](https://forum.midnight.network)

Feedback aims to clarify scope, identify duplicates, and refine open questions and goals.

#### 2. Editors' Role

##### 2.a. Triage in bi-weekly meetings

MIP editors conduct triage during regularly scheduled meetings. They assess new MPS submissions for clarity, relevance, technical completeness, and format compliance. Editors may provide feedback or request revisions before an MPS is eligible for merge.

##### 2.b. Reviews

Editors provide structured reviews to ensure:

- The problem is clearly defined and distinct
- Use cases are well-articulated and grounded in ecosystem needs
- Goals and open questions are included and actionable
- Licensing and structural requirements are met

Editors coordinate final feedback and guide the author toward merge-readiness.

#### 3. Merging MPSs into the Repository

A problem statement may be merged into the official repository when:

- It is well-formulated, precise, and unambiguous
- It demonstrates a real, unmet need with practical use cases
- It has no existing, viable solutions available
- It has been acknowledged, when relevant, by affected project maintainers

In some cases, MPSs may document already-solved problems retroactively and may be merged with the status `Solved` to serve as historical context and rationale for existing MIPs.

MPSs deemed unclear, unrealistic, or already addressed may be:

- Rejected (pull request closed with justification)
- Withdrawn by the author
- Deferred until the author or another contributor revises and resubmits

MPSs abandoned without follow-up activity may also be rejected after a reasonable period of inactivity.

#### 4. Actors of the Ecosystem Design and Work on Possible Solutions

Once merged, MPSs become reference documents. Authors, developers, contributors, or maintainers may propose solutions to the problem in the form of new MIPs.

Accepted MIPs that solve or partially solve the problem should be added to the `Proposed Solutions` field in the MPS preamble.

If only part of the problem is resolved, the MPS should be updated to reflect the remaining open issues and keep the problem statement active for future solutions.

### Editors

The role and responsibilities of MIP editors are defined in the core MIP-0001: Midnight Improvement Proposal Process. Editors are responsible for:

- Maintaining the quality and structure of the MIPs/MPSs repository
- Facilitating reviews and triage
- Helping authors align with process expectations
- Assigning numbers to accepted MIPs and MPSs

## Rationale

### Goals

Goals make it easier to assess whether a solution solves a problem. Goals also give a direction for projects to follow and can help navigate the design space. The section is purposely flexible -- which we may want to make more rigid in the future if it is proven hard for authors to articulate their intents. Ideally, goals capture high-level requirements.

### Use Cases

Use cases are essential to understanding a problem and showing that a problem addresses a need. Without use cases, there is, in fact, no problem, and merely disliking a design doesn't make it problematic. A use case is also generally user-driven, which encourages the ecosystem to open a dialogue with users to build a system that is useful to others and not only well-designed for the mere satisfaction of engineers.

### Open questions

This section is meant to _save time_, especially for problem statement authors who will likely be the ones who end up reviewing proposed solutions. Open questions allow authors to state upfront elements they have already thought of and that any solution should consider in its design. Moreso, it is an opportunity to mention, for example, security considerations or common pitfalls that solutions should avoid.

## Path to Active

### Acceptance Criteria
- [] Review this proposal with existing actors of the ecosystem
- [] Formulate at least one problem statement following this process

### Implementation Plan 
- [] Confirm after repeated cycles of CPS submissions, reviews, and merges that the CPS process is both effective and accessible to the community.

## Backwards Compatibility Assessment
None

## Security Considerations
None

## Implementation
None

## Testing
None

## References (Optional)
None

## Acknowledgements 
None

## Copyright Waiver

This MPS is submitted under the terms of the Apache License, Version 2.0, in accordance with the [Midnight Foundation Contributor License Agreement](Link to CLA), which includes the assignment of copyright for all contributions to the Midnight Foundation.