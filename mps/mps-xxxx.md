---
MPS: xxxx
Title: Confidential Contract Notifications
Authors: Dominik Zajkowski (@dzajkowski)
Status: Draft
Category: Core
Created: 20-Jul-2026
Requires: MPS-0005 (events), MIP-0002 (public-contract-log-emission)
Replaces: none
MIP: none
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

Midnight can convey contract activity publicly (MIP-0002), letting dApps observe it by reading logs.
MPS-0005 deferred the private case.
This MPS states that problem.

The need is to convey information tied to on-chain activity to a specific party without revealing key elements of it: what kind it is, who it is for, when, whether it happened at all, and its contents.
This is a problem about information flow, not about any particular mechanism.
The information might be passed on-chain, off-chain, or as a hybrid.

Two challenges recur regardless of the medium: keeping the sensitive elements confidential, and letting the intended party learn of and obtain the information cheaply, without any intermediary learning who is interested in what.

Confidentiality must also be durable.
Information tied to on-chain activity may need to stay secret for years, and an adversary who captures protected data keeps it for as long as breaking it takes.
Protection must therefore be chosen to cover the confidentiality horizon rather than to counter the threats currently in view, the nearest of which is quantum attack on today's key agreement.
One way to satisfy this is to never place ciphertext on a permanent public medium at all.

## Vision

A contract author expresses the intent to convey information to a party privately as easily as MIP-0002 makes public information available, choosing per case what stays confidential (type, recipient, the fact that anything was conveyed, or contents).

The intended party, a wallet, a dApp backend, an agent, is informed promptly and obtains the information, while everyone else, including any node, indexer, or relay involved in carrying it, learns nothing they shouldn't.
A recipient's work tracks what is addressed to them rather than total chain activity.
Where a design cannot reach that fully, the cost of the shortfall is explicit: more work for servers, more trust in an intermediary, or leakage that is bounded and stated rather than incidental.

Developers express this declaratively in Compact and reach it through familiar tooling; they do not hand-roll cryptography or run bespoke delivery and scanning infrastructure.

Confidentiality holds for the lifetime the information demands.
Every payload carries a stated confidentiality horizon, and that horizon governs how it may be carried.
Publishing ciphertext on a permanent public medium is treated as what it is: a bounded compromise, secret only until the primitive behind it falls, and the exposure is settled when the data is published rather than when it is attacked.
An author whose horizon fits inside the confidence available in a primitive may take that trade knowingly, and post-quantum schemes push that window furthest today.
An author whose horizon does not fit is not offered the trade, and the chain carries only what can be shown without the payload.

## Problem

Midnight has no first-class way to tell a party about contract activity that concerns them while keeping the sensitive parts of that information from everyone else.

The need is **conveying information tied to on-chain activity to a specific party without revealing key elements of it**: what kind it is, who it is for, when, whether it happened at all, and its contents.
MIP-0002 conveys such information publicly; MPS-0005 deferred the private case.
Today a dApp with this need has no first-class mechanism, and falls back on polling state or overloading primitives not meant for it, obfuscated public events, state cells used as a message bus, or hand-rolled off-chain side channels.

The information could be passed on-chain (e.g. as private events), off-chain (e.g. direct or relayed messaging keyed to on-chain identities), or as a hybrid (an on-chain commitment with off-chain content).
These trade off differently, and choosing among them is part of the problem, not a precondition: on-chain delivery is permanent, globally available, and verifiable, but carries a lasting confidentiality burden; off-chain delivery avoids permanent public storage but must solve reachability, retention, and trust on its own.

Encrypting a payload is not the hard part, and needs only some work: a public event can already carry ciphertext.
What that does not provide is everything around the payload: 
- Hiding the event's type, existence, timing, and recipient.
- Letting the intended party find their information without trial-decrypting everything. 
- Keeping a recipient's separate items unlinkable.
- Keeping ciphertext durable on a permanent public medium.
The problem is these properties, not payload secrecy alone.

Whatever the medium, the same challenges recur. They are also coupled: cheapening discovery tends to leak metadata, and hiding metadata tends to make discovery expensive, so they cannot be satisfied independently.

**Confidentiality.**
The novel requirement is hiding what a public event cannot: the event's type, its recipient, its timing, and the fact that anything was conveyed.
It also requires unlinkability: an observer must not learn that two items share a recipient, nor which party any item is for.
Payload secrecy, by contrast, is the easy part and needs no new mechanism.
What must stay hidden versus what may remain visible (e.g. for indexing or routing) is itself a design tension.

