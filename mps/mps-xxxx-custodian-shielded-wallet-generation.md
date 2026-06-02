---

MPS: TBD
Title: Custodian-Compatible Shielded Wallet Generation
Category: Core
Status: Proposed
Authors:

* Jalal Hannan <Jalal-1>
* Hector Bulgarini <hbulgarini>
  Proposed Solutions: []
  Discussions: []
  Created: 2026-06-02
  License: CC-BY-4.0

---

## Abstract

Midnight shielded wallet generation does not currently map cleanly to the key-generation models used by institutional custodians.

Custodians such as BitGo, Fireblocks, and similar institutional providers normally create and manage wallets using MPC, TSS, HSMs, DKG, multisig, or other threshold-control systems. These systems are designed so that no single party or ordinary application service reconstructs the full private key or key-equivalent secret.

ZSwap shielded wallet generation is different. The public receiving key is not derived from a normal asymmetric public/private keypair in the way most custodians expect. Instead, ZSwap address derivation depends directly on secret material. This creates a blocker for institutional custody because the custodian may not be able to generate a shielded wallet or address without reconstructing sensitive shielded secret material somewhere inside its infrastructure.

This MPS defines the problem of generating Midnight shielded wallets in a way that is compatible with institutional custody security models.

## Problem Statement

Regulated custodians need to create shielded wallets for customers without exposing full shielded secret material outside approved custody boundaries.

In conventional custody integrations, a custodian can often generate a wallet from threshold key material, derive a public address, and keep signing authority protected inside MPC, TSS, HSM, or equivalent infrastructure. The public address can usually be derived from public key material without reconstructing a complete private key in ordinary application infrastructure.

ZSwap does not obviously fit this model. The ZSwap coin public key is derived from secret material using a hash operation. This means address generation depends directly on the shielded secret, rather than on a public key generated through a familiar asymmetric key process.

This creates a specific problem for custodians: if generating a shielded address requires the full ZSwap secret to exist in one place, then the custodian may be unable to support Midnight shielded wallets without violating its existing security, compliance, or key-management model.

The core question is:

**How can a custodian generate Midnight shielded wallets and addresses without reconstructing or exposing the full ZSwap secret outside its approved custody boundary?**

## Background

Institutional custody systems are usually built around threshold-controlled key management.

A custodian may use MPC, TSS, HSMs, multisig, policy engines, approval workflows, and recovery procedures to ensure that no single operator, server, or application process can unilaterally control customer assets. In these systems, wallet creation and address derivation are part of a tightly controlled key-management lifecycle.

Midnight shielded assets introduce a different model. ZSwap uses shielded notes or coins, and the receiving capability is tied to secret material that does not behave like a standard ECDSA, EdDSA, or Schnorr public/private keypair. In particular, if the shielded public receiving key is derived by hashing secret material, then the address-generation step itself may require access to data that custodians treat as key-equivalent.

This creates a mismatch between ZSwap wallet generation and the assumptions used by institutional custody systems.

## Core Problem

A custodian needs a way to generate a Midnight shielded wallet for a customer while preserving the following properties:

1. No ordinary application server should receive the full shielded secret.
2. No single custodian participant should be able to reconstruct the full shielded secret outside approved secure infrastructure.
3. Address generation should be compatible with institutional custody controls such as MPC, TSS, HSMs, DKG, policy approvals, and audit trails.
4. The resulting shielded wallet should be usable for future custody workflows, including receiving, viewing, proving, and spending, without requiring an unsafe wallet-generation workaround.
5. The model should be clear enough for custodians to assess whether the generated secret material is key-equivalent and whether it remains inside their approved custody boundary.

## Scope

This MPS is only about **shielded wallet and address generation**.

It covers:

* Generating ZSwap-compatible shielded wallet material.
* Deriving shielded receiving addresses or public receiving keys.
* Compatibility with MPC, TSS, HSM, DKG, multisig, or equivalent custody infrastructure.
* Avoiding unsafe reconstruction or exposure of full shielded secret material.
* Clarifying what material is generated, where it exists, and whether it is key-equivalent.
* Supporting institutional onboarding flows where a custodian creates one or many shielded wallets for customers.

