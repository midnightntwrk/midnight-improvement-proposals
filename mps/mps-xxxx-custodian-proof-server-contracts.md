---
MPS: xxxx
Title: Custodian-Compatible Proof Server Architecture for Contract Interactions
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

Midnight contract interaction introduces a custody execution problem that does not exist in conventional signing-only custody integrations.

In most institutional custody systems, the secure custody boundary is built around transaction approval and signing. A custodian constructs a transaction, applies policy, collects approvals, and produces the required signature through MPC, TSS, HSM, multisig, or equivalent infrastructure.

Compact contract calls on Midnight require more than signing. They also require proof generation. Even if a custodian uses a contract-based pattern for mint, burn, treasury control, multisig approval, or a restricted tokenized deposit flow, the transaction may still need to be proven before it can be submitted and verified on Midnight.

This creates a separate architectural blocker: the custodian needs to understand where proof generation happens, what material enters the proving environment, whether that material is key-equivalent or spend-authorizing, and whether the proof server must sit inside the custodian’s approved security boundary.

This MPS defines the problem of integrating Midnight proof generation for Compact contract interactions into institutional custody architecture.

## Problem Statement

A regulated custodian may be able to approve a Compact contract action using its normal policy and signing controls, but that approval is not sufficient to execute the action on Midnight.

Contract interaction on Midnight requires a proving step. Depending on the contract and transaction flow, the proof-generation environment may receive private inputs, witness data, transaction context, wallet-derived material, asset-specific data, or other sensitive information. Some of this material may be operationally sensitive. Some may be key-equivalent. Some may be spend-authorizing. Some may be safe to handle outside the custody boundary. Today, this classification is not clear enough for institutional custodians.

This creates an execution architecture gap.

The custodian needs to know whether the proof server is:

* ordinary application infrastructure,
* part of the secure custody boundary,
* a delegated service,
* a custodian-operated service,
* a confidential or isolated environment,
* or a protocol-critical component that requires a different trust model entirely.

The core question is:

**How can a custodian execute Compact contract calls when Midnight requires proof generation in addition to normal signing, approval, or policy workflows?**

## Why This Is a Separate Problem

This problem should not be collapsed into native shielded asset transfer.

Native ZSwap spending is about moving a shielded coin from a wallet-style account. That path requires a spend proof over private coin material.

This MPS is about Compact contract interaction. A custodian may be using a contract-based pattern precisely to avoid open-ended native shielded wallet transfers in an initial deployment. For example, a tokenized deposit product may use controlled mint and burn flows, treasury contracts, admin controls, or multisig-like contract logic.

Even in those restricted flows, the custodian still needs a proving architecture.

The distinction is:

* **Native shielded spend:** How does the custodian spend a ZSwap coin?
* **Contract interaction:** How does the custodian execute a Compact contract call that requires proof generation?
* **Custody approval:** How does the custodian decide the action is allowed?
* **Proof generation:** How does Midnight prove the action is valid?

This MPS is about the contract proof-generation path and how it fits into custody architecture.

## Core Problem

Custodians need a Midnight contract execution model that preserves the following properties:

1. Contract calls can be initiated through institutional policy and approval workflows.
2. Proof generation is clearly integrated into the transaction execution path.
3. The proof server’s trust boundary is explicit.
4. The data entering the proof server is clearly classified.
5. Key-equivalent or spend-authorizing material is not exposed to ordinary application infrastructure.
6. Custodians can determine whether the proof server must be operated internally, isolated, attested, delegated, or integrated through another approved model.
7. Contract-based MVP flows do not rely on unsafe or undefined proof-generation assumptions.
8. The model supports regulated use cases such as mint, burn, redemption, treasury movement, admin actions, and multisig-controlled contract calls.

## Scope

This MPS covers proof generation for **Compact contract interactions in custodial environments**.

It includes:

* Contract calls that require proof generation.
* Mint, burn, redemption, treasury, admin, and multisig-style contract flows.
* Custodian policy approval combined with Midnight proof generation.
* Data classification for proof-server inputs and outputs.
* Determining whether proof-generation inputs are private, sensitive, key-equivalent, or spend-authorizing.
* Defining the required trust boundary for the proof server.
* Understanding whether a proof server can be operated by the custodian, delegated to a third party, or isolated through a secure execution environment.
* Supporting restricted institutional MVP flows where contracts are used instead of general-purpose native shielded transfers.

## Non-Goals

This MPS does not define shielded wallet generation.

It does not define native ZSwap shielded spends or wallet-style asset transfers.

It does not define source-of-funds attribution for incoming shielded assets.

It does not define asset-level business logic such as freeze, force transfer, holder eligibility, or updatable transfer restrictions.

It does not prescribe a specific proving architecture such as TEE, HSM-adjacent proving, coSNARKs, collaborative proving, proof-share aggregation, remote proving, local proving, or protocol changes.

