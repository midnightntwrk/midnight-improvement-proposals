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
MPS: 0006  
Title: Institutional Custody for Native Shielded Assets (ZSwap)  
Category: Core  
Status: Proposed  
Authors:
  - Jalal Hannan <Jalal-1>
  - Hector Bulgarini <hbulgarini>  
Proposed Solutions: []  
Discussions: []  
Created: 2026-05-06  
License: CC-BY-4.0  
---

## Abstract

Native shielded asset custody on Midnight is not a conventional custody integration problem. It is not only a question of whether a custodian can hold or control a key. It is a full-stack custody problem involving shielded account generation, note discovery, viewing capability, private UTXO state management, proof generation, DUST management, compliance visibility, and transaction execution.

The central mismatch is that native shielded asset movement in ZSwap is proof-authorized rather than purely signature-authorized. In an account-based system, a custodian can usually approve asset movement by applying its existing threshold signing, MPC, HSM, TSS, or equivalent controls. In ZSwap, spending a shielded asset requires constructing a shielded UTXO transaction and generating a zero-knowledge proof over private witness material. Any service that handles that material becomes part of the custody security boundary and may be treated by custodians as handling key-equivalent data.

This creates integration challenges for regulated custodians, exchanges, banks, and institutional wallet providers. They need to support threshold-controlled custody, balance reporting, auditability, transaction execution, source-of-funds controls, and recovery workflows without weakening Midnight's privacy model or exposing sensitive material outside approved secure environments.

This MPS defines the problem space for native shielded asset custody so that future MIPs and implementation work can address it with protocol, SDK, indexer, wallet, and infrastructure changes.

## Problem Statement

Midnight's current shielded asset model does not yet provide a custody pattern that is operationally familiar to regulated institutional custodians. Existing institutional custody systems are built around threshold authorization and secure key isolation. Native shielded asset movement requires proof generation over private note or coin data, which introduces sensitive material and operational state that custodians do not normally manage in standard signing-based workflows.

As a result, custodians may be forced into constrained workarounds, bespoke off-chain state systems, or product-specific flows that support narrow use cases but do not provide general native shielded custody.

## Background

Institutional custody normally assumes that asset movement is authorized by signatures. The custodian can enforce policy using MPC, HSMs, TSS, multisig, approval workflows, recovery keys, or other threshold-control mechanisms. The transaction may be constructed outside the secure boundary, while the signing material remains protected inside it.

ZSwap uses a different model. Shielded assets are represented as private notes or UTXOs. To move a shielded asset, the spender must know the relevant private coin information, construct the correct inputs and outputs, and produce a zero-knowledge proof that validates the spend without revealing the private data. This means that custody must account not only for keys, but also for note discovery, private state retention, proof generation, DUST funding, and selective visibility.

## Core Problem Areas

### 1. Shielded account generation is not obviously threshold-native

Regulated custodians commonly use DKG, MPC, HSMs, or TSS so that no single party or service reconstructs a full secret. Shielded address or account derivation must therefore be compatible with threshold custody. If address derivation requires direct operations over full secret material, custodians may be unable to create shielded accounts without violating their security model.

### 2. Custody control is threshold-based, while shielded ownership is secret-based

Institutional custody is built around threshold authorization, policy enforcement, secure key isolation, and recovery. Native shielded ownership and spending must support this model as a first-class requirement. Otherwise, custodians must either weaken their controls or avoid native shielded custody.

### 3. Viewing keys support balance visibility, not full provenance

A viewing capability may allow a wallet or custodian to identify shielded assets belonging to an account and report balances. It does not, by itself, provide reliable sender attribution, source-of-funds history, or KYC-able provenance. Ledger-level privacy is a feature of Midnight, but regulated custodians still need a compliant way to evaluate whether received assets can be accepted, monitored, reconciled, and audited.

### 4. Spending requires proof generation over private witness material

Receiving a shielded asset is only part of the custody problem. The custodian must also be able to transfer, burn, redeem, or otherwise spend the asset later. That requires proof generation over private note or coin data. This does not fit the standard pattern where a transaction is prepared and then approved only by threshold signing.

### 5. Proof generation becomes part of the custody boundary

A proof server or proving service that handles private coin information, spend-authorizing material, or key-equivalent witness data may be treated as a hot-key or sensitive-signing-material risk. A custodian-safe proving model must avoid unsafe reconstruction, movement, or exposure of full secrets and must clearly define which data enters which secure environment.

### 6. Key shares and proof shares may still be sensitive

Collaborative proving, coSNARKs, proof-share aggregation, or TEE-based approaches may help, but they do not automatically solve the custody problem. Key shares, secret shares, and some witness components may still be sensitive. Any proposed model must state precisely what material is exposed, where it is exposed, whether it is key-equivalent, and how it is protected.

### 7. Custodians must manage private coin state

A shielded asset is not spent from a publicly readable account balance. Future spends require the custodian to know and preserve the correct private coin information, including enough state to identify, select, prove, and spend the relevant notes. If this state is missing, stale, or incorrectly associated with a customer account, the custodian may be unable to move the asset even if it has the required authorization rights.

### 8. DUST management is part of custody execution

Transaction execution on Midnight requires DUST. Custodians therefore need a custody-compatible way to fund, sponsor, account for, and monitor DUST usage. This is not only a fee abstraction; it affects transaction construction, wallet operations, proof generation, and the risk of stuck assets.

### 9. Contract-based mint/burn patterns are useful but incomplete

Constrained contract-based flows can support early products where shielded assets are minted, held, and burned inside a narrow operational boundary. These flows may be valuable for initial deployments. However, they are not a substitute for general native shielded custody, where assets can be held in wallet-style accounts and later transferred, redeemed, or used in broader application flows while preserving privacy.

## Goals

