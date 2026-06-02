---
MPS: xxxx
Title: Custodian-Safe Native Shielded Asset Transfer
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

Native shielded asset transfer on Midnight does not currently map cleanly to the transaction-authorization models used by institutional custodians.

Custodians such as BitGo, Fireblocks, and similar institutional providers normally authorize asset movement through MPC, TSS, HSMs, multisig, policy engines, and approval workflows. These systems are built around the assumption that a transaction can be constructed, reviewed, approved, and then authorized through a controlled signing process.

ZSwap shielded asset transfer is different. Moving a shielded asset requires spending an existing shielded coin by producing a valid zero-knowledge proof over private coin material. The ledger verifies the spend proof and nullifier, rather than relying only on a conventional signature or 2-of-3 signing flow.

This creates a blocker for institutional custody. A custodian may be able to approve a transaction through its normal threshold-control process, but that approval does not by itself produce the ZSwap spend proof required to move the shielded asset.

This MPS defines the problem of executing native ZSwap shielded asset transfers in a way that is compatible with institutional custody security models.

## Problem Statement

Regulated custodians need to authorize and execute native shielded asset transfers without exposing full coin secrets, private spend witnesses, or other key-equivalent material outside approved custody boundaries.

In conventional custody integrations, asset movement is usually authorized by a digital signature. The custodian can construct a transaction, apply policy controls, collect approvals, and then use MPC, TSS, HSMs, or multisig infrastructure to produce the required signature.

ZSwap does not fit this model directly. A native shielded transfer requires the spender to produce a zero-knowledge proof over private coin material. This may include the coin secret key, coin information, Merkle inclusion data, commitment relationships, nullifier derivation data, and other private witness material required to prove that the coin can be spent.

This creates a specific problem for custodians: even if the custodian can approve the transfer under its normal policy model, it still needs a custody-safe way to generate the required spend proof without exposing private coin material to ordinary application infrastructure.

The core question is:

**How can a custodian authorize and execute a native ZSwap shielded spend when movement requires a proof over private coin material rather than a normal signature?**

## Background

Institutional custody systems are usually built around threshold-controlled signing.

A custodian may use MPC, TSS, HSMs, multisig, approval workflows, transaction policies, recovery keys, and audit controls to ensure that customer assets cannot move without the required institutional authorization. The private signing material remains inside a secure boundary, while ordinary application infrastructure may prepare the transaction, request approvals, and broadcast the final signed transaction.

Midnight shielded assets introduce a different execution model. ZSwap assets exist as shielded notes or coins. To move one, the spender must prove knowledge of the relevant private coin material and prove that the spend is valid without revealing the private details publicly.

This means that native shielded asset transfer is not simply a question of “can the custodian sign the transaction?” The custodian must also manage the private witness material needed to create the ZSwap spend proof. Any system that handles that material may become part of the custody security boundary.

## Core Problem

A custodian needs a way to execute native ZSwap shielded spends while preserving the following properties:

1. Transfer authorization should remain compatible with institutional approval, policy, MPC, TSS, HSM, or multisig controls.
2. Full coin secrets and private spend witness material should not be exposed to ordinary application infrastructure.
3. Proof generation should not create an uncontrolled hot-key-equivalent service.
4. The custodian should be able to prove, spend, transfer, redeem, or otherwise move shielded assets after receiving them.
5. The model should clearly define what material is required to generate a spend proof, where that material is processed, and whether any of it is key-equivalent.
6. The resulting transfer flow should be auditable and operationally usable by regulated custodians without weakening Midnight’s privacy guarantees.

## Scope

This MPS is only about **native ZSwap shielded asset spends and transfers**.

It covers:

* Spending existing shielded coins from a wallet-style ZSwap account.
* Authorizing shielded asset movement under institutional custody controls.
* Generating the spend proof required for native ZSwap transfer.
* Protecting coin secrets, nullifier-related data, Merkle inclusion data, and other private witness material.
* Clarifying the relationship between threshold approval and proof-based spend authorization.
* Supporting custodian workflows where a customer or institution instructs the custodian to move a shielded asset.

## Non-Goals

This MPS does not address shielded wallet generation.

It does not define how a custodian generates a new shielded address, derives receiving keys, or creates shielded wallet material for customers.

It does not address source-of-funds attribution. Determining where an incoming shielded asset came from is a separate compliance and provenance problem.

