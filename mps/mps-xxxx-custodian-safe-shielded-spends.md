---
MPS: "0024"
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

Native ZSwap shielded asset transfer is a blocker for institutional custody because asset movement on Midnight is proof-authorized, not primarily signature-authorized.

Institutional custodians normally authorize asset movement through threshold-controlled infrastructure such as MPC, TSS, HSMs, multisig, policy engines, approval workflows, and audited recovery procedures. These systems are built around a familiar pattern: a transaction is constructed, reviewed, approved, and then authorized through a controlled signing process.

ZSwap shielded asset transfer follows a different model. Moving a native shielded asset requires spending an existing shielded coin by producing a valid zero-knowledge proof over private coin and witness material. The ledger verifies the spend proof and nullifier. A conventional signature or 2-of-3 approval flow does not, by itself, authorize or execute the shielded spend.

This creates a custody blocker: a custodian may be able to approve a transfer through its normal threshold-control process, but still lack a custody-safe way to generate the ZSwap spend proof required to move the asset.

This MPS defines the narrow problem of enabling native ZSwap shielded asset transfers inside institutional custody security boundaries.

## Problem Statement

Regulated custodians need to safely handle and transfer ZSwap-based assets without exposing coin secrets, private spend witnesses, or other spend-authorizing material outside approved custody infrastructure.

In conventional custody integrations, asset movement is usually authorized by a digital signature. The custodian can construct a transaction, apply policy controls, collect approvals, and use MPC, TSS, HSM, or multisig infrastructure to produce the required signature. The signing operation is the primary custody-controlled authorization event.

ZSwap does not follow this model. A native shielded transfer requires a zero-knowledge proof over private coin material, including coin secret material, coin information, Merkle inclusion data, commitment relationships, nullifier derivation data, and related private witness inputs required to prove that the coin can be spent.

This is not a discovery problem. The sensitive material required for a ZSwap spend is already part of the native shielded transfer model. The problem is that institutional custody systems are not designed around exporting that material to ordinary application infrastructure or treating an external proof server as a safe equivalent to an MPC, TSS, HSM, or policy-controlled signing boundary.

This creates a specific incompatibility for custodians. Policy approval and threshold signing can approve the intent to move an asset, but they do not generate the ZSwap proof required by the ledger. If the proof-generation flow requires spend-authorizing material to be exported to ordinary infrastructure, the custodian cannot support native shielded transfer without weakening its security, compliance, or key-management model.

The core requirement is:

**Enable custodians to safely authorize and execute native ZSwap shielded spends when asset movement requires a zero-knowledge proof over private coin material rather than a conventional transaction signature.**

## Technical Background

Institutional custody systems are built around secure transaction authorization. A custodian may use MPC, TSS, HSMs, multisig, approval workflows, policy engines, recovery keys, and audit controls to ensure that customer assets cannot move without the required institutional authorization.

In most custody integrations, ordinary application infrastructure can prepare the transaction, request approvals, and broadcast the final transaction. The sensitive operation is signing, and the signing material remains inside the approved custody boundary.

Midnight shielded assets require an additional custody-controlled operation. ZSwap assets exist as shielded notes or coins. To transfer one, the spender must construct a valid shielded spend and produce a zero-knowledge proof that demonstrates spend validity without revealing the private coin data publicly.

This means native ZSwap transfer is not simply a question of whether a custodian can sign a transaction. The custodian must also control the private witness material required to create the spend proof. Any service that can access or process that material becomes part of the custody security boundary and should be treated as handling key-equivalent or spend-authorizing data.

The blocker is therefore not general transaction submission. The blocker is custodian-safe proof generation and spend execution for native ZSwap assets.

## Core Requirements

A custodian-safe native shielded transfer model should preserve the following properties:

1. ZSwap-based assets can be transferred from custodian-controlled shielded wallets without exporting spend-authorizing material to ordinary application infrastructure.
2. Institutional policy approval, threshold control, and audit workflows remain required parts of the transfer process.
3. Proof generation is executed inside an approved custody boundary or through an architecture that provides equivalent security assurances to the custodian.
4. Coin secrets, private witness material, nullifier derivation data, Merkle inclusion data, and private coin state are protected as custody-sensitive material.
5. The custodian can identify, select, prove, and spend the correct shielded notes without relying on unsafe bespoke off-chain workarounds.
6. The transfer flow binds custody approval to proof generation and transaction submission, so that a valid spend cannot be produced outside the approved policy path.
7. The resulting process is operationally auditable without weakening Midnight's shielded privacy guarantees.

## Scope

This MPS is only about **native ZSwap shielded asset spends and transfers in custodian environments**.

It covers:

- Spending existing shielded coins from a wallet-style ZSwap account.
- Authorizing native shielded asset movement under institutional custody controls.
- Generating the zero-knowledge spend proof required for native ZSwap transfer.
- Protecting coin secrets, nullifier-related data, Merkle inclusion data, and private witness material.
- Binding institutional policy approval to proof-based spend execution.
- Supporting custodian workflows where a customer or institution instructs the custodian to move, transfer, redeem, or otherwise spend a native shielded asset.

