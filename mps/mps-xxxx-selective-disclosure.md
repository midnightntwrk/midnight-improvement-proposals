---
MPS: "xxxx"
Title: Selective Disclosure to a Chosen Party 
Authors:
  - Karmel Elshinnawi <Karmoola>
  - Ricardo Ruis <riusricardo>
Status: Draft  
Category: Core  
Created: 17-Sep-2026  
Requires: none  
Replaces: none  
MIP: none  

---

<!-- Copyright 2026 Midnight Foundation Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at https://www.apache.org/licenses/LICENSE-2.0 Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License. -->

## **Abstract**

Midnight promises selective disclosure: prove facts about private data without revealing it, and reveal specific data to a chosen party while keeping the rest private. Compact delivers the first half. The second half has one mechanism, a whole-wallet, permanent, incoming-only viewing key, and no contract-side equivalent.

This MPS states that gap. It is the access-control slice of Confidential Contract Notifications (`mps-xxxx`, PR #271): who may read a specific piece of shielded data, for how long, and how they know it is real.

## **Vision**

A user or a contract author names what is revealed, to whom, and for how long, as easily as marking something public today. A grant covers one payment, one counterparty, or one period, and nothing else. The reader can check what they receive against the chain and can tell who disclosed it. A regulator gets exactly the evidence the obligation requires, and nobody else learns anything.

## **Problem**

Midnight has no first-class way to reveal specific shielded data to a specific party while keeping it hidden from everyone else. The one mechanism it has cannot be scoped, bounded in time, or withdrawn.

**What is promised.** The documentation says users "maintain complete control through selective disclosure, choosing precisely what information to reveal and to whom", that regulated entities can report to authorities "without compromising user privacy", and that users "can reveal specific transactions for compliance while keeping others private" through viewing keys.

**What exists.** A shielded wallet has one viewing key: the key that decrypts payments sent to it. It is whole-wallet: there is no narrower key for one payment, one counterparty, or one period, and no way to derive one. It is permanent: a copy handed out today reads everything that arrives tomorrow, and the wallet cannot revoke it without changing address. It is incoming-only: it shows what was received but not what was spent, so it cannot establish a balance. A sender can prove a payment only by hand, by handing over the coin's preimage so the reader recomputes the commitment; nothing in the wallet produces this, nothing ties it to the sender, and the reader cannot tell whether the coin was later spent. The indexer already accepts this key to filter a wallet's transactions, so any hosted service that helps a wallet find its coins holds the amounts of everything that wallet has ever received.

**Contracts have the same gap.** A value in a Compact contract is either private to the person running it or public to everyone. The one intermediate is a commitment: a sealed record of a value published without revealing it. Opening that record to one chosen reader, so they can verify it against the chain and know who opened it, is possible only by hand, with a format, delivery path, and verification code built separately by every application. Public events do not help: they are authenticated to the contract and entry point that emitted them, not to the user who triggered the call, so encrypting a payload to one reader tells the reader which contract spoke, not which party.

**What others learned.** Every mature shielded system started with a whole-account viewing key and found it too coarse for compliance. Zcash's specification records that its viewing keys cannot be limited in time or per transaction. Penumbra and Aleo added per-transaction disclosure objects, so a user reveals one transaction instead of a key that opens all past and future activity, and Aleo anchors each one on chain so the reader can verify it. Recent designs for regulated ledgers add grants that expire on their own and grants the user can withdraw. Midnight has only the coarse instrument, and it is coarser than Zcash's, since it has no sender side.

**Out of scope.** Concealing that a disclosure happened, letting a reader find what was granted without being told, and choosing how the data travels belong to PR #271.

## **Use Cases**

**Audit of a period.** A business under audit must show every shielded receipt in one fiscal year and nothing else. Today the only thing it can hand over is a key that exposes every receipt it has ever had and ever will have.

**Proof of payment.** A sender must show a counterparty, a court, or a tax authority that a specific shielded payment was made, to whom, and for how much. Today a sender can only hand over raw coin data by hand, with nothing tying it to them.

**Regulator access to one contract record.** A contract's activity produces a record that a regulator, often appointed after the fact, must read and trust while it stays hidden from everyone else. Today the author must publish it to the whole network or build a bespoke disclosure path.

**Delegated monitoring that can be ended.** A custodian or accounting service watches a wallet's incoming activity on the owner's behalf, and the owner later ends the arrangement. Today ending it means moving to a new wallet.

## **Goals**

1. A user or a contract can reveal a specific piece of shielded data to a named reader, and nobody else, including any node or indexer, can read it.
2. A grant covers only what it names: one item, one direction, one counterparty, or one period. It must not expose the granting party's other data and must not require handing over a long-term key.
3. A grant has a stated lifetime, and a grant over future activity can be ended without changing address. What a reader has already learned cannot be taken back; withdrawal means ending access to what has not yet been disclosed.
4. The reader can check the revealed data against what the chain recorded, so nothing can be substituted, and can establish who made the disclosure. A reader can be added after the fact for any data the chain recorded.
5. Authors express disclosure in Compact, users through their wallet, and readers verify through a standard SDK path.
