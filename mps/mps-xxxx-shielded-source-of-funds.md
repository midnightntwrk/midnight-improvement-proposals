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

Regulated custodians cannot treat asset receipt and asset acceptance as the same event.

With transparent assets, a custodian can usually begin an inbound compliance review from visible transaction data: sender address, transaction history, contract path, exchange origin, or other public chain metadata. ZSwap intentionally removes much of that information from the public ledger. A custodian may be able to detect that a shielded asset arrived, but the ledger does not expose a plaintext sender or a `msg.sender`-style origin that can be used for source-of-funds review.

This creates a specific institutional blocker: a custodian may receive a shielded asset but be unable to determine whether it is allowed to credit, accept, redeem, transfer onward, or associate that asset with a customer account.

This MPS defines the problem of generating sufficient source-of-funds evidence for incoming shielded assets without turning Midnight’s shielded transaction model into a publicly traceable system.

## Problem Statement

A regulated custodian needs a way to answer a simple operational question:

**Can this incoming shielded asset be accepted?**

Today, ZSwap receipt alone does not answer that question. The custodian may know that an asset was received into a shielded wallet it controls or monitors, but receipt does not provide reliable provenance. It does not necessarily identify the sender, originating institution, customer, transfer context, or compliance status of the funds.

For regulated custody, this creates an acceptance gap. Before an incoming asset can be credited, redeemed, transferred onward, or made available to a customer, the custodian may need evidence that the source of funds is acceptable under AML, sanctions, travel-rule, audit, reconciliation, and internal risk policies.

The core issue is therefore:

**How can a custodian obtain enough provenance evidence to accept or reject an incoming shielded asset when ZSwap does not publicly reveal the sender or source of funds?**

## Why This Is a Separate Problem

This problem should not be collapsed into wallet generation, shielded spending, or proof-server architecture.

It is also not the same as balance visibility.

A viewing capability may help a custodian detect that an asset was received. It may support balance reporting, account reconciliation, or operational monitoring. But it does not necessarily tell the custodian where the asset came from, who sent it, which customer or institution originated it, or whether the inbound flow satisfies the custodian’s risk policy.

The distinction is:

* **Receipt:** the custodian can see that an asset arrived.
* **Attribution:** the custodian can identify or evidence where it came from.
* **Acceptance:** the custodian can decide whether it is permitted to credit, hold, redeem, or transfer the asset.

This MPS is about attribution and acceptance.

## Core Problem

A custodian needs a privacy-preserving mechanism for inbound shielded asset review that supports the following properties:

1. The custodian can obtain sufficient source-of-funds evidence for an incoming shielded asset.
2. The evidence can support an accept, reject, quarantine, or escalate decision.
3. The mechanism does not require all ZSwap transfers to become publicly traceable.
4. The sender, recipient, amount, and broader transaction graph remain shielded by default.
5. Disclosure is limited to the parties and context that require it.
6. The custodian can retain audit evidence for why the asset was accepted or rejected.
7. The model can support regulated institutional workflows without relying on one-off manual agreements for every inbound transfer.

## Scope

This MPS covers source-of-funds evidence for **incoming shielded assets**.

It includes:

* Provenance evidence for inbound shielded transfers.
* Sender, originator, institution, customer, or transfer-context attribution.
* Acceptance workflows for custodians, banks, exchanges, and regulated asset service providers.
* Audit evidence for why a shielded asset was accepted, rejected, quarantined, or escalated.
* Privacy-preserving disclosure models that avoid public traceability by default.
* The boundary between what the ledger reveals, what counterparties disclose, and what custodians retain for compliance.

## Non-Goals

This MPS does not define shielded wallet generation.

It does not define how to spend a ZSwap coin, generate a native shielded spend proof, manage nullifiers, or authorize asset movement.

It does not define Compact contract proof-server architecture.

It does not propose to make all shielded transfers publicly traceable.

It does not require every shielded asset flow to support unrestricted third-party deposits.

It does not define the legal compliance policy for any specific jurisdiction. The goal is to define the technical and architectural problem that prevents custodians from satisfying their own source-of-funds requirements.

## Representative Use Cases

### Use Case 1: Custodian receives an inbound shielded transfer

A custodian detects that a shielded asset has arrived in a wallet it controls. Before crediting the customer, the custodian needs to determine whether the asset came from an acceptable source.

The ledger confirms receipt, but does not provide enough provenance for an acceptance decision.

### Use Case 2: Regulated institution sends to another regulated institution

A bank, issuer, exchange, or custodian sends a shielded asset to another regulated institution.

The receiving institution needs evidence that the transfer came from the claimed sender and relates to the expected customer or transaction context, without requiring the transfer to become publicly traceable.

### Use Case 3: Custodian must quarantine an asset

A shielded asset arrives without sufficient provenance evidence.

The custodian needs an operationally safe way to quarantine, reject, hold, or escalate the asset while preserving auditability and without exposing unrelated shielded activity.

### Use Case 4: Auditor asks why an asset was accepted

A custodian later needs to evidence why a shielded asset was credited or redeemed.

The custodian needs a retained record of the source-of-funds basis for acceptance, without needing to reveal all customer balances, unrelated transfers, or the broader shielded transaction graph.

### Use Case 5: Tokenized deposit or RWA redemption

A customer or institution presents a shielded asset for redemption.

Before honoring the redemption, the issuer or custodian may need to confirm that the asset came through an acceptable route and is associated with an approved holder or institution.

## Goals

1. Enable custodians to obtain sufficient provenance evidence for incoming shielded assets when required.

2. Support clear inbound asset states such as pending review, accepted, rejected, quarantined, or escalated.

3. Preserve Midnight’s privacy model by avoiding public sender/recipient traceability as the default compliance mechanism.

4. Distinguish source-of-funds evidence from balance visibility, viewing keys, wallet generation, and spend authorization.

5. Support audit-grade evidence retention for regulated custodians without exposing unrelated shielded activity.

6. Provide future MIPs with a clear problem boundary for selective disclosure, provenance attestations, counterparty evidence, or other privacy-preserving source-of-funds mechanisms.

## Open Questions

1. What is the minimum provenance evidence a custodian needs before accepting an incoming shielded asset?

2. Should provenance evidence identify the sender wallet, sending institution, customer, transfer instruction, asset issuer, or some combination of these?

3. Should evidence be attached to the transfer, provided out-of-band, selectively disclosed, encrypted to the recipient, or produced as a later attestation?

4. Who is responsible for producing source-of-funds evidence: the sender, the sending custodian, the issuer, the receiving custodian, or a third-party compliance provider?

5. Can provenance evidence be verified without exposing sender, recipient, amount, or transaction graph information publicly?

6. How should a custodian handle assets that arrive without sufficient evidence?

7. What should be visible to auditors or regulators, and what should remain private?

8. Which parts of this problem require protocol support, and which can be handled through SDKs, wallets, metadata standards, custody infrastructure, or off-chain compliance messaging?

9. How can the system support regulated flows without making permissionless shielded transfers unusable for non-regulated contexts?

10. Can source-of-funds evidence be standardized enough that custodians are not forced into bespoke integrations for every counterparty or issuer?

## Expected Outcome

If this problem is addressed, custodians will be able to make defensible inbound acceptance decisions for shielded assets.

A receiving institution should be able to determine whether a shielded asset can be credited, held, redeemed, transferred onward, rejected, or quarantined, while preserving Midnight’s privacy guarantees.

The desired outcome is not public traceability. The desired outcome is selective, auditable, privacy-preserving provenance evidence that allows regulated institutions to support shielded assets without losing the ability to enforce source-of-funds controls.
