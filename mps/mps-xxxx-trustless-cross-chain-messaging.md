---
MPS: "XXXX"
Title: Trustless Cross-Chain Messaging
Authors:
  - Trustless team (trustless-protocol)
Status: Draft
Category: Core
Created: 19-Oct-2026
Requires: none
Replaces: none
MIP: MIP-XXXX (Trustless Messaging Protocol)
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

Midnight lacks a **general-purpose cross-chain messaging protocol**: a way for its contracts to act on messages from other blockchains, backed by evidence they can check themselves. That evidence must show where a message came from, that it really is in the source chain's state, and that the source chain has finalized it. Today the only external route is a one-way path for **NIGHT**, Midnight's token, from **Cardano**, the chain Midnight is linked to. Connecting any other chain usually means building infrastructure for that chain alone, and trusting someone new to vouch for it.

The gap keeps Midnight applications away from the assets and applications of other ecosystems, such as Solana and EVM-compatible chains. Closing it takes more than moving assets: it takes **one common way to handle messages** that still respects how each source chain decides a block is final (or, for rollups, settled).

This MPS states the problem of cross-chain messaging that **relies on no trusted committee** to vouch for messages. It calls for a reusable messaging layer that **keeps how messages are identified and delivered separate from how each chain is checked**. Its first use is asset transfer: bringing value into Midnight, with a way back out.

## Vision

A contract on Midnight acts on a message from another blockchain on **evidence it checks itself**: the message is part of state the source chain has finalized, and settled where that applies. In the other direction, supported chains check messages from Midnight against Midnight's own consensus.

The strongest evidence is the source chain's own **consensus**, the signatures its validators put on every final block. Checked by a **light client**, a verifier of another chain that runs on Midnight, it leaves no committee to trust. What stays trusted is the source chain's consensus, the checkpoint the light client starts from, the cryptography and the code that does the checking.

Each message is **used at most once**, and **no single operator can stop delivery**, under conditions the protocol states, such as the destination chain accepting the transaction.

That opens up cross-chain asset transfer, private eligibility checks and verifiable cross-chain actions. One messaging layer is **reused across chains**, while each chain is checked by its own consensus and settlement rules. Backing assets, holding them safely and recovering them are the job of the asset-transfer rules built on top, not of messaging.

## Problem

### No route to other chains

Midnight has **no general way** to send a message to, or accept one from, another chain. The only external route carries **NIGHT from Cardano**, in one direction; the return path is the subject of MPS-0033.

So there is **no route to or from the chains that hold large amounts of value**, such as Solana and **Robinhood Chain**, a chain Robinhood runs on top of Ethereum (more below). Dollar stablecoins on both chains and Robinhood's tokenized equities cannot reach Midnight, and assets on Midnight cannot reach them. Robinhood Chain alone held about **$2.49B** on 2026-10-14 (total value secured, per L2BEAT), and an application that needs those assets **cannot use them on Midnight**.

### Bridges add a trusted party

The usual way to build a route is a **bridge**. A chain cannot read another chain, so a bridge needs **evidence**: something the receiving chain can check to accept that a message really happened on the sending chain. Most bridges today get it from a **third party** that watches one chain and vouches for it on the other: a multisig, a validator committee or an oracle network. Whoever controls those keys, or gets around the check of their signatures, can **create unbacked assets** on the receiving chain, tokens with nothing locked behind them. That failure is behind the industry's largest losses, such as **Ronin** (about $625M, 2022) and **Wormhole** (about $320M, 2022).

For Midnight the cost is higher than elsewhere. Its value to the institutions it serves is data protection that is **provable rather than promised**, so a route that asks them to trust a committee **re-introduces the counterparty** they chose Midnight to avoid.

### Checking consensus instead

A bridge does not need a third party, because a chain already produces its own evidence. Its consensus is how its validators agree on blocks: on a proof-of-stake chain they sign each block, and a block is **final** once validators holding enough of the chain's **stake** (the tokens they lock up as a guarantee of good behavior) have signed it. Everyone who uses that chain already relies on those signatures. A rollup instead settles on the chain beneath it, which then provides that evidence (see below).

