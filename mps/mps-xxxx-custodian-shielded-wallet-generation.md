---
MPS: xxxx
Title: Custodian-Compatible Shielded Wallet Generation
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

Midnight shielded wallet generation is a blocker for institutional custody because ZSwap receiving addresses are derived from shielded secret material, not from a standard public key produced by a conventional asymmetric key-generation process.

Institutional custodians normally create and manage wallets through threshold-controlled infrastructure such as MPC, TSS, HSMs, DKG, multisig, policy engines, and audited recovery workflows. These systems are designed so that no ordinary application service, operator, or single custodian participant reconstructs a full private key or key-equivalent secret.

ZSwap wallet generation follows a different model. The ZSwap coin secret key is random 256-bit material, and the corresponding coin public key is derived as `Hash<ZswapCoinSecretKey>`, using SHA-256. As a result, address generation depends directly on shielded secret material. This creates a custody blocker: generating a shielded wallet may require processing material that custodians treat as key-equivalent.

This MPS defines the narrow problem of generating ZSwap-compatible shielded wallets and receiving addresses inside institutional custody security boundaries.

## Problem Statement

Regulated custodians need to create Midnight shielded wallets for customers without exposing full shielded secret material outside approved custody infrastructure.

In conventional custody integrations, a custodian can usually generate threshold-controlled key material, derive a public address from public key material, and keep signing authority protected inside MPC, TSS, HSM, or equivalent infrastructure. Address derivation does not normally require the full private key to be reconstructed in ordinary application infrastructure.

ZSwap does not follow this model. Its coin public key is derived by hashing shielded secret material. In a standard threshold custody environment, SHA-256 over a secret is not equivalent to ordinary public-key derivation, where participants can independently derive public shares and combine them into an address.

This creates a specific incompatibility for custodians. If generating a shielded address requires the full ZSwap secret to exist in one unapproved place, a custodian cannot support Midnight shielded wallet generation without weakening its security, compliance, or key-management model.

The core question is:

**How can a custodian generate Midnight shielded wallets and receiving addresses without reconstructing or exposing the full ZSwap secret outside its approved custody boundary?**

## Technical Background

Institutional custody systems are built around secure key lifecycle management. Wallet creation, address derivation, policy approval, transaction authorization, recovery, and auditability are all designed around the principle that no single uncontrolled environment can unilaterally control customer assets.

Midnight shielded assets introduce a different requirement at the wallet-generation layer. ZSwap uses shielded notes or coins, and the receiving capability is tied to secret material that does not behave like a standard ECDSA, EdDSA, or Schnorr public/private keypair. The public receiving key is derived from the shielded secret itself.

For ordinary end-user wallets, generating this material locally may be acceptable. For institutional custody, the question is materially different: where is the shielded secret generated, where is it stored, where is it processed, who or what can reconstruct it, and whether the address-generation process remains inside the custodian's approved security boundary.

The blocker is therefore not general wallet creation. The blocker is custodian-compatible shielded wallet generation where the receiving address depends on key-equivalent shielded material.

## Core Requirements

A custodian-compatible shielded wallet generation model should preserve the following properties:

1. Full shielded secret material is not exposed to ordinary application servers.
2. No unapproved custodian participant, service, or operator can reconstruct the full shielded secret.
3. Address generation occurs inside an approved custody boundary or through a threshold-compatible mechanism acceptable to the custodian's security and risk teams.
4. The generated wallet material is recoverable, auditable, and operationally manageable at institutional scale.
5. The generation model does not create a dead-end architecture that prevents future custody workflows such as receiving, viewing, proving, or spending from being implemented safely.
6. The model clearly documents what material is generated, where it exists, how it is protected, and whether custodians should treat it as key-equivalent.

## Scope

This MPS is only about **shielded wallet and receiving address generation**.

It covers:

* Generating ZSwap-compatible shielded wallet material.
* Deriving shielded receiving addresses or public receiving keys.
* Making address generation compatible with institutional custody infrastructure.
* Avoiding unsafe reconstruction or exposure of full shielded secret material.
* Defining the custody boundary for wallet generation.
* Supporting onboarding flows where a custodian creates one or many shielded wallets for customers.

## Non-Goals

This MPS does not address native shielded asset transfer.

It does not define how a custodian spends a ZSwap coin, generates a spend proof, manages nullifiers, constructs shielded transactions, funds DUST, executes Compact contract calls, or performs source-of-funds attribution.

