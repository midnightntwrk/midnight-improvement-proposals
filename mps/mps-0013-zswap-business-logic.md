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

### Root cause

1. **Zswap ledger tokens lack asset-level spend controls.** Ledger tokens cannot carry spend logic sufficient for issuer controls such as freeze, pause or verified-counterparty transfer. Shielded inputs and outputs do not reveal their asset, so any asset-level rule changes how every shielded spend and output is validated.
2. **Compact-contract enforcement is bypassable.** No spend condition is attached to the asset itself; every condition the network enforces belongs to the coin’s current owner. A coin held by a contract moves only under that contract’s logic, but once a coin is held under a user’s key, any check a contract implements can be bypassed by spending it through Zswap directly.
3. **Only the key holder can cancel a coin.** A coin held under a user’s key can be spent only by a party holding that key, which today includes any proving service the wallet sends the key to. The network’s only record that the coin is spent is a marker computed with that key. There is no issuer-authorised way to cancel such a coin, which is what seizure and lost-key recovery require.
4. **The spender of a shielded coin cannot be proved to a rule.** A rule that depends on who is spending (eligibility, blocklists) has nothing to evaluate against when the input is held under a user’s key: the network sees neither its owner nor, in a balanced transfer, its asset.
5. **Rules cannot be attached after issuance.** Tokens issued today have no place for a rule to live, so an issuer that launches now may be unable to add controls later without moving every holder to a new token, and an issuing contract that nobody can update has no party able to adopt them. The issuer generally cannot identify holdings that exist before controls are added, and for one held under a user’s key the network can confirm it is unspent only with that key.

### Gaps

| Gap | Description |
| --- | --- |
| Permissionless circulation under issuer authority | The token moves freely, including into and out of DeFi contracts and to holders who never onboard with the issuer. Holders control their own wallet keys. The issuer retains authority over its token wherever it is held. Today these properties cannot coexist: a coin held under its holder’s own key escapes any contract’s restrictions, and a coin held by a contract is restricted only because the contract has custody of it. |
| Freeze | Issuers receive court orders, sanctions designations and law-enforcement requests with non-negotiable response windows (OFAC requires action on designation effectively immediately). There is no native way to immobilise a specific holding at the asset level. |
| Pause | Issuers need to halt movement of their token at two levels: a single wallet or holding, and the whole token (for example during an incident, a corporate action or a regulatory instruction). Neither exists. |
| Force transfer (seizure) | Freeze alone leaves frozen units in total supply, so a sanctioned holder or a lost key produces a permanent mismatch between on-chain supply and off-chain backing. Force transfer lets the issuer reassign or retire units back into reconciliation, including units held by holders who never onboarded. Only the issuer can perform it. Holdings held under a user’s key that predate the asset adopting controls can be frozen but not force-transferred with a supply guarantee, so the issuer still needs a remedy for them. |
| Issuer visibility | The issuer needs to see balances of its token across all holders, including holders who never onboarded and never voluntarily shared viewing access, for holdings created after the asset adopts controls and for older holdings once they move. Today an issuer learns the balance of a holder that holds under its own key only if the holder discloses it. |
| Issuance and redemption under issuer authority | Issuers create units as backing arrives and destroy them when redemptions are honoured, so on-chain supply tracks the underlying. Sending units to inaccessible addresses or parking them in an issuer wallet distorts supply and fails audit, and today any holder can also destroy units without the issuer. Continuous mint and redeem is preferred; a treasury-based model (pre-minted supply released from and returned to an issuer treasury) is acceptable. |
| Threshold approval for privileged actions | Every privileged issuer operation (mint, burn, freeze, unfreeze, pause, resume, seize, adopting controls, rule changes including eligibility-list changes, granting viewing access, recovery) must require a configured approval threshold rather than a single key. MPC or an audited contract multisig are both acceptable approaches. |
| Holder eligibility (permissioned profile) | Some instruments may only be held by parties meeting jurisdiction, accreditation or AML criteria. For these, issuance and holding are restricted to whitelisted holders. Off-chain whitelists break at secondary transfer because the chain has no notion of holder eligibility. Eligibility must be evaluated against the actual spender and recipient of a shielded coin, not a self-asserted party. |
| Controlled granting of viewing access | Granting viewing access to a third party (auditor, regulator, custodian) should require two-person approval where technically feasible. |
| Verified-counterparty transfer | Some instruments require both counterparties to be verified at the moment of transfer, a stronger property than holder-side gating. |
| Updatable rules without holder migration | Regulations change, jurisdictions are added and removed, lockups expire. Issuers must update the rules governing their asset without redeploying it and forcing holders to migrate. |
| Lost-key recovery | A holder who loses keys to a regulated holding needs an issuer-mediated path to restore it, consistent with the issuer’s obligations to that holder. A holder may arrange this in advance; otherwise it relies on force transfer or the remedy for older holdings. |
| Public audit event per privileged action | Every privileged action emits a public, verifiable event sufficient for auditors and regulators to verify it, even where amount and counterparty are private. Force transfer in particular must be publicly verifiable, because it is the mechanism that closes the supply mismatch: the ledger itself, not only the issuer, must ensure that a force-transferred holding can never again be spent and that a holding already spent cannot be force-transferred. |

