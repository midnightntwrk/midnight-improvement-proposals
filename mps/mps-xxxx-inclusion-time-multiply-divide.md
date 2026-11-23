---
MPS: "xxxx" # to be assigned
Title: No Multiplication or Division on Live Ledger State
Authors:
- Farmswap (FarmerRJ)
Status: Proposed
Category: Core
Created: 23-Nov-2026
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

A contract call on Midnight computes in two places. Inside the proof, arithmetic is cheap, but every value is fixed
when the transaction is built. In the public transcript, the on-chain VM runs against the contract state as it is
when the transaction is included, but the VM can only add, subtract and compare.

Any rule of the form "new state = f(state at inclusion, user input)" where f multiplies or divides therefore has two
bad options. It can be computed in the proof against a state snapshot, in which case two users who built against
the same snapshot conflict and one transaction fails. Or it can be spelled out in the transcript as bit-by-bit
shift-and-add and compare-and-subtract sequences of several thousand operations.

Counters already show the value of inclusion-time arithmetic: an increment executes at inclusion, so concurrent
increments never conflict. Nothing comparable exists for products and quotients, which is what pricing, pro-rata
shares, fees, interest and exchange rates need.

This MPS describes the cost of that gap, with measurements of a constant-product pricing step, and asks for
a way to multiply and divide ledger values at inclusion time.

## Vision

A contract can state "price this against the reserves as they are when the transaction lands" as directly as it can
state "increment this counter", at a cost of a few VM operations. Many users can then act on the same contract state
in the same block without conflicting, and without each contract author hand-building integer arithmetic.

## Problem

**The VM's arithmetic.** `midnight-onchain-vm` 4.0.1 offers `add`, `sub` (failing below zero), `addi`, `subi`, `lt`,
`eq`, the boolean `and` / `or` / `neg`, stack operations, forward `branch` / `jmp`, and the state operations. There is
no multiply, divide, remainder, shift or bit-extraction operation, and no loop.

**What it takes to work around it.** A contract that must compute at inclusion has to spell the arithmetic out as
VM operations: a multiplication as repeated doubling and conditional addition, a division as an unrolled
compare-and-subtract with one step per bit. We measured this for the textbook constant-product formula
(output = reserve_out x amount_in / (reserve_in + amount_in)) on 64-bit amounts, node 2.1.101, local network:

- one multiply-and-divide costs roughly 2,000 to 3,000 VM operations, depending on how it is unrolled;
- a second pair, for a proportional fee, brings a single call to as many as 4,000 operations;
- with native operations the same arithmetic would be a handful of operations.

**Consequences.**

- Throughput and fees: this arithmetic is most of the call's block weight and most of what its users pay.
- Transaction limits: one such call can use most of the 1,024 declared-node limit of a transaction, which leaves
  little room to compose it with another call.
- Engineering risk: the sequences cannot be written in Compact today. They have to be produced as raw VM operations
  by custom tooling, outside the guarantees of the stock compiler, and need their own review and differential
  testing against an integer model.

**The alternative is worse for users.** Computing the price in the proof makes every swap depend on the exact state
it was built against. Of the transactions built against one snapshot of a pool only the first can land; the others
fail and must be rebuilt and proven again.
Designs that avoid this with operator-settled order queues give up immediate settlement.

## Use Cases

- **Exchange pools.** Output = reserve_out x amount_in / (reserve_in + amount_in), and a proportional fee, against
  reserves at inclusion.
- **Pro-rata accounting.** Rewards per share, vault share prices, liquidity-share minting and redemption: all are
  products and quotients of totals that change between proving and inclusion.
- **Rates and fees.** Interest accrual, percentage fees and rebasing balances on live totals.
- **Bonding curves and auctions.** A price that is a function of the amount sold so far.

In each case the developer today chooses between conflicts under concurrency and hand-built arithmetic.

## Goals

1. A contract can multiply, divide and take the remainder of integer values read from ledger state at inclusion
   time, with defined behaviour on overflow and on division by zero.
2. The cost of one such operation is of the order of an addition, not thousands of operations.
3. The capability is reachable from Compact, so that no application has to emit VM operations by hand.
4. The result remains bound by the proof in the way other transcript operations are.

## Expected Outcomes

- Several times more state-dependent calls per block on one contract, at lower fees.
- Transactions small enough to compose with other calls.
- Less hand-written low-level code in deployed contracts, and less to audit.
- Applications that need live pricing can be written by developers who are not VM specialists.

## Open Questions

1. Operand width and overflow: fail, saturate or wrap; is a widening multiply (two words out) needed so that
   a x b / c can be computed without overflow?
2. Rounding: floor only, or floor and ceiling variants; remainder as a separate operation or a second result.
3. Gas: how the operations are priced relative to `add`, and whether their cost depends on operand size.
4. Is a general bounded-repeat construct preferable to individual arithmetic operations, or complementary?
5. How Compact should express "computed at inclusion on ledger values" as distinct from in-circuit arithmetic.
6. Interaction with the public-input layout of existing circuits and with transcript well-formedness rules.

## Recommended MIPs

- **Integer multiply, divide and remainder in the on-chain VM.** Operation semantics, failure cases and gas costs.
  Addresses goals 1, 2 and 4.
- **Compact support for inclusion-time arithmetic on ledger values.** Language surface and compiler lowering for the
  new operations. Addresses goal 3.

## References

- `midnight-onchain-vm` 4.0.1, operation set (`ops.rs`, `vm.rs`).
- Measurements: node 2.1.101, ledger-v9 1.0.1, local network, November 2026.

## Acknowledgements

None.

## Copyright

This MPS is licensed under CC-BY-4.0.
