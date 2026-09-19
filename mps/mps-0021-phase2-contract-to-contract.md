---
MPS: "0021" 
Title: Phase 2- Contract 2 Contract
Authors: Karmel E - karmoola
Status: Proposed  
Category: Core   
Created: 06-08-2026  
Requires: [none]  
Replaces: [none]  
MIP: none  

---

# Contract to Contract -- Phase 2

## **Abstract**

Phase 1 of Contract to Contract delivers the initial capabilities needed for contract-to-contract composition on Midnight: interfaces as types, references to contracts held on the ledger, and cross-contract calls. Phase 2 focuses on enabling witnesses and private state. 

**Problem**

**Witnesses and private state:** A contract is made of two kinds of code. Circuits are the public part: their execution is visible on chain, and every DApp and user that touches a contract uses the same circuit code; a locally tampered version will not produce a verifiable proof, because the verifier keys stored on chain for the circuits will reject it. Witness functions are the private part: the contract leaves holes where witness functions go, the code for those holes is supplied separately, and it executes locally on the user's device and never on chain.

The combined effect of the Phase 1 limitations: a DEX can be built now, but all the tokens use the same contract code configured differently with different ledger state. Pool AB and pool BC can both be instances of the same pool contract; token A, token B, and token C can all be instances of the same token contract, with the differences expressed as configuration in the ledger state. What Phase 1 cannot do is support implementations using different circuit code, or any witness code.

## Use Case

**Actor:** A developer building on Midnight.

**Goal:** Build a DEX that accepts any token meeting the token interface, and build tokens with custom logic and private behavior that still work in other people's apps without either side coordinating or shipping each other's code.

**Flow:** The DEX calls the interface functions on whatever token is being swapped. The developer does not need to know who wrote the token or hold its code. A token can satisfy the interface while adding its own circuit code and witness functions.

**Outcome:** Developers integrate by interface instead of by coordination; an app depends on contracts without bundling their code; a token ships distinctive, privacy-bearing logic and stays interoperable.

## **Goals**

1. Allow more than one implementation of an interface, so contracts with different circuit code can satisfy the same interface.
2. Support witnesses and private state in contracts that implement interfaces

## **Expected Outcomes**

- Builders can create multi-contract systems in which independently authored contracts implement the same interface with different circuit code and witness code. End users can interact with contracts whose code did not come from the hosting DApp, because the browser can discover and assemble what it needs. Token authors can ship distinctive logic and private behavior while remaining usable through standard interfaces.

- A circuit in contract A can call a circuit in contract B where the callee (B) defines witness functions and holds and updates its own private state across the call.
- Private values can cross the call boundary in both directions — arguments into the callee and return values back to the caller — without being disclosed to the public transcript, unless explicitly disclosed by the contract author.
- Privacy isolation holds between parties: the caller learns nothing about the callee's private state beyond the returned values, and the callee learns nothing about the caller's private state beyond the passed arguments.
- The proofs for caller and callee circuits in a single logical call are cryptographically bound, such that neither can be replayed or recombined with a different counterpart transaction.
- Existing v1 semantics are preserved: contracts without private state remain callable exactly as before, re-entrancy remains disallowed, and no previously valid multi-contract program changes behavior.
- The feature is expressible in Compact with compiler, ledger, and SDK support released together, and demonstrated end to end with a working example in which a caller with private state invokes a callee that reads and updates its own private state (e.g. a token contract with private balances called by a second contract).