**Private discovery cost.**
The intended party must learn of and obtain the information without costly per-item work across all activity, whether they scan a ledger or receive a message: the naive fallback of trial-decrypting everything imposes exactly that, and eventually defeats any client, not just light or mobile ones.
Cheap per-item filtering removes that cost but not all of it, since a recipient who must observe everything still takes in data volume that grows with the chain.

**Metadata leakage.**
Any intermediary that routes or filters, a node, an indexer, a relay, a messaging server, must not learn which information, or which parties, are involved, including at the network layer.

**Durability.**
Wherever the information or its ciphertext is observable, confidentiality must last as long as the contents are sensitive.
An adversary can capture protected data now and decrypt it later, whether by quantum attack on today's key agreement or by some advance not yet named, so the solution needs to account for a responsible data sharing strategy.
Permanence removes any upper bound on the adversary's time, so on a permanent public medium the confidentiality a design can offer equals the trust horizon of the weakest primitive protecting the payload, and nothing extends it.
A medium the data can leave bounds that window by retention instead, which is why the choice of medium and the choice of primitive cannot be made independently.

**Ergonomics.**
If authors must hand-roll cryptography or run bespoke delivery and scanning infrastructure, the mechanism will be misused or unused.
It needs to be as approachable as MIP-0002's public events.

## Use Cases

**Shielded payment notification.**
A recipient of a shielded transfer needs to know they were paid, and read the details, without examining all activity and without any observer learning who paid whom.
Today this requires polling state or broad trial-decryption.
The canonical case: the recipient must be informed of something addressed to them cheaply.

**Payment memo to a counterparty.**
A sender attaches information to a payment (invoice or reference number, refund address, a note) readable only by the recipient.
Encrypting the payload is straightforward; the friction is discovery.
Carried as a public event, the memo's existence is visible either way, and neither discovery option is satisfactory:
- Tag it so they can find it, and it reveals who it is for and links their memos together.
- Leave it untagged, and they must trial-decrypt broadly to find it.

**Travel Rule compliance.**
Regulated transfers must carry originator and beneficiary identifying information alongside the payment, readable by the counterparties and, on demand, an auditor.
Encrypting that data is the easy part.
The friction is delivering it so the right parties, including a later-designated auditor, can read it while the transfer's existence and participants stay hidden from everyone else.

**Selective disclosure to a designated party.**
A contract's activity produces information a specific non-participant (an auditor or regulator) can read, while it stays hidden from everyone else.
This is a different key model from recipient-addressed information: the reader is neither sender nor the transacting party.

**Long-lived regulated-asset actions.**
An issuer notifies a specific holder of a corporate action (dividend, redemption, lockup) that must remain confidential for the life of the asset, potentially years or decades.
No primitive carries a guarantee on that horizon, so today the issuer must choose between accepting eventual exposure and keeping the notification off chain, in which case the chain can attest that something happened but not what.

## Goals

1. **Confidentiality.**
Hide what a public event cannot: the type, the recipient, the timing, and the fact that anything was conveyed. Keep a recipient's separate items unlinkable.
Payload confidentiality is a baseline.
Authors control what beyond it stays hidden versus visible.

2. **Efficient delivery.**
The intended party learns of and obtains what concerns them with per-item work that scales with what is addressed to them, not with total chain size.
Observation cost may grow with the chain, provided it stays within the reach of a consumer device such as a mobile wallet.

3. **Minimal metadata leakage.**
Delivery must not reveal to any intermediary (node, indexer, relay, or messaging server) which information or which parties are involved.
Any residual leakage must be explicit and bounded.

4. **Durable confidentiality.**
Confidentiality must hold for the lifetime the information demands, against an adversary who captures protected data now and attacks later.
It must not rest solely on assumptions expected to fall to quantum attack.

5. **Selective disclosure.**
Support granting read access to a designated party who is not the sender or recipient (an auditor or regulator) without exposing the information to anyone else, and where the use case requires, after the fact.

6. **Developer ergonomics.**
Authors convey and consumers receive private information through familiar Compact and tooling, without hand-rolling cryptography or running bespoke delivery and scanning infrastructure.

7. **Bounded cost.**
Per-item and aggregate overhead must stay within the budgets of whatever medium carries the information, including the larger key material post-quantum schemes require.

8. **Verifiable aggregation.**
Allow public verification of facts derived from private items (e.g. a vote tally) without revealing the underlying data.

## Expected Outcomes

- **New application classes become native.** Private payments with receipts, compliant transfers, and auditor-visible records can be built on a first-class Midnight mechanism rather than ad-hoc workarounds or state polling.