## Use Cases

**Tokenized asset with permissionless secondary circulation.** A tokenization platform issues a regulated token that it wants to trade freely on Midnight DeFi. Many holders will acquire the token from a pool and never onboard with the issuer. The issuer must still be able to freeze, pause or seize any holding, see balances across all holders, mint and redeem against backing, and require a threshold of approvers for each freeze, pause, seizure, mint and redemption. Holders keep their own keys. Cannot launch on Midnight today.

**Tokenized money market fund.** The issuer needs mint on subscription, burn on redemption, freeze on sanctions, reassignment on death or court order, holding restricted to accredited investors in permitted jurisdictions, and updatable jurisdiction lists. Cannot launch on Midnight today with holders keeping their own keys.

**Stablecoin issued natively on Midnight.** A native issuer needs blocklisting, pause, authorised mint and burn, and the ability to destroy frozen funds. This may be a lighter profile than the permissioned profile.

## Goals

In priority order.

1. **Enable regulated issuance with permissionless secondary circulation.** A regulated issuer can launch a token that circulates freely while it keeps visibility of the token and retains freeze, pause, force transfer, issuance and redemption controls, each under threshold approval.
2. **Preserve Midnight’s privacy guarantees.** The identities and balances of holders who hold under their own keys, and transfer amounts, remain hidden from the public by default. Controls must not require them to become public, or reveal which asset a coin belongs to or whether it belongs to an issuer-controlled asset. A party that sent a holding held under a user’s key must not learn when it is spent, unless it is the issuer, the holder tells it, or the holding is spent in the transaction that created it. What the issuer can learn, including when a holding is spent and which spends are linked, and what a holder’s proving service learns because of the controls, is declared at issuance or adoption and whenever it changes, and the issuer learns no more than declared.
3. **Support permissioned instruments.** Holder eligibility and controlled granting of viewing access for instruments that require them.
4. **Maintain audit integrity.** On-chain supply reconciles to off-chain backing, and privileged actions are publicly verifiable.
5. **Close the remaining baseline gaps:** verified-counterparty transfer, updatable rules without holder migration, and lost-key recovery.
6. **Reduce per-issuer compliance audit surface** by providing these controls at a shared level, so no issuer needs bespoke per-asset cryptographic engineering.

## Expected Outcomes

1. Regulated asset classes, including stablecoins and tokenized funds, become launchable on Midnight with permissionless secondary circulation.
2. Lower cost and faster time to launch for compliant assets.
3. Issuers can launch today knowing their assets will be able to adopt controls later, provided they keep an authority able to adopt them, such as one that can update the asset’s minting contract. Freeze and pause then reach every holding, including older ones, and pausing the token needs no holding to be identified; force transfer reaches every holding except older ones held under a user’s key, and issuer visibility reaches older holdings once they move. An issuer that needs every power over every unit adopts controls before issuing.
4. Assets that do not adopt controls, and the builders who use them, keep their rules and privacy. Because a shielded spend or output does not reveal its asset, the coordinated network upgrades that deliver controls may add a bounded proving, size, ledger-state and fee cost to every spend and output, and must not shorten proof, offer or swap lifetimes beyond a stated bound. Wallets, provers and indexers update with each upgrade, and contracts that pay out an asset may need to update before it adopts controls.

## Acceptance Criteria

The problem is solved when each criterion below can be demonstrated on a shielded asset without bespoke per-asset cryptographic engineering. This MPS stages the criteria in three tiers. A holding is one shielded coin of the asset. An asset adopts controls when its issuer places it under these controls, before issuing or later, and is then issuer-controlled.

