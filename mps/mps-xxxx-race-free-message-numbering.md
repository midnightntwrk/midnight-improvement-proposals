---
MPS: X
Title: Race-free Message Numbering
Authors: Kevin Millikin (kmillikin)
Status: Proposed
Category: Libraries and Tooling
Created: 09-Oct-2026
Requires: none
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

A **data race** can happen when two or more transactions access the same part of a contract's public state, and at least one of the transactions is a write to that state.
Transactions with data races are necessarily rejected by the Midnight Network.
The Compact programming language provides a collection of **ledger ADTs** (Abstract Data Types) to organize public state.
Contract developers want a ledger ADT that allows them to number incoming messages while avoiding unnecessary data races.

**This is a Compact-only feature request.  It will not require changes to the Midnight Network.**

## Vision

It is possible for contract developers to use a set-like abstraction where each inserted element is numbered consecutively according to insertion order on chain,
without any data race involving insertion numbering.

## Problem

A data race can happen when multiple transactions are trying read and write to the same part of a contract's public ledger state.
Transactions with actual data races are rejected by the Midnight Network.
This is essential for correct contracts.

As a simple example where it is essential to reject such transactions, consider a contract with a set of "authorized" users who are allowed to perform some cricital action.
Consider a pair of transactions:

1. An authorized user performs a critical action, and
2. That user's authorization is removed.

There is a potential data race because transaction 1 is reading the "authorized" set of users and transaction 2 is writing to that set.
If transaction 2 is recorded on-chain first, then transaction 1 **must fail** in order to maintain the correctness of the contract state.
There are numerous similar examples involving auctions, voting, payouts, etc.

Public state updates in the Midnight Network are performed by executing programs expressed in **Impact**, a simple special-purpose programming language.
Part of the design of Impact is that it should minimize unnecessary data races.
The Compact programming language's standard library includes a collection of ledger ADTs, which are data structures used for organizing a contract's public state.
The Compact compiler generates Impact code for the on-chain implementation of ledger ADT reads and writes.

Developers want to combine these ledger ADTs in complex ways, but doing that in a Compact contract's source code can introduce (unnecessary) data races.

## Use Cases

The specific concrete problem in this MPS is that a contract developer wants a set-like abstraction that records the insertion order of elements.
A standard trick to implement a set is to use a `Map` with the elements as keys.
The desired set-like abstraction can be implemented a contract's public ledger state using a the `Counter` ADT to count insertions and a `Map` ADT as a set,
where the set elements are the `Map` keys and counter values are the `Map` values.
A pair of insertion transactions will have a data race, since both transactions read and write to the counter.
This data race is unnecessary.
Neither contract cares about the **actual specific** value of the counter, but they have no choice in Compact source code but to read its value.

It is possible to avoid this data race by implementing a new, custom ledger ADT.
A proof of concept implementation exists as a [Compact pull request](https://github.com/LFDT-Minokawa/compact/pull/775).

## Goals

The specific goal is that Compact smart contract developers have access to a race-free message numbering mechanism.
However, a general solution for building race-free ledger ADTs is preferred over adding custom ledger ADTs to the standard library for every contract developer that asks.

## Expected Outcomes

Developers will have access to a data structure with Compact signatures:

```
module OrderedSet<T> {
  circuit insert(x: T): [];
  circuit remove(x: T): Uint<64>;
  circuit member(x: T): Boolean;
  circuit size(): Uint<64>;
}
```

This is the same interface as the Compact `Set` ledger ADT,
except that elements are consecutively numbered according to their insertion order on chain,
and `remove` returns an element's insertion order.

This will not necessarily be exposed as a Compact `module`.

## Open Questions

There is a large design space to explore.
Ideally, the solution is a general one that allows Compact developers to define their own ledger ADTs in some way.
It does not scale to require the next 700 custom contract ledger ADTs to be committed as supported data structures in the Compact standard library.

## Recommended MIPs

This is purely a Compact programming language design issue.
Compact runs off chain, and solving this issue is not expected to immediately require any changes to the Midnight Network.
There are no expected MIPs needed at this time.

## References (Optional)

A proof of concept implementation of a custom ledger ADT: https://github.com/LFDT-Minokawa/compact/pull/775

## Acknowledgements

David Millar-Durant provided the proof of concept implementation and discussion of the requirements.

## Copyright

This MPS is licensed under CC-BY-4.0.