- **Private dApps stay usable at scale.** Recipients (including wallets on phones) keep pace with a growing chain, because their per-item work is bounded by what is addressed to them and the cost that does scale stays within a consumer device's reach.

- **Confidentiality that ages well.** Every payload has a stated confidentiality horizon, matched against how long the information must stay secret, with the medium chosen to fit that requirement rather than assumed.

- **Approachable, not hand-rolled.** Conveying information privately becomes a first-class, well-defined capability instead of a per-app workaround, lowering the barrier to adopting privacy and reducing the chance of misuse.

- **Compliance without a transparency tradeoff.** Regulated participants can meet obligations like the Travel Rule while keeping counterparty data private, widening who can build and transact on Midnight.

## Open Questions

Each of the following needs to be addressed before a solution is designed:

- **Delivery medium.**
Whether the information should be carried on-chain, off-chain, or as a hybrid needs to be addressed, along with the tradeoffs each medium makes across availability, verifiability, confidentiality burden, reachability, retention, and trust.

- **Reaching the right party.**
How the intended party learns of and obtains what concerns them needs to be addressed, including how much work per item is acceptable, how much data they must observe, and the tradeoff between reducing either and the metadata it may expose.

- **Acceptable metadata visibility.**
What may remain visible (for example, to support indexing or routing) versus what must be hidden (type, recipient, timing, and existence) needs to be addressed, including leakage to nodes, indexers, relays, and network observers.

- **Confidentiality horizon.**
How confidentiality is kept durable against an adversary who captures protected data now and attacks later needs to be addressed, including whether to avoid placing ciphertext on a permanent public medium, whether an unconditionally hiding on-chain representation removes the question by leaving no secret to recover, and the impact the chosen approach has on size and cost.

- **Delivery to unknown parties.**
Whether information must be deliverable to a recipient not known to the sender in advance needs to be addressed, as some approaches cannot support this.

- **Selective disclosure.**
How read access is granted to a designated party who is neither sender nor recipient (including after the fact) needs to be addressed.

- **Authoring surface.**
Compact today expresses only public event emission.
How contract authors express *private* information flow (what stays confidential, to whom, and how it is delivered) needs to be addressed at the language level, so that authors are not left to assemble confidentiality from low-level primitives.

- **Relationship to existing shielded functionality.**
Whether this should extend Midnight's existing shielded mechanisms or stand as a separate facility needs to be addressed, including whether post-quantum confidentiality should be pursued here alone or as part of a broader shielded-layer effort.

- **Delivery semantics.**
How the mechanism guarantees conveyed information is not silently lost, so that outcomes are predictable and observable to the sender, needs to be addressed.

- **Verifiable aggregation.**
How facts derived from private items can be made publicly verifiable without revealing the underlying data needs to be addressed.

## Recommended MIPs

_To be determined._

## References

Internal:

- MIP-0002 - Public Contract Log Emission for Compact Smart Contracts
- MPS-0005 - Events

Prior art in privacy-preserving on-chain information flow and notification:

- Aztec - Note Discovery and note tagging: <https://docs.aztec.network/developers/docs/foundational-topics/advanced/storage/note_discovery>
- Zcash - Encrypted memo field: <https://blog.z.cash/encrypted-memo-field/>; ZIP-307 (light client protocol): <https://zips.z.cash/zip-0307>
- Penumbra - Fuzzy Message Detection: <https://protocol.penumbra.zone/main/crypto/fmd.html>
- Beck, Len, Miers, Green - Fuzzy Message Detection (CCS 2021): <https://www.cs.umd.edu/~imiers/pdf/fuzzy.pdf>
- Seres, Pejó, Burcsi - Why FMD Leads to Fuzzy Privacy Guarantees (FC 2022): <https://fc22.ifca.ai/preproceedings/9.pdf>
- Liu, Tromer - Oblivious Message Retrieval (CRYPTO 2022): <https://eprint.iacr.org/2021/1256.pdf>
- Liu et al. - PerfOMR (USENIX Security 2024): <https://www.usenix.org/system/files/usenixsecurity24-liu-zeyu.pdf>
- ERC-5564 - Stealth Addresses: <https://eips.ethereum.org/EIPS/eip-5564>
- Oasis Sapphire - Encrypted Events: <https://docs.oasis.io/build/sapphire/develop/encrypted-events/>

## Acknowledgements

- Inigo Querejeta Azurmendi (@iquerejeta)

## Copyright

This MPS is licensed under CC-BY-4.0.