The criteria must hold in every note format the network accepts, including the one introduced by MIP-0017, the proposed solution to MPS-0035; where these criteria name MIP-0017, they mean whichever solution to MPS-0035 the network adopts. A solution states how it composes with MIP-0017, how many coordinated network upgrades it needs, and the bounds in Expected Outcome 4. It must not require holders to move holdings to a further note format, and should not require receiving addresses other than today’s and the one MIP-0017 introduces. The criteria depend on choices MIP-0017 fixes at activation: its receiving-address format, the operations performed with a holder’s spending key, which records may consume a coin, and whether existing notes can be made subject to new validity conditions. These must be settled, consistently with these criteria, before MIP-0017 activates.

### Tier 0: non-bypassable spend rules

The foundation the other tiers build on. It is not enough for a regulated launch on its own: a rule at this tier is checked against the spend alone, so it cannot read issuer-maintained state such as a freeze list.

1. **Bypass prevented.** An issuer can attach a spend rule to an asset. A spend of a coin that does not satisfy the asset's rule is rejected by the network, including when the spender presents the coin as carrying no rule or a different rule.
2. **Eligibility gate.** A rule can require the spender to prove possession of a credential signed by the issuer’s key; a spend without it is rejected.
3. **Existing tokens unchanged.** Assets that carry no rule, including every asset issued today, keep their token type and spend as they do today.

### Tier 1: required for first regulated launch

1. **Non-bypassable.** A spend of an issuer-controlled asset that does not satisfy the asset’s rule is rejected by the network, including when the holder spends directly through Zswap with its own spending authority, under any note format, key custody, contract custody or delegated-proving arrangement, and for a coin created and spent in the same transaction.
2. **Permissionless circulation with self-custody.** Holders control their own wallet keys and, unless the asset is permissioned (Tier 2 criterion 1), can transfer it without onboarding with the issuer, including when acquired through DeFi, while the issuer retains authority over the asset wherever it is held. Self-custody means only the holder holds its spending key: no control may require a holder to disclose or escrow that key to the issuer or to a proving service, and self-custody does not give the holder exclusive authority over an issuer-controlled asset. A solution states which deployed contracts and wallets must update, and no holding created after adoption falls outside the issuer’s authority because a contract or wallet has not updated.
3. **Freeze.** The issuer can immobilize a specific holding; spends of it are rejected from a maximum delay after the freeze until it is unfrozen. The solution states that delay, which is short enough to meet the response windows in the Freeze gap, and the holder notice declares it.
4. **Pause.** The issuer can halt movement of a specific holder’s balance, including holdings it receives while paused, and separately halt all movement of the token, and resume either, each within the delay bound of criterion 3.
5. **Force transfer.** The issuer can reassign or retire a specific holding without the holder’s key, and total supply reflects the result. This covers every holding except one held under a user’s key that was created before the asset adopted controls; units sent to the burn address, which no key can spend, are not excepted. The network rejects any later spend of a force-transferred holding and any force transfer of a holding already spent. For an excepted holding, the issuer can make a freeze permanent and account for its units as retired or reissued, with the public event (criterion 10) stating what rests on the issuer’s attestation. For a holding held by a contract, the solution states how force transfer affects that contract’s state and the users whose balances it records. Force transfer depends on an issuer-authorised spend path that keeps one canonical nullifier per holding, which [MPS-0035](./mps-0035-shielded-spend-key-exposure.md) lists as its Issuer-authorized spend path MIP.
6. **Issuer visibility.** The issuer can determine the balance of its token held by any holder, including holders who never onboarded or shared viewing access, without that information becoming public. Holdings that existed before the asset adopted controls become visible when they next move.
7. **Issuance and redemption.** The issuer can mint and burn under its own authority, either continuously or through an issuer treasury, and can require that units are destroyed only through its own redemption. On-chain supply reconciles to the issuer’s records.
8. **Threshold approval.** Each privileged action the asset uses (issuance, redemption, freeze and unfreeze, pause and resume, force transfer, adopting controls, rule changes including eligibility-list changes, granting viewing access and recovery) executes only with the configured approval threshold. The issuer can also hold its view under criterion 6 under that threshold.
9. **Holder notice.** A wallet can determine from the chain, before acquiring the asset and while holding it, whether it is issuer-controlled and since when, or by which authority it can adopt controls later, which powers apply and with what delays, how far the issuer can change its rules, what the issuer and any proving service can learn because of the controls, and any recovery arrangement the issuer offers.
10. **Public audit events.** Every privileged action emits a public event from which an auditor can verify that the action occurred and that supply reconciles, without the public learning the affected holder, or the amount of an action that leaves total supply unchanged. For a recovery that changes no ledger state, the event records the issuer’s approved participation.