A receiving chain can check the same signatures itself, with no third party in between, and the light client is the component that does it:

- it keeps a small, trusted view of the sending chain's validators, called a **checkpoint**;
- it checks that a block was signed by enough of them;
- it checks that the message is inside that block, using a short **inclusion proof** against a commitment in the block's header (often a Merkle proof).

The receiving chain then trusts what the sending chain's own users trust, plus the light client's code and the checkpoint it starts from.

Someone still has to carry the block, its signatures and the message across. That is a **relayer**: it only carries data, and the light client checks everything it brings, so a relayer can **delay a message but cannot forge one** on its own.

Doing this today runs into three problems, covered in turn below: there is no reusable messaging layer, Midnight cannot yet run a light client's checks, and other chains cannot easily prove Midnight's messages.

### No reusable messaging layer

Where chains are connected today, each route defines its own way of identifying messages, its own rules for delivering them and stopping them from being used twice (**replay protection**), and its own way of checking the other chain, **all tied together**. Two things follow:

- **Every new chain needs a new route.** Applications built on one route cannot reuse it for another chain, and each route must be reviewed and trusted separately.
- **A change in how one chain is checked changes the whole route.** A consensus upgrade on that chain, a cheaper way of checking it or a new key format forces a change everywhere, including the delivery rules applications depend on.

And chains do differ, and do change. What "final" means, and what evidence shows it, is specific to each:

| Chain               | What it is like                                                                                                                                                                                                                                                   | Effect on checking it                                                                                                                                  |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Solana**          | Switched to a new consensus, **Alpenglow**. Validators no longer vote through transactions; they exchange votes bundled into **certificates**                                                                                                                     | The evidence of finality changes form                                                                                                                  |
| **Robinhood Chain** | Is an **optimistic rollup**: it runs its own blocks and **settles** on Ethereum by posting claims about them, called **assertions**. An assertion is accepted unless someone proves it wrong within a challenge window of **6 days 8 hours** (L2BEAT, 2026-10-14) | A block counts as settled once an assertion covering it is confirmed on Ethereum: at least one challenge window, longer if the assertion is challenged |

Because each chain finalizes differently and stores data in its own format (an event log on an **EVM** chain, one running the Ethereum Virtual Machine, and a Solana transaction look nothing alike), **each chain needs its own light client**. The rest does not: identifying a message, delivering it to an application and making sure it is used only once **could work the same way for every chain**.

### Midnight cannot check other chains

A light client on Midnight must check the other chain's **validator signatures**, and each chain signs its own way. Ethereum and Solana under Alpenglow use **BLS signatures**, which fold many validators' signatures into one; checking one takes an elliptic-curve operation called a **pairing**, among other steps. Robinhood Chain settles on Ethereum, so checking it means checking Ethereum's BLS signatures too. Ethereum also hashes its data with **Keccak-256**, and other chains sign with other schemes, such as **ECDSA** or **Ed25519**.

**In short: none of the ways Midnight could run a light client today can check Ethereum's or Solana's validator signatures, mainly because none of them offers a pairing.** The rest of this section explains why.

The result has to end up where a Midnight contract can rely on it. Contracts are written in **Compact**, and each contract call has two parts. The private part is proven by the caller with a **zero-knowledge proof**, which the chain checks against the operation's **verifier key**. The public part is run on-chain by every validator, under Midnight's **ledger**, the rules every node applies. So a light client's checks could run in the private part, written in Compact or as a hand-written circuit proven off-chain; in the public part, as native ledger operations; or outside contracts.

#### Checking directly in Compact

The light client is an ordinary Compact contract that checks signatures and inclusion proofs itself. Two things stand in the way:

- **Missing cryptography.** Ledger 9, live on every Midnight network since November 2026, gave Compact most of the checks other chains need, but still not pairing:

| Check | Needed for | In Compact (ledger 9) |
|---|---|---|
| Keccak-256, SHA-512 | Ethereum data; Ed25519 uses SHA-512 | yes |
| ECDSA (secp256k1), Ed25519 | Other chains' signatures | yes |
| Pairing | BLS signatures: Ethereum, Solana | **no** |

