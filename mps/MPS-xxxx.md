---
MPS: xxxx
Title: Security-Review Evidence for Compact Contract Releases
Authors: Jiawen Li @Tracyli025
Status: Draft
Category: Standards
Created: 19-AUG-2026
Requires: none
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

Midnight lacks a standard way to bind an exact Compact release to the security-review statement covering it and to the verifier-key set observed at a contract address. Valid review evidence may therefore be applied to unreviewed code, later changes, or a deployment it no longer covers.

Midnight.js's verifyContractState can compare supplied verifier keys with current contract state, but it does not link them to source, builds, or reviews, and maintenance can change them at the same address. The compiler's documented outputs include no stable privacy-flow account for successful releases. For confidential source or reports, authorized parties can compare hidden material with its identifiers. The public can inspect only a bounded statement and verify its signature, not the hidden content; attributing the signing key still depends on a trust policy.

This MPS describes these evidence-binding gaps. It does not choose a package format, signature or identity system, registry, audit method, access policy, or storage mechanism.

## Vision

A developer, auditor, wallet, authorized reviewer, or deployment reviewer can start with a Compact release or contract address and determine:

- which source, dependencies, build inputs, compiled artifacts, and local witness implementations a release or review claim covers;
- how the release's privacy-relevant behavior differs from another release;
- which review statement applies, what it includes or excludes, and what remediation status it states;
- how the statement's issuer is attributed, which evidence the consumer can independently check, and which material is issuer-asserted, restricted, or unavailable; and
- whether the evidence matches the verifier-key set observed at the contract address and a stated state reference.

This does not prove that the contract is safe. It makes the reviewed subject, claim boundaries, trust assumptions, and point-in-time deployment applicability inspectable.

## Problem

### Review scope and release identity can diverge

A Compact release can span source, dependencies, build inputs and toolchain versions, privacy-flow evidence, generated code, ZKIR, proving and verifier material, and local witness implementations. A security review may cover only some of these components and may record exclusions, findings, and later remediation. A branch, release label, report filename, or contract address identifies only part of that subject and may remain unchanged while other components change.

Consumers must relate three distinct subjects:

```text
release subject
(source, build inputs, privacy-flow evidence, artifacts, and witnesses)
        <- covered by ->
review claim
(scope, exclusions, findings, and remediation)
        <- applied to ->
deployment observation
(contract address, state reference, and verifier-key set)
```

A shared project name or URL establishes none of these relationships. [`midnightzk-anchor` deployment review discussion](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/155) illustrates the gap: after a reviewer requested a contract change, the applicant reported updating external source and compiled artifacts before approval, leaving later consumers to correlate the review thread, repository, and deployment document.

### Privacy-relevant release changes lack a stable comparison surface

Compact requires a program to declare when potentially private data may reach public ledger state, an exported circuit return, or another contract. The compiler can report origins and paths when it rejects undeclared disclosure, but its documented successful-build outputs do not define a stable, machine-readable account of the accepted privacy flows.