### Tier 2: permissioned profile and extended controls

1. **Eligibility.** For an asset configured as permissioned, a transfer to or from a holder not on the issuer’s eligibility list is rejected, with eligibility evaluated against the actual spender and recipient. List changes take effect within the delay bound of Tier 1 criterion 3.
2. **Controlled viewing access.** Granting viewing access to a third party requires two approvals where technically feasible.
3. **Verified counterparty.** A transfer can be required to prove both parties eligible at the moment of transfer.
4. **Updatable rules.** The issuer can change the rules governing its asset without redeploying it and without holders moving to a new token.
5. **Lost-key recovery.** For an owner the issuer can identify, a holding whose keys are lost can be restored to that owner through an issuer-authorised path: a force transfer to the owner’s new key, the remedy in Tier 1 criterion 5 for older holdings, or a recovery arrangement the holder entered voluntarily, which keeps the holding under the same key and under which neither the issuer nor an agent can spend without the holder’s participation.

## Recommended MIPs

A single MIP is unlikely to address all the goals this MPS states. This MPS recommends a series of MIPs that lift the technical limitations one at a time and coordinate with solutions to related Problem Statements, such as [MPS-0035: Shielded Spend Authorization Requires Exposing the Spend Key](./mps-0035-shielded-spend-key-exposure.md).

### Token-based custom spend logic

Designs in full how custom logic is attached to a ZSwap spend through recursive proof verification, so a coin carries a spend rule that spending through ZSwap directly with the coin secret key cannot bypass, and meets Tier 0. Addresses root cause 2 (Compact-contract enforcement is bypassable), and root cause 1 (asset-level spend controls) in part: freeze and updatable restrictions need issuer-maintained state, which is left to the stateful gate proof verification follow-up.

It should also outline the further mechanisms this MPS needs to meet the Tier 1 criteria, and give a best-effort, directional view of the Tier 2 criteria. Every network upgrade is a hard fork, which costs users and wallets an update. Outlining the later designs up front lets the first upgrade include placeholders that later upgrades make fully functional, so design and delivery can be staged while keeping the number and impact of hard forks low.

The MIP should specify:

- **The rule bound to the token type.** How a spend-rule descriptor is bound to the token type, with coins that carry no rule left unaffected.
- **Adopting and changing rules.** How binding the rule to the token type squares with Expected Outcome 3 (assets issued today can adopt controls later) and Tier 2 criterion 4 (rules change without holders moving to a new token). If the rule descriptor is an input to the token-type derivation, a token issued without a rule can never acquire one, and changing a rule yields a different token type, so holders would have to migrate. The MIP resolves this or defers it to a named follow-up with its reasons, and considers how [MIP-0011](../mips/mip-0011-native-shielded-token.md) adds circuits to deployed tokens through a verifier-key rotation with no state migration.
- **Pure-circuit rules in full.** How the spend circuit recursively verifies a gate proof (a proof that the coin's rule is satisfied) bound to the spend and enforces the rule bound to the coin's token type, so a spender cannot substitute a different rule.
- **Contract-enforced rules as a future extension.** The extension's dependencies (a proved shielded spender, the Capsule Runtime MIP) and its privacy costs.
- **A goal-by-goal table against this MPS.** For each gap, goal, expected outcome and Tier 1 criterion: delivered by this MIP, or deferred to a named follow-up MIP with its dependencies. Tier 2 criteria are listed with a direction where one is known.
- **The ZSwap changes it needs.** The rule descriptor and spend-proof inputs, written so they can be bundled with the spend-authorization changes for MPS-0035 into one network upgrade.

It should be accompanied by a working prototype, built on the midnight-zk recursion work, delivered as a tagged release in a public repository with tests running in CI and published example usage, demonstrating the Tier 0 criteria, with a negative test showing that the ledger rejects a spend of a coin with a rule but without a valid proof. Because Goal 2 requires that a spend not reveal whether its coin is issuer-controlled, the prototype states whether every spend carries gate verification. Alongside it, a benchmark report gives proving time and circuit size for a spend with no rule and a spend with a rule, on stated reference hardware; the no-rule figure is the per-spend cost bound that Expected Outcome 4 asks for. The report has no pass/fail target; it informs the decisions on bundling with the MPS-0035 changes and on whether every spend carries gate verification.

### Follow-ups and coordinated designs

Once spend rules cannot be bypassed, the following designs can address the goals and criteria the Token-based custom spend logic MIP leaves open.

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

## Copyright

This MPS is licensed under CC-BY-4.0.
