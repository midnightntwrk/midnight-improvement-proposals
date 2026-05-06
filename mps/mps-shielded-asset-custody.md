---
MPS: "?"
Title: Custodian Support for Native Shielded Assets (ZSwap)
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

Regulated custodians, exchanges, banks, and institutional wallet providers require custody architectures that enforce strict threshold-control models, typically through MPC, HSMs, TSS, or equivalent secure signing systems. Midnight shielded assets introduce a different custody challenge because native shielded ownership, address generation, visibility, and spending are not expressed purely as standard signature workflows. In particular, shielded address generation currently depends on hashing secret material, while shielded spending requires proof generation over private coin material. These requirements create friction for custodians whose security model prevents any single party, device, service, or proof server from reconstructing or accessing full private material.

There is also a compliance and source-of-funds challenge. When a recipient receives a shielded asset, the current model does not provide a reliable way to determine with certainty who sent the asset or where it originated. Viewing keys may help identify assets belonging to a wallet, but they do not provide sender attribution or sufficient source-of-funds information for most regulated custody workflows. This creates a significant barrier for custodians that must satisfy KYC, AML, transaction monitoring, audit, and reconciliation obligations.

This MPS defines the problem space so future MIPs can propose custodian-compatible shielded key generation, viewing, proof generation, spending, and compliance-support mechanisms.

## Vision

Midnight should support regulated custodians, exchanges, banks, and institutional wallet providers without requiring them to weaken their security model or us to abandon the privacy guarantees of shielded assets. A custodian should be able to create shielded accounts, receive shielded assets, report balances, identify assets under custody, support audit and compliance workflows, and authorize shielded spends using a threshold custody model such as 2-of-3 MPC or equivalent.

The ideal future state is one where native shielded asset custody feels operationally familiar to custodians while preserving Midnight's privacy model. Custodians should not need to rely on temporary smart-contract workarounds, visible asset-holding patterns, or bespoke off-chain state systems merely to support basic custody. They should also have enough visibility to satisfy regulated custody obligations without receiving unnecessary access to customer spending authority or compromising network-level privacy.

## Problem

### 1. DKG and shielded address generation do not fit cleanly into MPC custody

The first friction point appears before any transaction occurs: wallet creation. Regulated custodians commonly use distributed key generation and threshold key material so that no single participant holds or reconstructs a complete private key. Midnight shielded address generation requires applying SHA-256 directly to secret material. That creates a mismatch because a custodian cannot simply gather the full secret and hash it without violating the assumptions of its MPC architecture. Even if threshold signing is available, the address and key derivation model itself is not share-friendly.

### 2. Custodian custody is based on threshold control, not single-secret control

Institutional custody is normally built around threshold authorization, policy controls, secure key isolation, and recovery mechanisms. This is materially different from a wallet model where a single user locally controls a full secret. Midnight shielded custody therefore needs to support threshold-controlled ownership as a first-class requirement rather than treating MPC or HSM integration as an external implementation detail. Without this, custodians may be forced into designs that either weaken their security model or avoid native shielded custody entirely.

### 3. Viewing keys do not reveal the source of a shielded asset

When a person or institution receives a shielded asset, they may be able to identify that a note or asset belongs to them, but they do not necessarily know with certainty who sent it or where it originated. Viewing keys can help a wallet or service identify assets that belong to an account and support balance reporting, but they do not provide a reliable source-of-funds trail. A viewing key may show that a shielded asset was received, but not definitively identify the sender or provide a KYC-able address history. This is a major issue for regulated custodians because custody is not only about controlling assets; it is also about understanding whether assets can be accepted, reported, monitored, and reconciled under compliance obligations. If a custodian cannot reliably determine source of funds, sender identity, or provenance, then accepting shielded deposits creates KYC, AML, sanctions-screening, and transaction-monitoring risk.

### 4. Native shielded spending requires proof generation over private coin material

For native shielded assets, the hard problem is not simply receiving assets; it is spending, burning, or transferring them later. Moving shielded assets out of a wallet-style account requires generating a zero-knowledge proof, and the current understanding is that this proof path requires access to private coin material. This does not map to a standard custody flow where two parties sign a transaction and the transaction is broadcast. It introduces a separate proof-generation step that must be made compatible with custody-grade key isolation.

### 5. Proof servers conflict with HSM and MPC security boundaries

Custodians generally do not allow private keys, key-equivalent material, or sensitive signing material to be exposed outside approved secure boundaries. A conventional proof server that receives full coin secrets, spend keys, or equivalent private material may therefore be treated as a hot-key risk. This creates a fundamental mismatch between native shielded proof generation and regulated custody infrastructure. A custodian-safe proof model must ensure that proof generation does not require unsafe movement or reconstruction of sensitive material.

### 6. Key shards are themselves sensitive material