- **Proving cost is very high.** Whoever submits a block to the light client, usually a relayer, proves every check of the call, on its own machine or on a **proof server**, a service that makes proofs for others and so sees their private inputs (MPS-0004). Checking a block means many signatures and long hash chains, and a pairing is among the most expensive things to prove: it uses a different kind of arithmetic from the one proofs are built on, so every step has to be emulated. Even simple contracts already need large files to make proofs: MPS-0039 reports proving keys of up to 73 MB per circuit for small test contracts.

#### Verifying an off-chain proof

The heavy checks run off-chain, in a circuit written by hand in `midnight-zk`, the proof system behind Compact, and Midnight checks only the resulting proof. This **works today**, though unsupported: a contract can use such a hand-written circuit as the check for one of its operations, and whoever submits the message makes the proof. Midnight documents no public-input layout, size limit or stability promise for such circuits, and two things stand in the way:

- **No pairing in `midnight-zk`.** The circuit still has to check BLS signatures, and `midnight-zk` has **no ready-made pairing component**. Reusing a proof another system already makes, such as the **Groth16** proofs some Ethereum light clients produce, does not avoid this: Midnight checks only `midnight-zk` proofs, so that proof must be checked again inside one, which also takes pairings.
- **A ledger upgrade may break it.** The circuit must reproduce exactly the data the ledger feeds into every proof check (its **public inputs**), so a ledger upgrade that changes that data breaks it.

#### Native ledger operations

The checks could run in the public part of a call, as operations built into the ledger, like the precompiled operations EVM chains offer: the user proves nothing extra, and each validator runs one fast native check. Two things stand in the way:

- **No such operations exist.** The public part of a call can move data, do arithmetic and read or write contract state, including the root of a Merkle tree it stores, but has no hash or signature checks.
- **Each one is a protocol change.** Adding one takes a ledger release, a price in the fee model and a node upgrade. MPS-0011 asks for Keccak-256 and ECDSA in Compact; no proposal asks for native ledger operations, or for pairing in either form.

#### Outside contracts

The **runtime**, the node's own built-in logic, could also run the checks. But contracts cannot read its results, and no built-in path carries a runtime result to a contract.

### Other chains cannot easily prove Midnight messages

The other direction has the same need: a chain receiving a message from Midnight needs a light client of Midnight. Midnight's validators finalize blocks with **GRANDPA** and sign with **Ed25519**, which EVM chains have no built-in check for, so checking it there is costly. **BEEFY**, a second set of finality signatures designed to be cheap for other chains to check, is in the runtime but **not signing on mainnet**.

A light client also checks that the message is inside a final block, through the **state root** in the block's header, which commits to all of that block's state. Midnight contract **events**, the notices a contract emits as it runs, are not part of that state ([MIP-0002](../mips/mip-0002-public-contract-log-emission.md)), so a contract has **no direct way to make its outgoing message provable** from a block header. Light clients of Midnight are the subject of MPS-0048, which this MPS builds on for outgoing messages.

### Related proposals

| Proposal | What it covers | Relation to this MPS |
|---|---|---|
| [MPS-0033: mNIGHT to cNIGHT](./mps-0033-mnight-to-cnight.md), [MIP-0023: Committee Bridge Consensus Integration](../mips/mip-0023-committee-bridge-consensus.md) | NIGHT between Midnight and Cardano, with a committee of Midnight validators attesting on Cardano | This MPS concerns **other chains and messages in general**, and asks for the **most trust-minimized verification**: the chains' own consensus, checked by light clients |
| [MPS-0048: Enabling Light Clients on Midnight](./mps-0048-enabling-light-clients.md) | The pieces that let a client check Midnight starting from a recent checkpoint | Messages leaving Midnight **depend on that work**, which does not by itself make a contract's message provable (see MIP-0002 below). This MPS adds the messaging layer, and the direction into Midnight |
| [MPS-0014: Proof Verification in Compact](./mps-0014-proof-verification-recursion.md), [MPS-0011: Native Cryptographic Primitives for Prime Stablecoin Integration](./mps-0011-native-crypto-prims.md), [MPS-0008: Keccak Hashing](./mps-0008-keccak-hashing.md), [MPS-0010: ECDSA Signature Verification](./mps-0010-sig-verify-ecdsa.md) | Capabilities a contract would need to check another chain, including proof verification in Compact (MPS-0014) | This MPS states **why they are needed together**, and what is still missing (pairing-based signatures) |
| [MIP-0002: Public Contract Log Emission](../mips/mip-0002-public-contract-log-emission.md) | Events for contracts | Events are not consensus state, so they **cannot yet be proven** to another chain |