## Non-Goals

This MPS does not address native shielded asset transfer.

It does not define how a custodian spends a ZSwap coin, generates a spend proof, manages nullifiers, constructs shielded transactions, funds DUST, executes Compact contract calls, or performs source-of-funds attribution.

It also does not prescribe a specific solution such as MPC hashing, HSM-resident generation, TEE generation, coSNARKs, protocol changes, or a specific custodian architecture. Those may be explored in future MIPs, but this MPS is limited to defining the wallet-generation problem.

## Representative Use Cases

### Use Case 1: Custodian creates a shielded wallet for a bank customer

A bank customer opts into a Midnight-backed product through a normal Web2 banking interface. Behind the scenes, the bank’s custodian must create a Midnight shielded wallet for that customer. The customer should not see seed phrases, wallet software, private keys, or blockchain transaction flows.

The custodian needs to generate the shielded wallet using its approved custody infrastructure and without exposing the full ZSwap secret to ordinary application services.

### Use Case 2: Custodian creates many shielded wallets at scale

An institutional custodian needs to create thousands or millions of customer shielded wallets. The wallet-generation process must be repeatable, auditable, recoverable, and compatible with the custodian’s existing security controls.

The custodian cannot rely on a manual or bespoke process where full shielded secrets are generated outside its approved custody environment.

### Use Case 3: Custodian derives a receiving address without spending authority leakage

A custodian needs to derive a shielded receiving address that can be safely shared with an issuer, counterparty, or internal treasury system. The address derivation process should not leak spending authority or expose key-equivalent material to systems that only need receiving information.

### Use Case 4: Custodian evaluates Midnight integration feasibility

Before supporting Midnight, a custodian’s security and risk teams need to understand whether shielded wallet generation can fit within their existing MPC, TSS, HSM, key-isolation, and audit architecture.

If the only available model requires full shielded secrets to be assembled in ordinary infrastructure, the custodian may reject support for native shielded assets.

## Goals

1. Ensure that shielded wallet generation is compatible with institutional custody architectures, including MPC, TSS, HSM, DKG, policy approval, and audit-control environments.
2. Ensure that custodians can generate ZSwap-compatible shielded wallets and receiving addresses without exposing full shielded secret material to ordinary application infrastructure.
3. Define the custody boundary requirements for shielded wallet generation, including where shielded secret material may be generated, derived, stored, or processed.
4. Support scalable and recoverable shielded wallet creation for institutional onboarding flows, including cases where a custodian must create wallets for many customers.
5. Provide future MIPs with a clear problem boundary for designing custodian-compatible shielded wallet generation, without conflating this work with native shielded transfers, proof generation, source-of-funds attribution, or contract execution.


## Open Questions

1. Can a ZSwap shielded address be generated without reconstructing the full shielded secret in one place?
2. Can the required hash operation over secret material be made compatible with MPC, TSS, HSM, DKG, or another approved custody model?
3. Which pieces of ZSwap wallet-generation material should be treated as key-equivalent by custodians?
4. Can shielded wallet generation be separated into receiving capability, viewing capability, and spending capability in a way that maps cleanly to custody controls?
5. What minimum protocol, SDK, wallet, or infrastructure changes would be required to support custodian-compatible shielded wallet generation?
6. Can custodians generate shielded wallets deterministically, recoverably, and audibly without weakening privacy or exposing customer-specific secret material?
7. What information must be documented for a custodian security team to approve a Midnight shielded wallet-generation flow?

## Expected Outcome

If this problem is addressed, institutional custodians will be able to create Midnight shielded wallets for customers without relying on unsafe secret reconstruction or bespoke off-chain workarounds.

This would unlock the first step of native shielded asset custody: allowing a custodian to onboard users into shielded asset products while preserving both Midnight’s privacy model and the custodian’s security model.

Solving this problem does not by itself solve shielded transfers, proof generation, source-of-funds, or contract execution. It only establishes the foundation: a custodian can safely create and manage shielded wallets as part of its normal institutional custody lifecycle.