Collaborative proof generation, coSNARKs, or proof-share approaches may be promising, but they do not automatically solve the custodian problem. Key shards are also sensitive signing material. If a proof server requires access to a key shard, even without reconstructing the full private key, that may still be unacceptable under a custodian's risk model. Any future solution must therefore avoid not only full private-key reconstruction, but also unsafe exposure of key-share material.

### 7. Contract-based mint/burn flows are useful but incomplete

A constrained smart-contract approach can support early use cases such as tokenized deposits where assets are minted, held, and burned within a narrow operational boundary. This may be sufficient for a simple V1 product where assets do not move freely. However, this is a workaround rather than full native shielded custody. It does not support the broader vision where users or institutions can hold shielded assets in wallet-style accounts and later move them into transfers, redemptions, DeFi flows, or other applications.

### 8. Custodians must store private coin information to support future spends

A shielded asset is not spent from a simple publicly readable account balance. To spend a shielded asset, the spender must know the relevant private coin information for the specific asset being consumed. Public ledger data may show commitments and state transitions, but it does not by itself give the custodian all of the information needed to identify, reconstruct, select, and spend a customer’s shielded coins. This means a custodian must capture and securely store the required coin information when assets are received, then preserve it accurately for future transfers, burns, redemptions, or recovery workflows. If this information is missing, stale, or incorrectly associated with a customer account, the custodian may be unable to spend the asset even if it has the required signing authority.

## Use Cases

### Use Case 1: Regulated custodian creates a shielded wallet for a customer

A regulated custodian wants to create a Midnight shielded wallet for a customer using its standard threshold custody architecture. Today, this is difficult because shielded address generation appears to require hashing secret material directly, while the custodian's DKG process is designed to avoid creating a single full secret. The custodian needs a way to derive shielded addresses without reconstructing or exposing the secret.

### Use Case 2: Custodian receives shielded assets on behalf of a customer

A custodian wants to receive shielded assets into a customer account and report the resulting balance. The custodian needs enough viewing or indexing capability to detect incoming assets, associate them with the correct customer, and maintain reliable accounting. However, receiving the asset does not necessarily tell the custodian with certainty who sent it or where it came from, which creates a source-of-funds and compliance challenge.

### Use Case 3: Custodian evaluates whether an incoming shielded asset can be accepted

A custodian receives a shielded deposit and must decide whether it can accept the asset under KYC, AML, sanctions, and internal risk policies. If the system only reveals that the asset belongs to the recipient but does not provide reliable sender attribution or provenance, the custodian may be unable to satisfy its compliance obligations. The result may be that the custodian refuses to support shielded deposits, supports only restricted flows, or requires assets to pass through controlled mint/burn processes.

### Use Case 4: Customer redeems or transfers a shielded asset

A customer or regulated institution wants to redeem, burn, transfer, or otherwise move a shielded asset. The custodian must authorize the action using its threshold custody model, but the asset movement also requires a shielded proof. Today, that proof path requires private coin material that the custodian cannot easily expose to a proof server.

### Use Case 5: Institution enables shielded assets to participate in broader DeFi workflows

An institution wants to support shielded assets in a way that goes beyond simple custody or static holding. To unlock the full range of DeFi capabilities, shielded assets must eventually be able to move between wallet-style accounts, smart contracts, liquidity venues, lending markets, structured products, and redemption flows without losing their privacy properties. This requires native shielded custody that can support controlled movement, proof generation, balance visibility, compliance checks, and secure authorization. If shielded assets can only be held in constrained patterns or moved through bespoke workflows, custodians and institutions will struggle to support the broader application layer that Midnight is designed to enable.

### Use Case 6: Exchange or custodian supports many shielded asset accounts

An exchange or institutional custodian may manage thousands of customer accounts. It needs scalable viewing, balance reporting, audit trails, and operational controls. Delegated viewing rights, indexer support, and efficient scanning are therefore essential. Without them, every account may require bespoke handling, large-scale scanning, or operationally fragile off-chain state management.

## Goals

1. **Support threshold custody for shielded assets.**
   Any future solution should allow custodians to operate under 2-of-3 or equivalent threshold-control models without reconstructing full private keys or exposing key-equivalent material.

2. **Enable custodian-safe shielded address generation.**
   Shielded address derivation should be compatible with DKG/MPC architectures, or an alternative derivation path should be provided for institutional custody.

3. **Provide reliable source-of-funds support for regulated flows.**
   Custodians should have a way to determine, prove, or obtain sufficient information about the origin of shielded assets they receive, without undermining Midnight's broader privacy model.

4. **Separate authorization from proof generation where possible.**
   Custodians should be able to authorize a shielded spend using standard threshold signing or equivalent controls, while proof generation should not require unsafe exposure of private key material.

5. **Provide a secure delegated proof-generation model.**
   If proof servers remain necessary, there should be a custody-compatible model for running them in a secure environment without exposing full coin secrets, spend keys, or sensitive key shards outside an approved security boundary.

