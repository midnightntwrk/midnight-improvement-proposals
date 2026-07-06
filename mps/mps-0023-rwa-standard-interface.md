---
MPS: "0023"  
Title: Standard Interface for Privacy-Preserving Real World Assets  
Authors: Muhammad Zidan Fatonie (mzf11125)
Status: Proposed  
Category: Standards  
Created: 11-Jun-2026  
Requires: none  
Replaces: none  

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

Institutional adoption of Real World Assets (RWAs) such as private equity, real estate, and bonds on public blockchains is blocked by the compliance versus privacy dilemma. Standards such as ERC-7943 (Universal RWA Interface) provide comprehensive compliance hooks including identity verification, balance freezing, and asset clawback, but executing these on a transparent ledger exposes the cap table, user balances, and whitelist status to all observers. This violates data protection regulations and exposes institutional trading strategies.

Midnight Network can resolve this tension through shielded state and zero-knowledge proofs, but no standard interface currently exists for issuers to deploy compliant RWAs, wallets to integrate them, or DEXs to trade them. Each issuer builds bespoke compliance logic, fragmentation accelerates, and the ecosystem misses a generational opportunity to become the privacy-first home for tokenized real world assets.

This MPS identifies the absence of a standard, privacy-preserving RWA token interface for the Midnight ecosystem. It defines the problem domain so that future MIPs can specify a uniform Compact interface, ZK-native compliance hooks, and administrative controls that issuers, wallets, DEXs, and regulators can adopt uniformly.

## Vision

An issuer deploys a single RWA token contract on Midnight using a standard interface. Wallets recognise the contract, know to prompt the user for the correct ZK proof of accreditation or KYC status, and execute transfers without exposing the user's identity or balance to the public ledger. The issuer maintains regulatory control including the ability to freeze specific positions or claw back assets under court order, all without revealing the shielded cap table. DEXs route trades against compliant RWA tokens through a predictable circuit interface. Regulated institutions participate because the standard satisfies both data privacy law and compliance requirements simultaneously.

## Problem

The current state of RWA tokenisation on Midnight presents four compounding problems.

**1. No standard interface for RWA tokens.** Every issuer building an RWA token on Midnight writes their own Compact contract with custom compliance hooks, balance models, and administrative controls. Wallets cannot integrate generically because there is no predictable function signature to query or invoke. DEXs cannot route trades because there is no standard circuit for compliance-checked transfers. This fragmentation prevents the emergence of an RWA ecosystem.

**2. The privacy versus compliance tension is unresolved.** ERC-7943 demonstrated that standardised compliance hooks are necessary for institutional adoption, but its transparent execution model exposes data that many RWA issuers cannot afford to reveal. On Midnight, the tension is both resolvable and unresolved. There is no agreed pattern for how a circuit should verify a KYC or accreditation credential without publishing the user's identity. There is no standard for how an issuer can selectively view balances or freeze positions without accessing all shielded state. Each contract reinvents these mechanisms independently.

**3. Wallets lack a predictable interface for compliance proofs.** Wallets on Midnight are capable of generating ZK proofs, but they do not know what proof a given token contract expects. A wallet cannot generically prompt a user to generate an accreditation proof if the circuit format, credential schema, and verification key differ per issuer. Without a standard, every wallet integration is bespoke, creating an adoption barrier for both wallet providers and token issuers.

**4. Cross-contract composability is blocked.** RWA tokens cannot be traded on DEXs, used as collateral in lending protocols, or composed into broader DeFi workflows without a standard interface that other contracts can predictably call. Each contract's unique compliance model means every integration must be manually negotiated and implemented.

## Use Cases

**Use Case 1: Issuer deploys a compliant RWA token.** A regulated asset manager tokenises a private equity fund on Midnight. They deploy a Compact contract that implements the standard RWA interface. The contract enforces that only accredited investors can hold or transfer the token. Investors submit ZK proofs of accreditation rather than publishing their legal names or net worth. The issuer retains the ability to freeze addresses under sanctions directives and claw back assets following a court order.

**Use Case 2: Wallet integrates RWA tokens generically.** A Midnight wallet detects that a token contract implements the standard RWA interface. When the user attempts to transfer tokens, the wallet reads the contract's required compliance proof type and prompts the user to generate the correct ZK credential. The wallet never exposes the user's private data. The integration works for any compliant RWA token without per-issuer customisation.

**Use Case 3: Regulator enforces a freeze without exposing the cap table.** A regulatory authority determines that a specific position must be frozen. The issuer invokes a freeze operation against the shielded identifier of the target. The operation succeeds without revealing the full list of holders, individual balances, or the relationship between the frozen position and other holders. The freeze is auditable via a public event that does not leak shielded data.

**Use Case 4: DEX routes a shielded RWA trade.** A DEX smart contract calls the standard transfer circuit of an RWA token. The circuit verifies the sender's compliance status, executes the shielded transfer, and updates balances. The DEX does not need to understand the issuer-specific compliance logic because the standard interface abstracts it. The trade completes with the same privacy guarantees as a direct transfer.

**Use Case 5: RWA token used as lending collateral.** A lending protocol accepts an RWA token as collateral through the standard interface. The protocol verifies the user's compliance status indirectly through a circuit call and records the collateral position without observing the user's shielded balance history. Liquidation follows the same standardised compliance path.

