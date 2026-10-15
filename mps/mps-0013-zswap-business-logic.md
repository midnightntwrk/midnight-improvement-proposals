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

The problem is solved when each criterion below can be demonstrated on a shielded asset without bespoke per-asset cryptographic engineering. MPS proposes a staged approach/tiers. A holding is one shielded coin of the asset. An asset adopts controls when its issuer places it under these controls, before issuing or later, and is then issuer-controlled.

The criteria must hold in every note format the network accepts, including the one introduced by MIP-0017, the proposed solution to MPS-0035; where these criteria name MIP-0017, they mean whichever solution to MPS-0035 the network adopts. A solution states how it composes with MIP-0017, how many coordinated network upgrades it needs, and the bounds in Expected Outcome 4. It must not require holders to move holdings to a further note format, and should not require receiving addresses other than today’s and the one MIP-0017 introduces. The criteria depend on choices MIP-0017 fixes at activation: its receiving-address format, the operations performed with a holder’s spending key, which records may consume a coin, and whether existing notes can be made subject to new validity conditions. These must be settled, consistently with these criteria, before MIP-0017 activates.

### Tier 1: required for first regulated launch

1. **Non-bypassable.** A spend of an issuer-controlled asset that does not satisfy the asset’s rule is rejected by the network, including when the holder spends directly through Zswap with its own spending authority, under any note format, key custody, contract custody or delegated-proving arrangement, and for a coin created and spent in the same transaction.
2. **Permissionless circulation with self-custody.** Holders control their own wallet keys and, unless the asset is permissioned (criterion 10), can transfer it without onboarding with the issuer, including when acquired through DeFi, while the issuer retains authority over the asset wherever it is held. Self-custody means only the holder holds its spending key: no control may require a holder to disclose or escrow that key to the issuer or to a proving service, and self-custody does not give the holder exclusive authority over an issuer-controlled asset. A solution states which deployed contracts and wallets must update, and no holding created after adoption falls outside the issuer’s authority because a contract or wallet has not updated.
3. **Freeze.** The issuer can immobilize a specific holding; spends of it are rejected from a maximum delay after the freeze until it is unfrozen. The solution states that delay, which is short enough to meet the response windows in the Freeze gap, and the holder notice declares it.
4. **Pause.** The issuer can halt movement of a specific holder’s balance, including holdings it receives while paused, and separately halt all movement of the token, and resume either, each within the delay bound of criterion 3.
5. **Force transfer.** The issuer can reassign or retire a specific holding without the holder’s key, and total supply reflects the result. This covers every holding except one held under a user’s key that was created before the asset adopted controls; units sent to the burn address, which no key can spend, are not excepted. The network rejects any later spend of a force-transferred holding and any force transfer of a holding already spent. For an excepted holding, the issuer can make a freeze permanent and account for its units as retired or reissued, with the public event (criterion 15) stating what rests on the issuer’s attestation. For a holding held by a contract, the solution states how force transfer affects that contract’s state and the users whose balances it records.
6. **Issuer visibility.** The issuer can determine the balance of its token held by any holder, including holders who never onboarded or shared viewing access, without that information becoming public. Holdings that existed before the asset adopted controls become visible when they next move.
7. **Issuance and redemption.** The issuer can mint and burn under its own authority, either continuously or through an issuer treasury, and can require that units are destroyed only through its own redemption. On-chain supply reconciles to the issuer’s records.
8. **Threshold approval.** Each privileged action the asset uses (issuance, redemption, freeze and unfreeze, pause and resume, force transfer, adopting controls, rule changes including eligibility-list changes, granting viewing access and recovery) executes only with the configured approval threshold. The issuer can also hold its view under criterion 6 under that threshold.
9. **Holder notice.** A wallet can determine from the chain, before acquiring the asset and while holding it, whether it is issuer-controlled and since when, or by which authority it can adopt controls later, which powers apply and with what delays, how far the issuer can change its rules, what the issuer and any proving service can learn because of the controls, and any recovery arrangement the issuer offers.
10. **Public audit events.** Every privileged action emits a public event from which an auditor can verify that the action occurred and that supply reconciles, without the public learning the affected holder, or the amount of an action that leaves total supply unchanged. For a recovery that changes no ledger state, the event records the issuer’s approved participation.

### Tier 2: permissioned profile and extended controls

1. **Eligibility.** For an asset configured as permissioned, a transfer to or from a holder not on the issuer’s eligibility list is rejected, with eligibility evaluated against the actual spender and recipient. List changes take effect within the delay bound of criterion 3.
2. **Controlled viewing access.** Granting viewing access to a third party requires two approvals where technically feasible.
3. **Verified counterparty.** A transfer can be required to prove both parties eligible at the moment of transfer.
4. **Updatable rules.** The issuer can change the rules governing its asset without redeploying it and without holders moving to a new token.
5. **Lost-key recovery.** For an owner the issuer can identify, a holding whose keys are lost can be restored to that owner through an issuer-authorised path: a force transfer to the owner’s new key, the remedy in criterion 5 for older holdings, or a recovery arrangement the holder entered voluntarily, which keeps the holding under the same key and under which neither the issuer nor an agent can spend without the holder’s participation.

## Copyright

This MPS is licensed under CC-BY-4.0.
