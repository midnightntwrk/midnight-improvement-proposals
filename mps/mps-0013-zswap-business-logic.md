---
MPS: "0013"
Title: zswap business logic
Authors: Karmel E - karmoola
Status: Proposed  
Category: Core   
Created: 29-05-2026  
Requires: [none]  
Replaces: [none]  
MIP: none  

---
# MPS: business logic for zswap tokens

## **Abstract**

ZSwap tokens give Midnight a powerful privacy primitive by allowing assets to exist and move as shielded notes, but they do not currently provide a general-purpose way to attach or enforce business logic across the lifecycle of those tokens. Once minted, a ZSwap token is largely limited to being held, sent, or received; it cannot natively express rules about how it should behave after issuance, what conditions must be satisfied before it moves, who or what is allowed to control it, or how its state should change over time. This makes ZSwap tokens unsuitable for any use case that requires more than simple private transfer. The core problem this MPS defines is that Midnight has private shielded tokens, but not yet programmable shielded tokens.

## **Vision**

An issuer of a tokenized RWA on Midnight has the same set of asset-level controls available on permissioned-token chains today, with privacy-preserving holdings and transfers added. Holders' balances and transfer history remain shielded; issuers, regulators, and disclosed auditors see what they are entitled to.

## **Problem**

Without a way to express lifecycle business logic, ZSwap tokens are limited to private movement of value rather than private programmable value. Builders can use them for simple send-and-receive flows, but not for products where an asset must be locked, released, redeemed, retired, restricted, updated, recovered, governed by a protocol, or made subject to rules after issuance. This creates a significant platform-level limitation: the stronger Midnight's privacy model is, the harder it becomes to build meaningful asset logic unless that logic can also operate against shielded state.

The reference for the asset-level shape of these controls on other chains is ERC-3643 which has been adopted by issuers on Polygon, Avalanche, Ethereum, and Hedera. Six gaps prevent equivalent issuance on Midnight.

**Mint under issuer authority.** Issuers create new units as new off-chain backing arrives: fund subscriptions, deposits, originated loans. There is no native mechanism for authorized mint with a verifiable audit trail at the asset level on Midnight.

**Burn against off-chain redemption.** When the issuer honors a redemption, the corresponding on-chain unit must be destroyed so on-chain supply tracks the underlying. Sending units to inaccessible addresses or parking them in an issuer wallet distorts supply metrics and fails audit.

**Freeze on legal or regulatory action.** Issuers receive court orders, OFAC SDN designations, and law-enforcement requests with non-negotiable response windows (OFAC requires action on designation effectively immediately). There is no native way to immobilize the relevant holding at the asset level.

**Involuntary reassignment (force transfer) under issuer authority.** Freeze alone is operationally incomplete and metrics-incorrect. Frozen units remain in total supply, so a wallet with lost keys or a sanctioned holder produces a permanent mismatch between on-chain supply and off-chain backing exactly the supply-versus-reporting problem flagged in scoping. Force transfer resolves it by allowing the issuer to reassign or retire the units back into reconciliation. *Assumption: only the issuer can perform this action. A public event is mandatory because force transfer is the only mechanism that closes the supply mismatch, and any reconciliation that is not publicly verifiable defeats its own purpose. The event must be public even where amount and counterparty are private.*

**Holder eligibility and verified-counterparty transfer.** Many RWAs can only be held by parties meeting jurisdiction, accreditation, or AML criteria (Reg D in the US, MiFID professional-investor categories in the EU, equivalent rules elsewhere). Off-chain whitelist enforcement breaks at secondary transfer because the chain has no notion of holder eligibility. Some instruments require both counterparties be verified at the moment of transfer, which is a stronger property than holder-side gating. Both must be per-asset opt-in; most assets on Midnight should remain permissionless. This is the function area ERC-3643 addresses through on-chain identity and an identity registry.

**Updatable transfer restrictions.** Regulation changes, permitted jurisdictions are added and removed, lockups expire, accreditation criteria evolve. Issuers must update the rules governing their asset without redeploying and forcing holder migration.

compliance enforcement must work against shielded state, and supply must be auditable without revealing individual balances. This is the central architectural problem this MPS is naming. **without a shared baseline, every issuer reimplements equivalent controls in bespoke contract code and re-audits them.**

## **Current Midnight Tech Stack Limitations**

The current Midnight architecture does not yet support the regulated requirements described above for the following technical limitations:

- **ZSwap Ledger tokens lack asset-level spend controls.** Pure ledger tokens do not support custom spending logic sufficient to implement issuer controls such as freeze, verified-counterparty transfer, and other restrictions.
- **Compact-contract enforcement is bypassable today.** While ledger tokens can be managed via compact contracts, UTXOs do not currently carry an unforgeable spend condition. As a result, any additional checks implemented in a contract can be bypassed by spending via ZSwap directly with the coin secret keys.

**Use Cases**

**Tokenized money market fund.** Issuer needs mint on subscription, burn on redemption, freeze on sanctions, reassignment on death or court order, holding restricted to accredited investors in permitted jurisdictions, and updatable jurisdiction lists. Cannot launch on Midnight today.

**Stablecoin issued natively on Midnight.** Distinct from a bridged pattern where some controls live on a bridge contract on another chain. A native issuer needs all of the above on Midnight itself.

## **Goals**