## Goals

1. **Standard Compact interface for RWA tokens.** A single set of circuit and function signatures that issuers implement, wallets detect, and DEXs consume. The interface must cover transfer, balance query, mint, and burn operations.

2. **ZK-native compliance hooks.** Compliance checks such as KYC verification, accreditation status, and sanction screening must be expressible as ZK proof requirements rather than public registry lookups. The standard must define where proofs enter the circuit and how they are verified.

3. **Administrative controls with privacy safeguards.** The interface must support issuer-controlled freeze and clawback operations without exposing the shielded cap table. These operations must emit audit events that confirm execution without leaking user identities or balances.

4. **Wallet discoverability.** Wallets must be able to detect that a token contract implements the RWA standard and determine what compliance proofs are required for a given operation, enabling generic integration.

5. **Privacy preservation.** User identities, individual balances, and holder relationships must remain shielded from public observation. The issuer should have selective visibility into holdings under their own tokens without accessing unrelated shielded state.

6. **Backward compatibility with non-shielded interoperability.** Where possible, the standard should compose with existing Midnight token standards and indexer patterns so that the broader ecosystem can integrate RWA tokens without dedicated support.

7. **Regulatory audit trail.** Administrative actions such as mints, burns, freezes, and clawbacks must produce reliably observable events for audit purposes without violating user privacy.

## Non-Goals

This MPS does not prescribe a specific ZK credential format, cryptographic signature scheme, or verification key registry. It does not specify the on-chain representation of KYC or accreditation data. It does not define the economics of RWA tokens or mandate any particular compliance framework. The purpose is to define the problem clearly so that future MIPs can evaluate competing solutions.

## Expected Outcomes

If this problem is addressed, the Midnight ecosystem will have a standard RWA token interface that wallets, DEXs, and lending protocols can integrate once and reuse across all compliant assets. Issuers will deploy tokens without building bespoke compliance infrastructure. Regulated institutions will have a clear, privacy-preserving path to tokenise assets on Midnight. The standard will remove the fragmentation that currently prevents RWA composability and will position Midnight as the leading network for privacy-preserving real world asset tokenisation.

## Open Questions

1. **Credential format.** What format should ZK compliance credentials use? Options include verifiable credentials with BBS+ signatures, anonymous credentials, or a custom Compact circuit pattern. The choice affects proof size, circuit complexity, and wallet integration burden.

2. **Compliance proof discovery.** How does a wallet determine what proof a token contract requires? Should the proof requirement be encoded in a public ledger field, a circuit introspection mechanism, or an off-chain registry?

3. **Freeze granularity.** Should freeze operate per wallet address, per specific balance commitment, or per compliance credential? Each choice has different implications for issuer control and user privacy.

4. **Clawback invariants.** What constraints should prevent issuer abuse of force transfer? Should clawback emit a mandatory public event? Should clawbacks require a multi-signature or time lock?

5. **Issuer viewing capability.** What mechanism allows an issuer to view balances under their own token? Should viewing be all-or-nothing, or can an issuer delegate view capability per position?

6. **Composability surface.** Which circuits should be callable from other contracts? Should the standard define a separate composability circuit that bypasses user-facing compliance prompts while preserving issuer controls?

7. **Event standard for shielded admin actions.** What fields should freeze and clawback events emit to satisfy auditability without leaking shielded data?

8. **Migration and versioning.** How should the standard handle upgrades to compliance logic or credential formats without breaking existing token holders or requiring mass migration?

## Recommended MIPs

The following MIP areas are recommended to address the problems above.

- **MIP: Standard RWA Compact Interface.** Specifies the required circuit and function signatures for RWA tokens on Midnight, including transfer, balance query, mint, burn, freeze, and clawback operations. Defines the ledger state layout expected by wallets and DEXs.

- **MIP: ZK Compliance Hook Specification.** Defines how compliance proofs enter the standard transfer circuit, what credential format the circuit accepts, and how the verification key is resolved. Must be compatible with multiple credential schemes.

- **MIP: RWA Token Event Standard.** Specifies what events are emitted for administrative actions and transfers, what fields each event includes, and how events preserve privacy while enabling auditability.

- **MIP: Wallet Integration Guide for RWA Tokens.** Specifies how wallets detect RWA token contracts, determine required compliance proofs, and prompt users accordingly. Should include reference implementation patterns for common wallet architectures.

## References

- ERC-7943: Universal RWA Interface (Ethereum) — prior art for standardised RWA compliance hooks on transparent ledgers.
- MPS-0006: Institutional Custody for Native Shielded Assets — related custody problem space for shielded tokens.
- MPS-0018: Multi-key Account Custody — related work on account models for Midnight-native assets.
- Midnight Compact Language Specification: docs.midnight.network/compact
- Midnight Security Patterns: midnightntwrk/midnight-agent-skills

## Acknowledgements

The author acknowledges the ERC-7943 authors and the Ethereum Magicians community for establishing the foundational RWA compliance interface, and the Midnight Foundation and ecosystem contributors for their ongoing work on shielded asset standards.

## Copyright

This MPS is licensed under CC-BY-4.0.