## Non-Goals

This MPS does not address shielded wallet generation.

It does not define how a custodian generates new shielded wallet material, derives receiving addresses, or creates customer shielded accounts.

It does not address source-of-funds attribution. Determining where an incoming shielded asset came from is a separate compliance and provenance problem.

It does not address Compact contract execution or contract-based mint, burn, treasury, or multisig patterns. Those flows may also require proof generation, but they are distinct from native wallet-style ZSwap spends.

It does not define a general DUST sponsorship, fee abstraction, or transaction-funding model. DUST may be required for transaction execution, but this MPS is limited to the native shielded spend problem.

It also does not prescribe a specific implementation such as TEEs, HSM-adjacent proving, MPC-based proving, collaborative proving, coSNARKs, proof-share aggregation, protocol changes, SDK changes, or a particular custodian architecture. Those may be explored in future MIPs.

## Representative Use Cases

### Use Case 1: Custodian transfers a shielded asset for a customer

A customer holds a native shielded asset through a regulated custodian and instructs the custodian to transfer it to another shielded address.

The custodian applies its normal policy and approval controls, then executes the required ZSwap spend proof inside an approved custody boundary without exposing private coin material to ordinary application systems.

### Use Case 2: Custodian redeems a shielded asset

A customer exits a tokenized deposit, stablecoin, money market fund, or RWA product. The custodian needs to move or redeem the shielded asset as part of the off-chain redemption process.

If the asset is held as a native ZSwap shielded coin, the custodian must be able to spend that coin safely. A standard transaction signature is not sufficient to execute the shielded spend.

### Use Case 3: Custodian moves assets between institutional accounts

An institution holds shielded assets in custodian-controlled wallets and needs to move them between internal treasury accounts, operational wallets, or counterparties.

The custodian must support native shielded transfer while preserving its threshold-control model and preventing private witness material from reaching ordinary application systems.

### Use Case 4: Custodian evaluates native shielded custody feasibility

Before supporting native Midnight shielded assets, a custodian's security and risk teams need to determine whether ZSwap spends can be executed inside their approved custody model.

If every spend requires private coin material to be exported to an ordinary proof server, the custodian may reject native shielded transfer support.

## Goals

1. Enable custodians to spend and transfer ZSwap-based assets inside institutional custody environments.
2. Allow native ZSwap spend execution to fit within existing custody security architectures, including MPC, TSS, HSMs, multisig, policy engines, approval workflows, recovery procedures, and audit controls.
3. Avoid any transfer model that requires custodians to weaken, bypass, or duplicate their existing security architecture in order to support ZSwap assets.
4. Define a custody-safe native spend model that supports customer transfers, redemptions, and institutional treasury movement from custodian-controlled shielded wallets.
5. Provide future MIPs with a clear design target for custodian-safe ZSwap spending, without conflating this work with wallet generation, source-of-funds attribution, DUST sponsorship, or Compact contract execution.

## Open Questions

1. What architecture or protocol support is required for a custodian to execute native ZSwap spends without moving spend-authorizing material outside its approved custody boundary?
2. Which components of the native ZSwap spend flow must operate inside the custodian's security boundary for the model to be acceptable to institutional risk and security teams?
3. What protocol, SDK, wallet, indexer, or proof-generation interfaces are required for a custodian to identify, select, prove, and spend shielded notes through its existing custody workflow?
4. How should the private coin state needed for future spends be stored, synchronized, recovered, and protected without creating an unapproved secondary custody system?
5. How should a custodian evidence that a native ZSwap spend was approved, policy-compliant, and executed through the authorized custody path without revealing shielded transaction details publicly?
6. What is the minimum viable native ZSwap spend flow required for custodians to support customer transfers, redemptions, and treasury movements while preserving their existing security architecture?

## Relationship to MPS-0006

This MPS is a narrowed child problem of MPS-0006, which defines the broader problem space for institutional custody of native shielded assets.

MPS-0006 includes multiple custody blockers: shielded wallet generation, native shielded asset transfer, proof generation, viewing and balance reporting, private coin state management, DUST management, and source-of-funds requirements. This document isolates the native transfer blocker only: whether a custodian can safely authorize and execute a ZSwap shielded spend when movement requires a zero-knowledge proof over private coin material.

The purpose of this separation is to make the problem actionable for future MIPs and implementation design. Solving native shielded transfer is necessary for general-purpose shielded custody, but it does not solve wallet generation, source-of-funds attribution, DUST sponsorship, or Compact contract proof-server architecture.

## Expected Outcome

If this problem is addressed, institutional custodians will be able to transfer native Midnight shielded assets without relying on unsafe witness export, bespoke off-chain workarounds, or ordinary application proof generation that handles spend-authorizing material.

This would unlock a core requirement for native shielded asset custody: ZSwap-based assets held in custodian-controlled shielded wallets could later be moved, transferred, redeemed, or used in broader institutional workflows while preserving both Midnight's privacy model and the custodian's security model.

The expected outcome is not a complete custody solution. It is a clear, custodian-compatible foundation for native ZSwap shielded transfer that future MIPs can build on for broader institutional shielded asset workflows.