## Use Cases

**UC1: Stablecoins into private applications**

* **Scenario:** A user holds dollar stablecoins on Robinhood Chain or Solana and wants to use them in a Midnight application, for payments or trading, without exposing balances and counterparties.
* **Limitations:** There is no route. A committee-run route would make the user's funds depend on that committee.
* **Desired Outcome:** The stablecoins arrive on Midnight under the **source chain's own security**, and the application uses them privately.

**UC2: Tokenized assets held privately**

* **Scenario:** A holder of Robinhood Chain tokenized assets wants to keep them on Midnight so their positions stay private, subject to the issuer's transfer rules.
* **Limitations:** No route exists, and the asset's value would depend on a bridge never minting more on Midnight than is locked on Robinhood Chain.
* **Desired Outcome:** Supply on Midnight is **always backed** by assets locked on the source chain, and anyone can check it from public chain data.

**UC3: Redemption**

* **Scenario:** A user who moved assets into Midnight moves them back to the original chain.
* **Limitations:** Without a return path, moving into Midnight is a **one-way commitment** and a reason not to move at all (the same effect MPS-0033 describes for NIGHT).
* **Desired Outcome:** The way out adds **no trusted party**, just like the way in.

**UC4: Private checks, public results**

* **Scenario:** A Solana or Robinhood Chain application asks Midnight to check a private condition (identity, eligibility, solvency) and acts on the yes or no result.
* **Limitations:** This needs **a message, not a token**, in both directions, and no messaging route exists.
* **Desired Outcome:** The application on the other chain acts on a result it can verify came from Midnight, while **the data behind it stays on Midnight**.

**UC5: Midnight-native assets abroad**

* **Scenario:** A Midnight application or the NIGHT token reaches users and markets on other chains.
* **Limitations:** Other chains cannot check Midnight cheaply today, and nothing carries the message.
* **Desired Outcome:** Midnight assets leave under **Midnight's own consensus** and return under the other chain's, with no added trusted party.

## Goals

Grouped by layer and listed by priority within each group. The **first deliverable needs Goals 1 to 6 and 8 to 10**. Messaging in both directions and asset transfer (Recommended MIP 6) proceed together: moving value into Midnight is the first use, a way back out is what makes moving in worthwhile (UC3), and value in flight needs controlled upgrades from the start. Goal 7's published figures follow with the reference implementation.

### Messaging

1. **The most trust-minimized verification.** Messages are verified in the most trust-minimized way possible, and the best way is a **light client checking the source chain's own consensus**. A Midnight contract then accepts a message only if the source chain finalized it, under that chain's settlement rules where it has them, and no party beyond the source chain can, on its own, make Midnight accept a message the source chain did not produce. The remaining assumptions (the source chain's consensus, the checkpoint a light client starts from, the cryptography, the correctness of the light client's code, who can upgrade it, and the destination chain executing the transaction) are **stated explicitly**.
2. **Messaging separate from chain verification.**
   - How a message is identified, delivered and protected against replay is **defined once**, for every source chain and both directions, so adding a chain or a direction changes nothing already deployed.
   - Messages keep their source chain's **native encoding**; how to read each encoding is declared, not built into the rules.
   - Each chain's **light client** is a **separate, replaceable part** that **declares what it guarantees** (for example finalized on the source chain, or settled on the chain it settles to). Adding a chain, or changing how one is checked (after a consensus upgrade, or to adopt a cheaper or safer method), does not change the delivery rules or the applications built on them.
