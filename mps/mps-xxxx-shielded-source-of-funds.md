---
MPS: xxxx
Title: Source-of-Funds Evidence for Incoming Shielded Assets
Authors:
  - Jalal Hannan <Jalal-1>
  - Hector Bulgarini <hbulgarini>
Status: Proposed
Category: Core
Created: 02-Jun-2026
Requires: none
Replaces: none
---

## Abstract

Incoming ZSwap shielded assets create a source-of-funds blocker for regulated custodians because receipt of an asset is not the same as acceptance of an asset.

With transparent assets, a custodian can often begin an inbound compliance review using public transaction data such as sender address, transaction history, contract path, exchange origin, or other visible chain metadata. ZSwap intentionally does not expose this information by default. A custodian may be able to detect that a shielded asset arrived, but the public ledger does not reveal a plaintext sender, originator, transfer path, or `msg.sender`-style source that can be used for source-of-funds review.

This creates a custody blocker: a regulated custodian may receive a shielded asset but lack sufficient evidence to decide whether it can be credited, accepted, redeemed, transferred onward, quarantined, or rejected.

This MPS defines the narrow problem of providing privacy-preserving source-of-funds evidence for incoming ZSwap shielded assets without turning Midnight’s shielded transaction model into a publicly traceable system.

## Problem Statement

Regulated custodians need a way to make defensible acceptance decisions for incoming ZSwap shielded assets.

A custodian may be able to detect that a shielded asset has arrived in a wallet it controls or monitors. That proves receipt. It does not, by itself, prove provenance. It does not reliably identify the sender, originating institution, customer, transfer instruction, issuer context, or compliance status of the funds.

For regulated custody, this creates an acceptance gap. Before an incoming asset can be credited to a customer, redeemed, transferred onward, or otherwise made available, the custodian may need evidence that the source of funds is acceptable under its AML, sanctions, travel-rule, audit, reconciliation, and internal risk controls.

ZSwap’s privacy model does not provide this evidence through public ledger data. That is intentional and should be preserved. The problem is not that all shielded transfers should become transparent. The problem is that regulated custodians need a privacy-preserving way to obtain and retain sufficient source-of-funds evidence where their operating model requires it.

The core question is:

**How can a custodian obtain sufficient source-of-funds evidence for an incoming ZSwap shielded asset without making all shielded transfers publicly traceable?**

## Technical Background

ZSwap shielded transfers are designed to protect transaction privacy. The ledger can validate that a shielded transfer is well formed without publicly revealing the sender, recipient, amount, or broader transaction graph.

This privacy model creates a distinction that does not exist in the same way for transparent account-based assets:

* **Receipt:** the custodian can detect that an asset arrived.
* **Attribution:** the custodian can identify or evidence where the asset came from.
* **Acceptance:** the custodian can decide whether the asset can be credited, held, redeemed, transferred onward, quarantined, or rejected.

Viewing capability may help a custodian detect received assets and report balances. It does not automatically provide source-of-funds attribution. A custodian knowing that it received a shielded asset is not the same as knowing whether the asset came from an acceptable sender, institution, customer, route, or transaction context.

The blocker is therefore not balance visibility. The blocker is privacy-preserving provenance evidence for inbound acceptance decisions.

## Core Requirements

A source-of-funds evidence model for incoming shielded assets should preserve the following properties:

1. A receiving custodian can obtain sufficient provenance evidence for an incoming ZSwap shielded asset when required.
2. The evidence supports an accept, reject, quarantine, escalate, credit, redeem, or transfer-onward decision.
3. The mechanism does not make all ZSwap transfers publicly traceable.
4. Sender, recipient, amount, and broader transaction graph information remain shielded by default.
5. Disclosure is limited to the parties and context that require it.
6. The custodian can retain audit evidence explaining why an asset was accepted, rejected, quarantined, or escalated.
7. The model can support regulated institutional workflows without forcing bespoke manual agreements for every inbound transfer.
8. The mechanism does not make permissionless shielded transfers unusable for non-regulated contexts.

## Scope

This MPS is only about **source-of-funds evidence for incoming ZSwap shielded assets**.

It covers:

* Provenance evidence for inbound shielded transfers.
* Sender, originator, institution, customer, issuer, route, or transfer-context attribution where required.
* Acceptance workflows for custodians, banks, exchanges, issuers, and regulated asset service providers.
* Audit evidence for why a shielded asset was accepted, rejected, quarantined, or escalated.
* Privacy-preserving disclosure models that avoid public traceability by default.
* The boundary between what the ledger reveals, what counterparties disclose, and what custodians retain for compliance and audit.

## Non-Goals

This MPS does not address shielded wallet generation.

It does not define how a custodian generates new shielded wallet material, derives receiving addresses, or creates customer shielded accounts.

It does not address native ZSwap shielded spending. It does not define how to spend a ZSwap coin, generate a native shielded spend proof, manage nullifiers, or authorize asset movement.