1. **Support threshold custody for shielded assets.** Custodians should be able to operate under 2-of-3 or equivalent threshold-control models without reconstructing full secrets or exposing key-equivalent material outside approved boundaries.

2. **Enable custodian-safe shielded account generation.** Shielded address and account derivation should be compatible with DKG, MPC, HSM, TSS, or equivalent institutional custody architectures.

3. **Make proof generation custody-safe.** Any proving model should clearly define what private material is required, where it is processed, and how it remains protected.

4. **Support note discovery and balance reporting.** Custodians should be able to identify assets under custody, report balances, reconcile accounts, and recover from missed events or infrastructure downtime.

5. **Support delegated visibility without delegated spending.** Custodians should be able to centralize or delegate viewing for operations, audit, and reporting without granting spending authority.

6. **Support source-of-funds and compliance workflows.** The ecosystem should provide a privacy-preserving way for regulated entities to obtain sufficient provenance, sender, or origin information where required.

7. **Support DUST-aware transaction execution.** Custody integrations should include clear patterns for DUST funding, sponsorship, accounting, and operational monitoring.

8. **Preserve Midnight's privacy guarantees.** Custody support should not require public exposure of shielded balances, sender/recipient relationships, or private transaction data merely to make institutional custody possible.

9. **Reduce bespoke off-chain state burden.** Required private state management should be clearly specified, testable, recoverable, and supported by robust SDK and indexer APIs.

## Non-Goals

This MPS does not prescribe a specific implementation, proving architecture, custody vendor model, or compliance policy. It does not propose to weaken shielded privacy or make all shielded transfers publicly traceable. It also does not require every shielded asset flow to support open third-party deposits on day one.

The purpose of this MPS is to define the problem clearly enough that future MIPs and implementation work can evaluate competing solutions.

## Representative Use Cases

### Use Case 1: Custodian creates a shielded account

A regulated custodian wants to create a shielded account for a customer using its standard threshold custody architecture. The custodian needs to derive and manage the account without reconstructing or exposing full secret material.

### Use Case 2: Custodian receives and reports shielded assets

A custodian receives shielded assets on behalf of a customer and must detect the assets, associate them with the correct account, report balances, and reconcile holdings.

### Use Case 3: Custodian evaluates whether assets can be accepted

A custodian receives a shielded asset and must determine whether it can be accepted under KYC, AML, sanctions, audit, and internal risk policies. Ledger-level receipt alone may not provide enough provenance or sender attribution.

### Use Case 4: Customer redeems, burns, or transfers a shielded asset

A customer or institution wants to redeem, burn, transfer, or otherwise move a shielded asset. The custodian must authorize the action under its threshold-control model and generate the required shielded proof without unsafe exposure of private material.

### Use Case 5: Institution supports broader shielded asset workflows

An institution wants shielded assets to move beyond static holding into payments, redemptions, tokenized deposits, RWAs, lending, liquidity, or other application flows. This requires native shielded custody that supports controlled movement, proof generation, compliance visibility, and secure execution.

### Use Case 6: Custodian supports many shielded accounts

An exchange or institutional custodian may manage thousands of customer accounts. It needs scalable viewing, indexing, state recovery, audit trails, and operational controls rather than bespoke handling per account.

## Open Questions

1. Can shielded address derivation be made compatible with DKG, MPC, HSM, or TSS custody models without reconstructing or hashing a full secret in one place?

2. Can shielded spending separate threshold authorization from proof generation, so that proof generation does not require exposure of key-equivalent material?

3. What exact private material is required for proof generation, and which parts are key-equivalent under a custodian security model?

4. Can proof generation be performed safely through TEEs, HSM-adjacent services, collaborative proving, coSNARKs, proof-share aggregation, or another custody-compatible model?

5. What should a viewing capability reveal to custodians, and how can it support reporting without granting spending authority?

6. What privacy-preserving mechanism can provide regulated entities with sufficient source-of-funds or sender-origin information when required?

7. What indexer and SDK APIs are required for reliable note discovery, Merkle-position tracking, balance reconciliation, and recovery from missed events?

8. How should DUST funding, sponsorship, capacity planning, and accounting work for institutional custody flows?

9. Which early contract-based flows are acceptable as temporary patterns, and where do they fall short of native shielded custody?

10. Which aspects of the problem require protocol changes, and which can be solved through SDKs, wallets, indexers, custody infrastructure, or application patterns?

## Future MIP Areas

Future MIPs may address the following areas:

- Custodian-compatible shielded address derivation.
- Threshold authorization for shielded spending.
- Secure delegated or collaborative proof generation.
- Delegated viewing and institutional reporting.
- Selective source-of-funds disclosure.
- Custodian-oriented shielded indexing and state recovery.
- DUST sponsorship and transaction execution for institutional custody.

## Expected Outcome

If this problem is addressed, Midnight will be easier for regulated custodians, exchanges, banks, and institutional wallet providers to support. Institutions will be able to custody native shielded assets without relying on one-off workarounds, users will be able to access private asset products through trusted service providers, and builders will have clearer integration patterns for tokenized deposits, RWAs, private payments, and future application-layer workflows.

The desired outcome is that native shielded asset custody becomes institutionally usable without sacrificing Midnight's privacy guarantees or custody-grade security.

## References

- Midnight MPS template: `mps/mps-template.md`
- Midnight MIP template: `mips/mip-template.md`
- MIP-0001: Midnight Improvement Proposal Process

## Acknowledgements

This problem statement is based on technical discussions with custodians, wallet providers, protocol contributors, smart-contract security contributors, and other stakeholders exploring native shielded asset custody, threshold-control architectures, proof generation, viewing, DUST management, source-of-funds requirements, and institutional wallet operations.

## Copyright

This MPS is licensed under CC-BY-4.0.