It does not address Compact contract execution or contract-based mint/burn patterns. Those flows may also require proof generation, but they are distinct from native wallet-style ZSwap spends.

It does not prescribe a specific solution such as TEEs, HSM-adjacent proving, MPC-in-the-head, collaborative proving, coSNARKs, proof-share aggregation, or protocol changes. Those may be explored in future MIPs, but this MPS is limited to defining the native shielded spend problem.

## Representative Use Cases

### Use Case 1: Custodian transfers a shielded asset for a customer

A customer holds a shielded asset through a regulated custodian and instructs the custodian to transfer it to another shielded address.

The custodian can apply normal approval and policy controls, but it must also generate the required ZSwap spend proof without exposing the customer’s private coin material outside approved custody infrastructure.

### Use Case 2: Custodian redeems a shielded asset

A customer exits a tokenized deposit, stablecoin, money market fund, or RWA product. The custodian needs to move or redeem the shielded asset as part of the off-chain redemption process.

If the asset is held as a native shielded coin, the custodian must be able to spend that coin safely. A standard signature approval is not enough.

### Use Case 3: Custodian manages institutional treasury movement

An institution holds shielded assets in a custodian-controlled wallet and needs to move them between internal treasury accounts, counterparties, or operational wallets.

The custodian must support native shielded transfer while preserving its threshold-control model and avoiding exposure of private witness material to ordinary application systems.

### Use Case 4: Custodian evaluates whether native shielded custody is feasible

Before supporting Midnight native shielded assets, a custodian’s security and risk teams need to understand whether ZSwap spends can be executed inside their custody model.

If every spend requires private coin material to be exported to an ordinary proof server, the custodian may reject support for native shielded asset transfers.

## Goals

1. Ensure that native ZSwap shielded asset transfers are compatible with institutional custody architectures, including MPC, TSS, HSM, multisig, policy approval, and audit-control environments.

2. Ensure that custodians can authorize shielded asset movement without relying only on ordinary signature-based transaction approval.

3. Ensure that the private material required for ZSwap spend proof generation is clearly identified, classified, and protected within the appropriate custody boundary.

4. Ensure that proof generation for native shielded spends does not require unsafe exposure of full coin secrets or private spend witness material to ordinary application infrastructure.

5. Support a clear separation between custody policy approval and ZSwap proof generation, while ensuring both are required parts of a valid institutional transfer flow.

6. Provide future MIPs with a clear problem boundary for designing custodian-compatible native shielded spend flows, without conflating this work with shielded wallet generation, source-of-funds attribution, or Compact contract execution.

## Open Questions

1. What exact private material is required to generate a native ZSwap spend proof?

2. Which parts of that material should custodians treat as key-equivalent or spend-authorizing?

3. Can threshold approval be separated from proof generation in a way that still satisfies institutional custody requirements?

4. Can a native ZSwap spend proof be generated without reconstructing full coin secrets or private witness material in one place?

5. Can proof generation be performed inside MPC, TSS, HSM-adjacent infrastructure, a TEE, collaborative proving, coSNARKs, proof-share aggregation, or another custody-compatible model?

6. What protocol, SDK, wallet, indexer, or infrastructure changes would be required to support custodian-safe native shielded spends?

7. How should the custodian manage the private coin state required to identify, select, prove, and spend the correct shielded notes?

8. How should the transfer flow account for DUST funding, transaction balancing, and submission without expanding the custody risk surface?

9. What audit evidence should be available to the custodian to prove that a shielded transfer was authorized, policy-compliant, and executed correctly?

10. What is the minimum viable native shielded spend flow that a custodian could review and approve for institutional deployment?

## Expected Outcome

If this problem is addressed, institutional custodians will be able to execute native Midnight shielded asset transfers without relying on unsafe witness export, bespoke off-chain workarounds, or ordinary application proof generation that handles key-equivalent material.

This would unlock a core requirement for native shielded asset custody: assets held in a custodian-controlled shielded wallet could later be moved, transferred, redeemed, or used in broader institutional workflows while preserving both Midnight’s privacy model and the custodian’s security model.

Solving this problem does not by itself solve shielded wallet generation, source-of-funds attribution, or Compact contract proof-server architecture. It only addresses the native ZSwap spend path: how a custodian safely authorizes and executes shielded asset movement when the ledger requires a zero-knowledge proof rather than a conventional transaction signature.