It does not address Compact contract execution or contract-based mint, burn, treasury, or multisig patterns.

It does not propose to make all shielded transfers publicly traceable.

It does not require every shielded asset flow to support unrestricted third-party deposits.

It does not define the legal compliance policy for any specific jurisdiction, institution, asset class, or regulator. The purpose of this MPS is to define the technical and architectural blocker that prevents custodians from satisfying their own source-of-funds requirements while preserving Midnight’s privacy model.

## Representative Use Cases

### Use Case 1: Custodian receives an inbound shielded transfer

A custodian detects that a shielded asset has arrived in a wallet it controls or monitors. Before crediting the customer, the custodian needs to determine whether the asset came from an acceptable source.

The ledger confirms receipt, but does not provide sufficient provenance for an acceptance decision.

### Use Case 2: Regulated institution sends to another regulated institution

A bank, issuer, exchange, or custodian sends a shielded asset to another regulated institution.

The receiving institution needs evidence that the transfer came from the claimed sender and relates to the expected customer, institution, or transaction context. This evidence should be available to the receiving institution without requiring the transfer to become publicly traceable.

### Use Case 3: Custodian quarantines an asset with insufficient evidence

A shielded asset arrives without sufficient provenance evidence.

The custodian needs an operationally safe way to quarantine, reject, hold, or escalate the asset while preserving auditability and without exposing unrelated shielded activity.

### Use Case 4: Auditor reviews an accepted asset

A custodian later needs to evidence why a shielded asset was credited, redeemed, or transferred onward.

The custodian needs a retained record of the source-of-funds basis for acceptance without revealing unrelated customer balances, unrelated transfers, or the broader shielded transaction graph.

### Use Case 5: Tokenized deposit or RWA redemption

A customer or institution presents a shielded asset for redemption.

Before honoring the redemption, the issuer or custodian may need to confirm that the asset came through an acceptable route and is associated with an approved holder, institution, or transaction context.

## Goals

1. Enable custodians to obtain privacy-preserving source-of-funds evidence for incoming ZSwap-based assets.
2. Allow regulated custodians to decide whether an incoming shielded asset can be accepted, credited, redeemed, transferred onward, quarantined, or rejected.
3. Preserve Midnight’s shielded privacy model by avoiding public sender, recipient, amount, or transaction-graph traceability as the default compliance mechanism.
4. Distinguish source-of-funds evidence from balance visibility, viewing keys, wallet generation, and spend authorization.
5. Support audit-grade evidence retention for regulated custodians without exposing unrelated shielded activity.
6. Provide future MIPs with a clear design target for selective disclosure, provenance attestations, encrypted transfer context, counterparty evidence, or other privacy-preserving source-of-funds mechanisms.

## Open Questions

1. What mechanism should allow a receiving custodian to obtain source-of-funds evidence for an incoming ZSwap asset without making the transfer publicly traceable?
2. Should provenance evidence be embedded in the shielded transfer flow, encrypted to the recipient, provided out-of-band, selectively disclosed, or issued as a verifiable attestation?
3. How can the receiving custodian verify that the evidence corresponds to the specific incoming shielded asset without revealing sender, recipient, amount, or broader transaction graph information publicly?
4. Which party should be able to produce or attest to the provenance evidence: the sender, sending custodian, issuer, receiving custodian, or an approved third-party compliance provider?
5. What protocol, SDK, wallet, indexer, metadata, or custody-integration support is required for custodians to collect, verify, store, and audit source-of-funds evidence?
6. How should the system support regulated inbound flows without making permissionless shielded transfers unusable in non-regulated contexts?
7. What is the minimum viable source-of-funds evidence flow that a custodian security, compliance, and risk team could review and approve for institutional deployment?

## Relationship to MPS-0006

This MPS is a narrowed child problem of MPS-0006, which defines the broader problem space for institutional custody of native shielded assets.

MPS-0006 includes multiple custody blockers: shielded wallet generation, native shielded asset transfer, proof generation, viewing and balance reporting, private coin state management, DUST management, and source-of-funds requirements. This document isolates the source-of-funds blocker only: whether a custodian can obtain enough provenance evidence to accept, reject, quarantine, redeem, or transfer onward an incoming shielded asset.

The purpose of this separation is to make the problem actionable for future MIPs and implementation design. Solving source-of-funds evidence is necessary for regulated inbound shielded asset workflows, but it does not solve wallet generation, native shielded spending, DUST sponsorship, or Compact contract proof-server architecture.

## Expected Outcome

If this problem is addressed, custodians will be able to make defensible inbound acceptance decisions for ZSwap shielded assets.

A receiving institution should be able to determine whether a shielded asset can be credited, held, redeemed, transferred onward, rejected, quarantined, or escalated while preserving Midnight’s privacy guarantees.

The expected outcome is NOT public traceability. It is a privacy-preserving source-of-funds evidence model that allows regulated institutions to support incoming shielded assets without losing the ability to enforce provenance, audit, and risk controls.
