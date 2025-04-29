# **Midnight Improvement Proposal (MIP) Process**

This document outlines the process for proposing, discussing, and implementing improvements to the Midnight Blockchain. This process is designed to be transparent, community-driven, and managed within GitHub in the Midnight Architecture Repo.


### **Goals of the MIP**



* **Transparency:** All proposed changes should be publicly visible and accessible.
* **Community Participation:** Encourage and facilitate community involvement in the design and decision-making process.
* **Technical Soundness:** Ensure that all changes are technically sound and well-specified.
* **Documentation:** Maintain a clear and comprehensive record of all changes to the Midnight Blockchain.
* **Decentralization:** Support the long-term decentralization of the Midnight Blockchain development.


### **Process Overview**

The Midnight Improvement Proposal (MIP) process consists of the following stages:



1. **Idea Formulation:** An idea for an improvement is conceived.
2. **MIP Drafting:** A formal MIP document is created.
3. **Submission:** The MIP is submitted as a Pull Request (PR) to the Midnight Architectured GitHub repository.
4. **Review and Discussion:** The community and MIP Editors review and discuss the MIP.
5. **Status Progression:** The MIP progresses through various statuses based on review and consensus.
6. **Implementation:** Accepted MIPs are implemented by development teams.
7. **Deployment:** Implemented changes are deployed in a Midnight network upgrade.


### **MIP Categories**

MIPs are categorized to help organize and manage the different types of proposals:



* **Core:** Changes to the core Midnight protocol, consensus mechanisms, virtual machine, or other fundamental aspects. These may require a network upgrade.
* **Standards:** Proposals for standards and conventions related to application development, smart contract interfaces, data formats, etc., within the Midnight ecosystem.
* **Networking:** Improvements to network communication, peer discovery, and other networking-related aspects.
* **Governance:** Proposals related to the governance of the Midnight blockchain, including decision-making processes, roles, and responsibilities.
* **Informational:** Provides general information, guidelines, or research related to the Midnight Blockchain but does not propose a specific change.


### **MIP Statuses**

MIPs can have the following statuses:



* **Idea:** The initial stage where an idea is being discussed before a formal MIP is drafted. (Primarily on Midnight forums, Discord, etc.)
* **Draft:** The MIP is under active development and being written.
* **Proposed:** The MIP has been formally submitted as a PR to the MIPs repository.
* **Review:** The MIP is undergoing community and editor review.
* **Last Call:** A final review period before a decision on acceptance.
* **Accepted:** The MIP is accepted and considered for implementation.
* **Implemented:** The MIP has been implemented in Midnight client software.
* **Active:** The MIP is part of the active Midnight network (for protocol changes) or is widely adopted in the ecosystem (for standards).
* **Superseded:** A newer MIP replaces this MIP.
* **Obsolete:** The MIP is no longer considered relevant.
* **Rejected:** The MIP was not accepted.
* **Withdrawn:** The author(s) have withdrawn the MIP.


#### **Submission**



* A MIP should be submitted as a Pull Request (PR) to the[ Midnight Improvements Proposals](https://github.com/midnightntwrk/midnight-improvement-proposals) (this) repository of the Midnight GitHub organization.
* The PR title should be: MIP-XXXX: [MIP Title] (where XXXX is the next available MIP number, assigned by a MIP Editor).


#### **Review and Discussion**



* Once a MIP is submitted, it enters a period of public review and discussion.
* Feedback is encouraged from all Midnight community members, including:
    * Developers
    * Node operators
    * Application developers
    * Token holders
    * Users
* Discussion should take place on the GitHub PR page.
* Workshop sessions to discuss and review proposals, coordinated by XXXXXX
* MIP Editors will:
    * Ensure the MIP follows the template and meets the minimum quality standards.
    * Facilitate discussion and ensure all concerns are addressed.
    * Assign a MIP number.
    * Track the MIP's progress.
* Technical experts may be consulted for specific aspects of the proposal.


#### **Implementation**



* Once a MIP is Accepted, it becomes a candidate for implementation.
* Engineering teams (Shielded Architects & Engineers, Community developers) will implement the changes specified in the MIP.
* Implementation may involve:
    * Changes to Midnight Architecture
    * Modifying the Midnight protocol
    * Updating client software
    * Developing new tools or libraries


### **Artifacts and Tools**



* **MIP Document:** The core artifact is the MIP document itself, written in Markdown and stored in the MIPs repository.
* **MIP Template:** A standardized Markdown template for creating MIPs (see below).
* **GitHub Repository:** The MIPs repository (in Midnight-Architecture) on GitHub will be used to:
    * Store all MIP documents.
    * Track the status of each MIP.
    * Facilitate discussion through PRs.
* **Discussion Forums:** The Midnight Governance Hub (future state) and Discord server should be used for broader discussions and early-stage idea sharing. 
* **Meeting Notes:** Notes from any meetings related to MIP discussions should be publicly available in this GitHub repository or linked from it.


### **MIP Editors**

The MIP process relies on a group of *MIP Editors* to manage the process, ensure quality, and facilitate community discussion.


#### **Role and Responsibilities**

MIP Editors are responsible for:



* Ensuring that MIPs adhere to the MIP template and meet the minimum quality standards (e.g., clarity, completeness, technical soundness).
* Assigning MIP numbers.
* Tracking the progress of MIPs.
* Facilitating community discussion and helping to resolve technical disagreements.
* Determining when a MIP has reached sufficient consensus to move to the Accepted status.
* Maintaining the MIPs repository.

### **Selection and Qualifications**


MIP Editors should be selected based on the following criteria:



* **Technical Expertise:** A strong understanding of blockchain technology and the Midnight Blockchain architecture.
* **Community Standing:** Respected and trusted members of the Midnight community.
* **Impartiality:** Ability to evaluate MIPs objectively and fairly.
* **Communication Skills:** Excellent written and verbal communication skills.
* **Availability:** Willingness to dedicate sufficient time to review and process MIPs.

The initial set of MIP Editors will be appointed by the Midnight Foundation and Shielded. A process for electing or appointing new editors should be established and documented in a Governance MIP.


### **MIP Template**

All MIP documents *must* adhere to the following Markdown template:
```
---
MIP: <MIP Number> (assigned by a MIP Editor)
Title: <MIP Title>
Authors: <Author Name> <Author Email/GitHub username>
Status: <Status> (Draft, Proposed, Accepted, Obsolete, Rejected, Final)
Category: <Category> (Core, Standards, Networking, Governance, Informational)
Created: <Date Created>
Requires: <List of other MIPS this MIP depends on>
Replaces:<List of a MIP that this one replaces>
---

## Abstract

A short (~200 word) description of the proposed improvement.

## Motivation

Why is this change needed? What problem does it solve? Clearly explain the use cases and the benefits for the Midnight ecosystem.

## Specification

Describe the proposed change in detail. This section should be technically precise and unambiguous. It should provide enough information to allow for implementation.

## Rationale

Explain the design decisions behind the proposed change. Why was this particular approach chosen? What alternatives were considered, and why were they rejected?

## Backwards Compatibility Assessment

Describe how the proposed change affects existing systems, applications, and users. Will it require a hard fork? Are there any compatibility issues? How will they be addressed?

## Security Considerations

Analyze the potential security implications of the proposed change. Are there any new attack vectors or vulnerabilities introduced? How will they be mitigated?

## Implementation

Describe how the proposed change will be implemented. Which parts/components of the Midnight need to be modified? What are the dependencies, if any?

## Testing

Describe the testing procedures for the proposed change. What tests will be performed to ensure that it works as expected and does not introduce any regressions?

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
```