1. Close each of the six gaps above so regulated RWA issuers can launch on Midnight.
2. Preserve Midnight's privacy guarantees: holder identities, balances, and amounts remain shielded by default; controls must not require these to become public to function.
3. Maintain audit integrity: every privileged issuer action emits a verifiable public event sufficient for auditors and regulators to reconstruct the action, even where amount and counterparty are private; on-chain supply must reconcile to off-chain backing.
4. Make rules updatable by the issuer without redeployment or holder migration.
5. Reduce per-issuer compliance audit surface by addressing these controls at a shared level rather than leaving each issuer to reimplement.
6. A regulated-asset issuer can natively perform every lifecycle control required to operate a compliant instrument on Midnight (mint, burn, freeze, seize, lost key recovery, holder eligibility, etc) without bespoke per-asset cryptographic engineering.

## Expected Outcomes

1. Regulated asset classes become launchable
2. Lower cost and faster time-to-launch for compliant assets
3. The broader builder ecosystem (developers who want to build non-privacy-preserving dapps) is unaffected

## Recommended MIPs

A single MIP is unlikely to address all the goals this MPS states. This MPS recommends a series of MIPs that lift the technical limitations one at a time and coordinate with solutions to related Problem Statements, such as [MPS-0035: Shielded Spend Authorization Requires Exposing the Spend Key](./mps-0035-shielded-spend-key-exposure.md).

### Token-based custom spend logic

Designs in full how custom logic is attached to a ZSwap spend through recursive proof verification, so a coin carries a spend rule that spending through ZSwap directly with the coin secret key cannot bypass. Addresses the second limitation stated in [Current Midnight Tech Stack Limitations](#current-midnight-tech-stack-limitations), and the first in part: freeze and updatable restrictions need issuer-maintained state, which is left to the stateful gate proof verification follow-up.

It should also outline the further mechanisms this MPS needs to meet all its goals. Every network upgrade is a hard fork, which costs users and wallets an update. Outlining the later designs up front lets the first upgrade include placeholders that later upgrades make fully functional, so design and delivery can be staged while keeping the number and impact of hard forks low.

The MIP should specify:

- **The rule bound to the token type.** How a spend-rule descriptor is bound to the token type, with coins that carry no rule left unaffected.
- **Pure-circuit rules in full.** How the spend circuit recursively verifies a gate proof (a proof that the coin's rule is satisfied) bound to the spend and enforces the rule bound to the coin's token type, so a spender cannot substitute a different rule.
- **Contract-enforced rules as a future extension.** The extension's dependencies (a proved shielded spender, the Capsule Runtime MIP) and its privacy costs.
- **A goal-by-goal table against this MPS.** For each gap, goal, and expected outcome: delivered by this MIP, or deferred to a named follow-up MIP with its dependencies.
- **The ZSwap changes it needs.** The rule descriptor and spend-proof inputs, written so they can be bundled with the spend-authorization changes for MPS-0035 into one network upgrade.

It should be accompanied by a working prototype, built on the midnight-zk recursion work, in a public repository with tests running in CI, showing:

1. **Bypass prevented.** A coin with a rule cannot be spent without a valid gate proof: a negative test shows that such a spend fails verification.
2. **Eligibility gate.** A spend succeeds only when the spender proves possession of a credential signed by the issuer's key; a spend without it fails.
3. **Existing tokens unchanged.** For coins with no rule, the derived token type matches the one the current ledger implementation derives.
4. **Benchmark report.** Proving time and circuit size for a spend with no rule and a spend with a rule, on stated reference hardware. The report has no pass/fail target; it informs the decision on that bundling.

### Follow-ups and coordinated designs

Once spend rules cannot be bypassed, the following designs can address the goals the Token-based custom spend logic MIP leaves open.

Expected follow-ups and coordinated designs include:

- Contract-enforced rules — lets a contract call guard token spends, making the expressiveness of contract calls available to spend rules, at some cost in privacy.
- Stateful gate proof verification — lets gate proofs read issuer-maintained state such as freeze lists and updatable restrictions.
- Spend authorization ([MPS-0035: Shielded Spend Authorization Requires Exposing the Spend Key](./mps-0035-shielded-spend-key-exposure.md)) — also changes how spends are authorized, so solutions to MPS-0035 and to this MPS need to take each other into account.
- Sender-based spend logic — lets the wallet customize spend authorization for different styles of key management: multi-signature, single signature, or other means.

## References

- [MPS-0014: Proof Verification in Compact](./mps-0014-proof-verification-recursion.md) — the in-circuit proof verification the Token-based custom spend logic MIP builds on.
- [MPS-0021: Phase 2- Contract 2 Contract](./mps-0021-phase2-contract-to-contract.md) — the contract-to-contract calls that contract-enforced rules build on.
- [MPS-0023: Standard Interface for Privacy-Preserving Real World Assets](./mps-0023-rwa-standard-interface.md) — asks for the same issuer controls (freeze, clawback, identity verification), framed on ERC-7943.
- [MPS-0035: Shielded Spend Authorization Requires Exposing the Spend Key](./mps-0035-shielded-spend-key-exposure.md) — spend authorization.
- [MIP-0017: Signature-Authorized Shielded Spends with VRF Nullifiers](../mips/mip-0017-shielded-spend-auth/mip-0017-shielded-spend-auth.md) — designs the signature-authorized shielded spend and its note format.
- [MIP-0011: Native Shielded Token Standard](../mips/mip-0011-native-shielded-token.md) — a shielded token standard that defers post-issuance controls to this MPS.
