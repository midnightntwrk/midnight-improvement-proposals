---
MPS: "0014" 
Title: Proof Verification in Compact
Authors: Karmel E - karmoola
Status: Proposed  
Category: Core   
Created: 02-06-2026  
Requires: [none]  
Replaces: [none]  

---
# MPS: Proof Verification in Compact

## **Abstract**

Every time a user interacts with a Midnight smart contract, it produces one proof. That proof has to be sent to the chain on its own. If something takes ten steps, that's ten proofs and ten on-chain transactions.

Compact does not let a developer write a contract that takes an existing proof and uses it as an input to a new proof. So there's no way to fold many steps into one. Every step pays the full on-chain cost.

The desired goal is to let a contract verify a previous proof inside its own logic, so the final output is one proof that stands in for everything that came before it.

**Vision**

A developer writing a Compact contract can take a proof someone already generated, feed it into a new circuit, and end up with a single proof that covers both. They can do this repeatedly to roll up many steps, and they can do it across different circuits.

End state:

- A game with 10 rounds settles with **one** on-chain transaction, not 10.
- A user proves "over 18" from one service and "non-US resident" from another, then combines both into one proof to interact with a contract that needs both without re-running either check.
- Off-chain multi-party flows (each party adds their step to a running proof) finish with one final proof submitted to the chain.

## **Problem**

**1. One action = one proof = one transaction.** Midnight's design is that proofs are generated on the user's device and then submitted to the chain. That's good for privacy. But there's no way today to combine proofs. Each one stands alone and pays full on-chain cost. For anything that naturally has many steps (a game, a multi-party flow, multiple compliance checks), that cost is multiplied by the number of steps.

**2. Compact has no language-level way to express "verify a proof inside this circuit."** Compact does not expose this capability today, so even though the underlying cryptography is well understood, a Midnight developer cannot use it.

**3. There is no way to combine proofs from different circuits.** Even if a developer could batch many runs of the same circuit, they still couldn't combine, say, a "user is over 18" proof from one service with a "user is not in a sanctioned country" proof from another. Each one currently has to be checked separately. That blocks any flow where two independent services contribute proofs that need to land together on-chain.

## **Use Cases**

## **UC 1: A game with many rounds**

Two players play a turn based battle (like Pokémon). The game has 30 rounds. They play the whole thing on their own devices. Every round has to be reported to the chain. 30 rounds means 30 chain transactions and 30 fees. The chain also sees every move along the way, which is more information than anyone needs.

**What should happen:** The players play all 30 rounds locally. At the end, one final result is sent to the chain (the winner, the final score or whatever the game needs). The chain doesn't see the move by move history. The players pay for one transaction not 30.

**Why it matters:** Any game with more than a couple of turns is too expensive to run on Midnight today.

## **UC 2: Combining proof of multiple different things from two different sources**

A user wants to trade on a regulated DEX. The DEX requires two things: the user is over 18, and the user is not a US resident. Two different services handle this: Service A handles age, Service B handles residency. Each is happy to give the user a "yes, this is true" without revealing the underlying documents.

The DEX has to check two separate things, or one service has to do the other's job. There's no clean way for the user to walk up with one piece of evidence that means both of these are true about me.

**What should happen:** The user gets the "over 18" answer from Service A and the "non-US" answer from Service B. They combine the two into a single piece of evidence: "I'm over 18 AND I'm a non-US resident, and it's the same me in both." They hand that one combined thing to the DEX. The DEX checks one thing, sees both facts are true, and lets them trade.

**Why it matters:** This is the building block for real-world compliance use cases. Identity, KYC, accreditation, sanctions screening, healthcare eligibility, they all involve combining facts from different sources, and right now there's no clean way to do that on Midnight.

## **UC 3: Lots of small updates batched into one**

A sealed-bid auction. 1,000 people bid over the course of an hour. Bids are private. Every bid is a chain transaction, or you have to trust someone off-chain to collect them. Neither is good.

**What should happen:** Bids are collected off-chain over the hour. Periodically, one transaction submits a checkpoint and eventually aresult: who won, what they paid. Losing bids stay private. The chain didn't have to process 1,000 separate events.

**Why it matters:** The same pattern shows up in any high-frequency, private use case: voting, micropayments, repeated order updates, recurring private transfers. Without this, none of them are practical on Midnight.

**Gaming: many rounds, one transaction.** A Pokémon-style battle runs N rounds off-chain. Each round produces a proof. Recursion folds them into one final proof that hits the chain. The chain sees one transaction; the player paid for one transaction; the game still has a full audit trail of every round. Today this costs N transactions.

**Multi-party off-chain flows.** An Among Us style game where players pass a running proof between each other. Each player adds their action, the proof gets re-wrapped to include the new step, and only the final state goes on-chain. Today this is impossible because each step would need its own on-chain submission.

**Composable identity and compliance.** Service A proves "user is over 18." Service B proves "user is a non-US resident." The user combines both into one proof and uses it to interact with a DEX that requires both. The DEX verifies one proof, sees both attestations are satisfied, and the user's underlying identity stays private. Today, the DEX would need to verify two separate proofs, or one of the services would have to re-run the other's check.

**Rollup-style batching for any high-frequency contract.**

Any application with frequent state updates (auctions, order updates, voting, micropayments) can batch many off-chain updates into one on-chain proof. Without recursion, every update is its own transaction.

## Acceptance Criteria: 
** A developer can write a Compact contract where a circuit accepts a previously generated proof of the same circuit as input and produces a single new proof that covers both the current and previous computation.
** A developer can write a Compact contract that accepts proofs from different circuits and produces a single proof that attests to the validity of all inputs.
** Performance benchmarks are published covering proof generation time and final proof size across increasing recursion depths for both same-circuit and cross-circuit composition. 