3. **At most once, always retryable.** Each message is executed **at most once**; replaying a message, or reusing the proof of one message for another, is rejected on-chain. A message that has not been delivered can be submitted again once the route is running; a failed attempt never uses it up. Applications state their refund path. Messaging does not undo what an application has already done; side effects belong to the application.
4. **No single point of control over delivery.** **More than one independent relayer** can carry each route, so no one relayer or operator can stall or censor a message alone, and relayers can be added or replaced without stopping delivery. This makes censorship harder, not impossible: if every relayer, or the destination chain itself, refuses a message, it stays undelivered. How relayers are approved, paid and replaced is stated, and so is the dependence on the destination chain including the transaction and on its fees being paid.
5. **Messages out of Midnight with the same guarantees.** Messages leaving Midnight meet Goals 1 to 4 too, with Midnight as the source chain, building on light clients of Midnight (MPS-0048).
6. **Usable by any contract.** Any Midnight contract can consume a verified message, within documented limits. Once contract messages can be proven (Recommended MIP 4), any contract can also send one that other chains can check. Asset transfer is **one application of this, not a special case**.
7. **Stated cost and latency.** Cost on each side is **bounded and published**: proving and verification cost, memory, proof size, latency per source chain, and the cost of keeping each light client up to date. Latency imposed by source-chain settlement (for example an optimistic rollup's challenge window) is stated, and any faster path makes its extra trust **explicit and opt-in**.
8. **Controlled upgrades.** Neither the messaging layer nor a chain's light client can be changed by a single party; any change is **announced and delayed**, except an emergency pause and lifting it. A pause must not trap funds: messages already verified stay usable, and anything waiting, such as a refund sent as a message, is delayed until the route is resumed or replaced, never lost because of the pause.

### Asset transfer (built on messaging)

9. **Supply always backed.** The asset-transfer rules built on top of messaging ensure that value released or minted on a destination **never exceeds** what is locked or burned on the source. Total supply on each side can be checked from public chain data.

### Privacy

10. **Receiving reveals no more than the application chooses.** Delivering a message into Midnight does not by itself link the sender on the source chain to the recipient on Midnight: an application can let a recipient take a message without the source chain ever naming them.

## Non-Goals

- **Keeping data private once it leaves Midnight.** A Midnight contract decides what it sends, and whatever it sends is public on the receiving chain. Privacy covers what stays on Midnight, such as the private data behind a result (UC4).
- **Replacing the Cardano to Midnight route** or its return path (MPS-0033).
- **Choosing a particular protocol or proof system, where verification runs, or which chains come first**; that belongs to MIPs.

## Expected Outcomes

- **Real liquidity for Midnight applications:** stablecoins and tokenized assets from other chains become usable in them.
- **Verification instead of attestation:** institutions can use Midnight for assets issued elsewhere on the strength of the source chain's own consensus, checked by a light client, rather than a new party that vouches for messages.
- **Integrate once, reach every supported chain:** supporting a new chain is work on verification for that chain only.
- **Capabilities that serve more than messaging:** what is needed (for example verifying external proofs or pairing-based signatures) also serves light clients, oracles, attestations and identity.
- **Consistent with Midnight's promise:** trust that is provable rather than promised extends to assets crossing chains.

## Open Questions

**Where verification runs.** Should another chain's consensus be checked directly in Compact, through a verified proof, by native ledger operations, or in the runtime with a new path to contracts?

**The boundary between messaging and light clients.** What must a chain's light client provide (for example a finalized commitment to the source chain's state) for the messaging layer to use it, whatever the chain?

**External verifier keys.** Should Midnight support contract operations whose verifier keys come from circuits not compiled from Compact, and if so with which public-input layout, size and cost limits?

**Native operations.** Which cryptographic operations, if any, should become native, and how should they be priced in the cost model? Candidates include pairings on BLS12-381 (the curve behind Ethereum's and Solana's BLS signatures) or BN254 (a curve many proof systems use), Keccak-256, and Blake2b-256 (the hash Midnight itself uses for block headers and state).