It does not assume that every proof-server input is key-equivalent. The purpose of this MPS is to make that classification explicit so custodians can assess the risk correctly.

## Representative Use Cases

### Use Case 1: Custodian executes a mint flow for a tokenized deposit

A bank customer opts into a tokenized deposit product through a normal Web2 banking interface. The custodian or banking platform needs to trigger a Compact contract call that mints or records the corresponding shielded asset position.

The custodian may approve the action through normal policy controls, but the transaction still requires proof generation before it can be submitted to Midnight.

The custodian needs to know what data the proof server receives and whether that server must sit inside the custody boundary.

### Use Case 2: Custodian executes a burn or redemption flow

A customer exits a product and the custodian needs to trigger a burn, redemption, or withdrawal-related contract call.

This may not be a general-purpose native shielded wallet transfer, but it still requires a valid Midnight transaction and proof. The custodian needs a safe execution path that does not expose sensitive witness or customer material to ordinary backend services.

### Use Case 3: Custodian interacts with a treasury or admin contract

An institution uses a Compact contract for treasury operations, asset administration, issuer controls, or operational permissions.

The custodian’s policy engine can decide whether an action is allowed, but Midnight execution still depends on proof generation. The custodian needs to integrate the proof-generation step into its policy, approval, audit, and submission pipeline.

### Use Case 4: Contract-based MVP avoids open shielded transfers

An early institutional product avoids general-purpose shielded wallet transfers and only supports controlled mint, hold, and burn flows.

This reduces the scope of native shielded custody, but does not remove the need for proof generation. The custodian still needs a proof-server architecture that is acceptable to its security and compliance teams.

### Use Case 5: Custodian risk team reviews Midnight integration

A custodian’s security team reviews Midnight support and asks:

* What does the proof server receive?
* Can it authorize asset movement?
* Does it ever see key-equivalent material?
* Can it be operated outside secure custody infrastructure?
* What happens if it is compromised?
* What audit evidence proves that a contract action was approved and executed correctly?

Without clear answers, the custodian may reject the integration or restrict Midnight support to non-shielded assets.

## Goals

1. Ensure that Compact contract interactions can be executed through custody-compatible architecture.

2. Define how proof generation fits into the custodian’s existing transaction construction, policy approval, signing, and submission workflow.

3. Ensure that proof-server inputs are clearly classified as public, private, sensitive, key-equivalent, spend-authorizing, or operational-only.

4. Ensure that any key-equivalent or spend-authorizing material required for contract proof generation remains inside an appropriate custody security boundary.

5. Support restricted institutional flows such as mint, burn, redemption, treasury, admin, and multisig-style contract calls without requiring unsafe proof-server assumptions.

6. Provide custodians with enough architectural clarity to decide whether the proof server can be operated internally, isolated, attested, delegated, or integrated through another approved model.

7. Provide future MIPs with a clear problem boundary for custodian-compatible Compact proof generation, without conflating this work with native shielded spends, wallet generation, source-of-funds attribution, or asset-level business logic.

## Open Questions

1. What exact data is required by the proof server for each class of Compact contract interaction?

2. Which proof-server inputs are public, private, sensitive, key-equivalent, or spend-authorizing?

3. Does the proof server need access to wallet-derived material, private inputs, contract witness data, asset state, customer state, or transaction context?

4. Can a custodian safely operate the proof server outside its secure custody boundary for any contract flows?

5. Which contract flows require the proof server to be inside the custodian’s approved security boundary?

6. Can proof generation be separated from policy approval and signing without weakening custody controls?

7. What audit evidence should link custodian approval, proof generation, transaction construction, signing, and submission?

8. How should DUST funding, transaction balancing, and fee execution interact with the proof-generation flow?

9. Can a restricted mint/burn MVP use a simpler proof-server trust model than general-purpose native shielded custody?

10. Which parts of this problem require protocol changes, and which can be solved through SDKs, wallet tooling, custody integration patterns, deployment guidance, or infrastructure hardening?

11. What minimum documentation would a custodian security team need to approve a Midnight contract proof-server deployment?

## Expected Outcome

If this problem is addressed, custodians will be able to execute Midnight Compact contract calls through a clearly defined and custody-compatible proof-generation architecture.

A custodian should be able to approve an action, generate the required proof, sign or authorize the transaction where needed, submit it to Midnight, and retain audit evidence without exposing sensitive witness or key-equivalent material to ordinary application infrastructure.

This would make contract-based institutional flows more viable, including tokenized deposit mint/burn, issuer treasury operations, controlled redemption flows, admin actions, and multisig-style contract interactions.

Solving this problem does not by itself solve native ZSwap wallet transfers, shielded wallet generation, source-of-funds attribution, or regulated asset business logic. It addresses one specific execution-layer question: how a custodian safely integrates Midnight proof generation into Compact contract workflows.