Reviewers and CI therefore reconstruct changes from source. Neither file diffs nor counts of `disclose()` calls reveal changes in value or control dependencies or their actual public sinks. [Compact issue #706](https://github.com/LFDT-Minokawa/compact/issues/706) illustrates this limitation: it reports that `ownPublicKey()` could reach public ledger state without `disclose()`, while a comparable custom witness triggered a compiler error.

### Deployment verification is partial and time-bound

Midnight.js can compare locally supplied verifier keys with current ContractState, but that check does not identify the release or review from which they came. Maintenance can insert or remove keys or replace the authority at the same address. Applicability must therefore identify the checked keys and observed state or time; later maintenance can make the conclusion stale.

### Security-review statements lack common semantics

Security-review statements differ in how they identify the reviewed release, scope, exclusions, unresolved findings, remediation, and later status. There is no common way to determine whether a statement still applies or has been superseded, withdrawn, expired, or revoked. A signature verifies bytes under a key, not auditor attribution, authority, or claim meaning. 

When sources or reports are confidential, authorized reviewers may compare the hidden material with identifiers in the statement, while public consumers can inspect the bounded statement, verify its signature, and see which information is withheld. A digest can identify hidden material for authorized comparison, but it does not let the public verify its contents.

## Related Midnight work

MPS-0002 covers developer tooling and understandable privacy flows but does not define a stable release-comparison artifact. MPS-0022 anticipates a language-agnostic compiled-contract representation but does not bind a complete release to review and deployment evidence. MPS-0004 addresses delegated proof-server attestation rather than a reviewed contract release. The Contract Deployment Rubric collects evidence for deployment authorization and distinguishes that process from an audit. Future work can reuse these efforts without depending on them.

## Use Cases

### UC1: Privacy-relevant release comparison

A helper change lets a new private input influence an existing public ledger field while the existing downstream `disclose()` call remains unchanged and the build still succeeds. A reviewer or CI system needs to identify the changed origin-to-sink relationship and flag it for review, without exposing witness values or treating every generated-file change as a privacy change.

### UC2: Exact audit and remediation scope

An auditor reviews a release and later confirms remediation of a finding. After the repository and toolchain advance, a consumer needs to distinguish the reviewed remediation release from the earlier affected release and from later unreviewed changes.

### UC3: Maintained deployment applicability

A wallet previously matched released verifier keys to a contract address. A maintenance operation later changes one circuit's key. The wallet needs to detect that the earlier point-in-time audit-applicability conclusion is stale.

### UC4: Confidential source or report

An enterprise cannot publish its source, full report, or detailed findings. Its auditor issues a bounded public statement for the reviewed release. After the developer grants access, an authorized regulator or reviewer can compare the underlying material; ordinary users see the published scope, remediation claims, limitations, and withheld details without being told that hidden evidence was independently verified.

### UC5: Bounded wallet and deployment-review claims

A DApp links to source and a signed report. A wallet or deployment reviewer needs to distinguish artifact identity, audit scope, signer attribution, deployment applicability, and unknowns rather than collapse them into a "safe" badge. The same subject evidence should be reusable without treating deployment authorization as an audit.

## Goals

1. Identify the exact source, dependencies, build inputs, privacy-flow information, compiled artifacts, local witness implementations, review materials, and deployment observation covered by a release or review claim.
2. Let tools detect security- and disclosure-relevant changes between releases without requiring witness values or confidential materials to be made public.
3. Keep artifact identity, build reproducibility, audit scope, remediation, signer attribution, deployment authorization, and deployment applicability as separate claims. 
4. Make replacement, supersession, withdrawal, expiry, and revocation detectable without silently substituting one subject or statement for another.
5. Determine whether review evidence applies to the verifier-key set observed at a contract address and state reference, and make later staleness detectable.
6. Support bounded claims over confidential subjects so that authorized consumers can verify underlying material while public consumers can distinguish independently verifiable evidence, issuer assertions, unavailable fields, and explicit exclusions.

## Expected Outcomes

- Reviewers focus on changed privacy flows and release components instead of reconstructing unchanged evidence.
- Audit and remediation statements remain tied to the exact subject they cover as repositories, releases, and deployments evolve.
- Wallets and deployment reviewers can report a match, mismatch, stale observation, or insufficient evidence without presenting those results as proof that the contract is safe.
- Confidential projects can make limited public claims while authorized parties verify the underlying source or report.
- Release, audit, and deployment workflows reuse evidence while preserving their different authorities and conclusions.

## Open Questions

1. Which minimum release components, identifiers, and build inputs are necessary for each claim type, and when can source-to-artifact reproducibility be meaningfully claimed?
2. What minimum privacy-flow facts should be comparable across releases, and how should tools handle compiler-version changes or ambiguous matching?
3. Which subject identifiers should the two downstream work areas share, and where should their responsibility boundary fall?
4. What minimum scope, exclusions, findings, remediation, completeness, limitations, and withheld-information fields must a security-review statement express?
5. Which trust and discovery models should consumers support for signer attribution, replacement, withdrawal, expiry, revocation, and conflicting statements?
6. What state reference is sufficient for a reproducible deployment-applicability check across maintenance, finality, and reorganization assumptions?
7. What minimum messages and states are needed to distinguish a review request, accepted scope, remediation submission, completed review, supersession, withdrawal, and rejection?

## Recommended MIPs

### Privacy-flow observability and release comparison

A downstream Standards MIP should determine interoperable semantics for a compiler-supported account of privacy-relevant flows in successful Compact releases. It should enable tools to distinguish origins, permission sites, transformations, actual public sinks, and relevant release changes without exposing witness values. The MIP must decide its representation, identifiers, versioning, completeness, and comparison behavior rather than inherit a predetermined schema from this MPS.

### Release Publication and Security-Review Protocol

A downstream Standards MIP should determine an interoperable protocol covering three related stages:

1. Release publication, through which an App Provider identifies the exact Compact release subject and makes its public or access-restricted components discoverable, including any applicable privacy-flow evidence.
2. Review request and response, through which the App Provider and Security Auditor bind an accepted review scope, exclusions, release identifiers, confidential-material access conditions, and remediation rounds.
3. Review-result publication, through which the Security Auditor issues a signed, lifecycle-aware statement for the reviewed subject and, where claimed, its point-in-time deployment applicability.

The protocol should support bounded public statements and authorized verification of confidential sources or reports. It should not prescribe an audit method, require public Git repositories or a mandatory registry, or treat a signature, matching digest, or review status as proof that a contract is safe.

## References

- [MPS-0001: Midnight Problem Statement Process](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0001-mps-process.md)
- [MPS-0002: Midnight Developer Tooling](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0002-developer-tooling.md)
- [MPS-0022: A Standard, Language-Agnostic Representation of Compiled Compact Contracts](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0022-standard-contract-representation.md)
- [MPS-0004: Trustworthy Delegated Proof Generation for Privacy-Preserving Transactions](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0004-trusted-proof-serving.md)
- [Midnight Contract Deployment Rubric](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/deployments/contract-deployment-rubric.md)
- [Explicit disclosure in Compact](https://docs.midnight.network/compact/reference/explicit-disclosure)
- [Compact compiler usage](https://docs.midnight.network/compact/compilation-and-tooling/compiler-usage)
- [`ContractState` API](https://docs.midnight.network/api-reference/ledger/classes/ContractState)
- [`verifyContractState` API](https://docs.midnight.network/api-reference/midnight-js/@midnight-ntwrk/midnight-js-contracts/functions/verifyContractState)
- [`submitInsertVerifierKeyTx` API](https://docs.midnight.network/api-reference/midnight-js/@midnight-ntwrk/midnight-js-contracts/functions/submitInsertVerifierKeyTx)
- [`submitRemoveVerifierKeyTx` API](https://docs.midnight.network/api-reference/midnight-js/@midnight-ntwrk/midnight-js-contracts/functions/submitRemoveVerifierKeyTx)
- [`midnightzk-anchor` deployment review](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/155)

## Acknowledgements

To be filled in when reviewers and co-authors are confirmed.

## Copyright

This MPS is licensed under CC-BY-4.0.