**Asset representation.** How should value arriving on Midnight be represented, unshielded or shielded (public or private balances), and how do [MIP-0011](../mips/mip-0011-native-shielded-token.md) (native shielded tokens), [MIP-0012](../mips/mip-0012-native-asset-custody.md) (contract custody of native assets) and [MIP-0014](../mips/mip-0014-native-unshielded-token.md) (native unshielded tokens) apply? If balances are shielded, how does total supply stay publicly checkable (Goal 9)?

**Provable outgoing messages.** How should contract messages be made provable to other chains, given that events are not consensus state (MIP-0002)?

**Bootstrap and light client changes.** How is a light client's first checkpoint chosen and checked? What happens when it falls far behind, for example after a long period without updates? When a chain's light client is replaced, what happens to messages already verified and to messages in flight?

**Upgrades.** Who may update the messaging layer, or a chain's light client, on a deployed route, and under which delay?

**Settlement delays.** How should source-chain settlement delays (days for optimistic rollups) be balanced against user experience without weakening Goal 1?

**Resource bounds.** How are proof sizes, the number of stored checkpoints and the growth of replay-protection records kept within limits, so that a route stays affordable to run over years?

**Fees.** How are relayer fees, and the DUST (Midnight's transaction fee resource) for delivery transactions, paid?

## Recommended MIPs

1. **Cross-chain messaging protocol.** How messages are identified, read from their native encodings, delivered and protected against replay; the interface each chain's light client must provide; the contracts on Midnight and on other chains; and relaying. Addresses Goals 1 to 8 and 10; how relayers and delivery fees are paid is left to a later MIP (Open Questions).
2. **Verification for each source chain.** How one chain's finality, and its settlement where it applies, is checked and exposed through that interface; one MIP or section per chain. Addresses Goals 1, 2 and 7.
3. **Verification capabilities on Midnight.** How a contract can check external finality: new cryptographic checks in Compact, supported external verifier keys, native ledger operations, or a runtime-to-contract path. Addresses Goals 1 and 2; relates to MPS-0014 and MPS-0011.
4. **Provable outgoing messages.** A consensus commitment to contract messages, or a standard state-proof path. Addresses Goals 5 and 6; relates to MIP-0002 and MPS-0048.
5. **Upgrade and pause governance for cross-chain routes.** Who changes the messaging layer and light clients, the delays, and pause semantics. Addresses Goal 8.
6. **Asset transfer on the messaging layer.** Lock and mint, burn and release rules, how assets are represented on Midnight, and supply accounting. Addresses Goal 9; relates to MIP-0011, MIP-0012 and MIP-0014.

## References

- MPS-0004: Trustworthy Delegated Proof Generation for Privacy-Preserving Transactions
- MPS-0008: Native Keccak Hashing for Signature Verification
- MPS-0010: ECDSA/secp256k1 Signature Verification
- MPS-0011: Native Cryptographic Primitives for Prime Stablecoin Integration
- MPS-0014: Proof Verification in Compact
- MPS-0033: mNIGHT to cNIGHT
- MPS-0039: Calling a Contract Requires Its Full Compiled Artifacts
- MPS-0048: Enabling Light Clients on Midnight
- MIP-0002: Public Contract Log Emission for Compact Smart Contracts
- MIP-0011: Native Shielded Token Standard; MIP-0012: Contract Custody of Midnight-Native Assets; MIP-0014: Native Unshielded Token Standard
- MIP-0023: Committee Bridge Consensus Integration
- Solana, "Alpenglow": https://solana.com/upgrades/alpenglow (read 2026-10-22)
- L2BEAT, Robinhood Chain: https://l2beat.com/layer2s/projects/robinhood (read 2026-10-14)
- CoinDesk, "Axie Infinity's Ronin Network Suffers $625M Exploit" (29-Mar-2022): https://www.coindesk.com/tech/2022/03/29/axie-infinitys-ronin-network-suffers-625m-exploit
- Halborn, "Explained: The Wormhole Hack (February 2022)": https://www.halborn.com/blog/post/explained-the-wormhole-hack-february-2022

## Acknowledgements

TBD.

## Copyright

This MPS is licensed under CC-BY-4.0.