6. **Support delegated viewing and institutional reporting.**
   Custodians should be able to delegate or centralize viewing rights for operational reporting, audit, and compliance, without delegating spending authority.

7. **Minimize bespoke off-chain state burden.**
   Custodians should not be forced to maintain fragile, product-specific private state unless that state-management model is clearly specified, testable, and supported by robust indexer APIs.

8. **Preserve Midnight's privacy guarantees.**
   Solutions should not require assets to be held in visible contract state or otherwise degrade shielded privacy merely to make custody possible.

## Expected Outcomes

If this problem is addressed, Midnight will become significantly easier for regulated custodians, exchanges, banks, and institutional wallet providers to support. Partners will be able to custody shielded assets without bespoke one-off architecture, users will be able to access shielded asset products through trusted custodians, and builders will have clearer integration patterns for tokenized deposits, RWAs, private payments, and future DeFi products.

The most important outcome is that Midnight's shielded asset model becomes institutionally usable without sacrificing either privacy or custody-grade security. This would reduce reliance on temporary contract-based workarounds and make shielded assets more composable across wallets, custodians, and applications.

## Open Questions

1. Can Midnight support shielded address generation in a way that is compatible with DKG/MPC, without requiring any party to reconstruct or hash the full secret?

2. Can shielded spending be redesigned so that threshold signatures authorize spends while proof generation uses separate, non-key-equivalent material?

3. What information should a shielded recipient be able to learn about the source of received assets, and under what permissioning model?

4. Can source-of-funds information be disclosed selectively to regulated custodians without making all shielded transfers publicly traceable?

5. Is a coSNARK or collaborative proof-generation model practical for Midnight's current proving stack, and would custodians accept the treatment of key shards or proof shares under that model?

6. Can proof generation be safely run inside TEEs, HSM-adjacent infrastructure, or custodian-controlled secure environments without exposing sensitive material?

7. Should Midnight introduce first-class delegated viewing rights for institutional custody, and if so, should delegation be protocol-level, wallet-level, or indexer-level?

8. Can viewing keys be extended or complemented to support compliance-grade provenance without compromising sender privacy for ordinary users?

9. Can contracts be given a privacy-preserving analogue to viewing capability, or should contract-based custody avoid viewing keys entirely?

10. What minimum indexer APIs are required for custodians to track shielded UTXO state safely and reliably?

11. Which parts of the problem are protocol changes, which are wallet SDK changes, which are indexer changes, and which are application-level patterns?

## Recommended MIPs

### MIP: Custodian-Compatible Shielded Address Derivation

This MIP should define a shielded address derivation method that works with DKG/MPC custody architectures. It should address the current friction caused by deriving shielded addresses from hashed secret material and define acceptance criteria for custodians that cannot reconstruct a full secret.

### MIP: Threshold Authorization for Shielded Spending

This MIP should explore whether shielded spending can be authorized through threshold signatures or equivalent controls, while keeping proof generation separate from the spending authority itself. The objective would be to make shielded spends fit more naturally into regulated custody workflows.

### MIP: Secure Delegated Proof Generation

This MIP should specify a custody-safe proof-generation architecture. Candidate approaches may include TEEs, HSM-adjacent proof services, collaborative proof generation, coSNARKs, or proof-share aggregation. The MIP should explicitly analyze whether key shards, proof shares, or coin secrets are exposed and whether the model is acceptable for institutional custody.

### MIP: Delegated Viewing and Institutional Reporting

This MIP should define how an account, wallet, or institution can delegate viewing rights to an operational key or service without delegating spending authority. It should cover balance reporting, audit, account-level visibility, and revocation.

### MIP: Selective Source-of-Funds Disclosure

This MIP should define a privacy-preserving mechanism by which the origin, sender, or provenance of a received shielded asset can be disclosed to an authorized party when required. The goal should be to support regulated custody and compliance workflows without making shielded transfers publicly traceable.

### MIP: Custodian-Oriented Shielded Indexing Requirements

This MIP should specify the indexer and API capabilities required for custodians to track shielded assets reliably. It should define how custodians discover relevant notes/UTXOs, maintain Merkle tree positions, reconcile balances, and recover from missed events or infrastructure downtime.

## References

- Midnight MPS template: `mps/mps-template.md`
- Midnight MIP template: `mips/mip-template.md`
- MIP-0001: Midnight Improvement Proposal Process

## Acknowledgements

This problem statement is based on discussions between Midnight Foundation contributors, custody providers, smart-contract security contributors, and related stakeholders exploring custody support for shielded assets, tokenized deposits, MPC-based custody, proof-server architecture, viewing keys, source-of-funds requirements, and institutional wallet operations.

## Copyright

This MPS is licensed under CC-BY-4.0.