It also does not prescribe a specific implementation such as MPC hashing, HSM-resident generation, TEE generation, coSNARKs, protocol changes, SDK changes, or a particular custodian architecture. Those may be explored in future MIPs. This MPS is limited to defining the wallet-generation problem.

## Representative Use Cases

### Use Case 1: Custodian creates a shielded wallet for a bank customer

A bank customer opts into a Midnight-backed product through a normal Web2 banking interface. Behind the scenes, the bank's custodian must create a Midnight shielded wallet for that customer. The customer should not see seed phrases, wallet software, private keys, blockchain transactions, or signing flows.

The custodian must generate the shielded wallet using approved custody infrastructure and without exposing the full ZSwap secret to ordinary application services.

### Use Case 2: Custodian creates shielded wallets at scale

An institutional custodian needs to create thousands or millions of customer shielded wallets. The generation process must be repeatable, auditable, recoverable, and compatible with existing custody controls.

The custodian cannot rely on a manual or bespoke process where full shielded secrets are generated outside its approved custody environment.

### Use Case 3: Custodian derives a receiving address without spending authority leakage

A custodian needs to derive a shielded receiving address that can be shared with an issuer, counterparty, or internal treasury system. The derivation process must not leak spending authority or expose key-equivalent material to systems that only need receiving information.

### Use Case 4: Custodian evaluates Midnight integration feasibility

Before supporting Midnight, a custodian's security and risk teams need to determine whether shielded wallet generation can fit within their existing key-isolation, policy, recovery, and audit architecture.

If the only available model requires full shielded secrets to be assembled in ordinary infrastructure, the custodian may reject native shielded wallet support.

## Goals

1. Ensure that ZSwap-compatible shielded wallet generation can be evaluated against institutional custody architectures.
2. Define the custody boundary requirements for generating, deriving, storing, and processing shielded wallet material.
3. Enable custodians to derive receiving addresses without exposing full shielded secret material to ordinary application infrastructure.
4. Support scalable, recoverable, and auditable wallet generation for institutional onboarding flows.
5. Give future MIPs a clear problem boundary for custodian-compatible shielded wallet generation, without conflating this work with native shielded transfers, proof generation, source-of-funds attribution, DUST management, or contract execution.

## Open Questions

1. Can a ZSwap receiving address be generated without reconstructing the full shielded secret in a single ordinary execution environment?
2. Can the required hash operation over secret material be made compatible with MPC, TSS, HSM, DKG, TEE, or another approved custody model?
3. Which pieces of ZSwap wallet-generation material should custodians treat as key-equivalent?
4. Can receiving capability, viewing capability, and spending capability be separated in a way that maps cleanly to custody controls?
5. What minimum protocol, SDK, wallet, or infrastructure changes would be required to support custodian-compatible shielded wallet generation?
6. Can custodians generate shielded wallets deterministically, recoverably, and auditably without weakening privacy or exposing customer-specific secret material?
7. What information must be documented for a custodian security team to approve a Midnight shielded wallet-generation flow?

## Relationship to MPS-0006

This MPS is a narrowed child problem of MPS-0006, which defines the broader problem space for institutional custody of native shielded assets.

MPS-0006 includes multiple custody blockers: shielded wallet generation, native shielded asset transfer, proof generation, viewing and balance reporting, private coin state management, DUST management, and source-of-funds requirements. This document isolates the first blocker only: whether a custodian can generate ZSwap-compatible shielded wallet material and receiving addresses without exposing full shielded secret material outside its approved custody boundary.

The purpose of this separation is to make the problem actionable for future MIPs and implementation design. Solving shielded wallet generation is necessary for native shielded custody, but it is not sufficient to solve shielded transfers, proving, compliance provenance, or contract execution.

## Expected Outcome

If this problem is addressed, institutional custodians will be able to create Midnight shielded wallets for customers without relying on unsafe secret reconstruction or bespoke off-chain workarounds.

This would unlock the first step of native shielded asset custody: allowing a custodian to onboard users into shielded asset products while preserving both Midnight's privacy model and the custodian's security model.

The expected outcome is not a complete custody solution. It is a clear, custodian-compatible foundation for shielded wallet and address generation that future MIPs can build on for receiving, viewing, proving, spending, and broader institutional shielded asset workflows.
