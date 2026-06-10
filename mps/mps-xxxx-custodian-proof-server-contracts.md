---
MPS: xxxx
Title: Custodian-Compatible Compact Contract Proof Generation
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

Compact contract interaction on Midnight introduces a custody execution problem because contract calls may require proof generation, not only transaction approval or signing.

Institutional custodians are built around controlled transaction workflows: policy review, approvals, MPC/TSS/HSM or multisig signing, submission, recovery, and audit. Compact contract calls add another required step: a proof must be generated over contract-specific witness data before the transaction can be verified on Midnight.

This creates a custodian blocker. Even if a custodian can approve and sign a contract action, it still needs a custody-compatible way to generate the required Compact proof without exposing sensitive private witness data to ordinary application infrastructure.

This MPS defines the narrow problem of making Compact contract proof generation usable inside institutional custody environments.

## Problem Statement

Regulated custodians need to execute Compact contract calls without compromising their existing security architecture.

A contract-based Midnight flow may be used for mint, burn, redemption, treasury, admin, issuer-control, or multisig-style operations. These flows may deliberately avoid open-ended native ZSwap transfers, but they do not avoid proof generation. If a Compact call requires private witness data, the custodian must understand where that data is processed, who can access it, and whether the proving environment belongs inside an approved custody or secure execution boundary.

The core issue is not native ZSwap spending. In a native ZSwap spend, the witness is private coin and wallet-state material. In a Compact contract interaction, the witness is contract-specific private data used to prove that a smart-contract action is valid. The custody concern is similar, but the problem boundary is different.

The core question is:

**How can a custodian execute Compact contract calls that require proof generation without exposing sensitive private contract witness data outside an approved custody or secure execution boundary?**

## Technical Context

Conventional custody integrations usually treat signing as the sensitive operation. A transaction can often be assembled outside the secure boundary, while signing authority remains protected inside MPC, TSS, HSM, multisig, or equivalent infrastructure.

Compact changes that model. The custodian must account not only for signing or approval, but also for the proving step required by Midnight contract execution. Depending on the contract flow, the witness may include private inputs, customer-specific data, authorization facts, issuer or treasury context, contract state, or other sensitive execution data.

The blocker is therefore the proof-generation trust boundary for custodial Compact workflows.

## Scope

This MPS is only about **Compact contract proof generation in custodial environments**.

It covers:

* Compact calls that require proof generation.
* Contract-based mint, burn, redemption, treasury, admin, issuer-control, and multisig-style flows.
* Handling private contract witness data in a custodian deployment.
* Integrating proof generation into custodian policy, signing, submission, and audit workflows.
* Determining when a proving environment must be internal, isolated, attested, delegated, or otherwise protected.

## Non-Goals

This MPS does not address shielded wallet generation.

It does not define native ZSwap shielded spends or wallet-style asset transfers.

It does not define source-of-funds attribution for incoming shielded assets.

It does not define asset-level business logic such as freeze, force transfer, holder eligibility, verified-counterparty transfer, or updatable transfer restrictions.

It does not define a general DUST sponsorship, fee abstraction, or transaction-funding model.

It does not prescribe a specific proving architecture such as TEEs, HSM-adjacent proving, coSNARKs, collaborative proving, proof-share aggregation, local proving, remote proving, or protocol changes.

## Representative Use Cases

### Use Case 1: Tokenized deposit mint

A bank customer opts into a tokenized deposit product through a Web2 banking interface. The custodian or banking platform triggers a Compact contract call to mint or record the corresponding position.

The custodian can approve the action through normal controls, but the Compact call still requires proof generation.

### Use Case 2: Burn or redemption

A customer exits a product and the custodian triggers a burn, redemption, or withdrawal-related Compact contract call.

This may not be a native ZSwap wallet transfer, but the custodian still needs a safe proving path for any private witness data used by the contract.

### Use Case 3: Treasury, admin, or issuer action

An institution uses Compact contracts for treasury operations, asset administration, issuer permissions, or multisig-style controls.

The custodian's policy engine can approve the action, but Midnight execution still depends on proof generation that must fit the custodian's security and audit model.

### Use Case 4: Custodian security review

A custodian's risk team reviews Midnight support and asks what the proving environment receives, whether it processes customer-specific or action-authorizing data, whether it can be operated outside approved infrastructure, and what happens if it is compromised.

Without clear answers, the custodian may reject Compact contract support or restrict Midnight integration to flows that do not use sensitive witness data.

## Goals

1. Enable custodians to execute Compact contract calls on Midnight without weakening or bypassing their existing custody security architecture.
2. Make proof generation for Compact contract interactions compatible with institutional custody workflows when private or sensitive witness data is involved.
3. Support restricted institutional flows such as mint, burn, redemption, treasury, admin, issuer-control, and multisig-style contract calls.
4. Clearly separate this problem from native ZSwap shielded spending, where the witness is private coin and wallet-state material.
5. Provide future MIPs with a focused design target for custodian-compatible Compact contract proof generation.

## Open Questions

1. What custody-compatible proving model should Compact contract calls support when private or sensitive witness data is used?
2. When must Compact proof generation occur inside an approved custody or secure execution boundary?
3. What protocol, SDK, wallet, proof-server, or custody-integration interfaces are required for custodians to invoke Compact proof generation through their existing workflows?
4. How should private contract witness data be passed into the proving environment, isolated during proving, and retained or deleted after proving?
5. What audit evidence should link custodian approval, proof generation, transaction construction, signing, submission, and final execution?
6. What is the minimum viable Compact proof-generation flow that a custodian security team could approve for restricted institutional deployments?

## Relationship to MPS-0006

This MPS is a narrowed child problem of MPS-0006, which defines the broader problem space for institutional custody of native shielded assets.

MPS-0006 includes multiple custody blockers, including shielded wallet generation, native shielded asset transfer, proof generation, viewing, private state management, DUST management, and source-of-funds requirements. This document isolates the Compact contract proof-generation blocker only.

It overlaps with the native shielded spend MPS at the proof-generation layer, but the witness material is different. Native shielded spends involve ZSwap coin and wallet-state material. Compact contract interactions involve contract-specific private witness data used to prove a contract action.

## Expected Outcome

If this problem is addressed, custodians will be able to execute Compact contract calls through a clearly defined, custody-compatible proof-generation architecture.

A custodian should be able to approve a contract action, generate the required proof, sign or authorize the transaction where needed, submit it to Midnight, and retain audit evidence without exposing sensitive contract witness data to ordinary application infrastructure.

The expected outcome is not a complete native shielded custody solution. It is a focused foundation for Compact contract proof generation that custodians can review, approve, and operate for institutional Midnight workflows.
