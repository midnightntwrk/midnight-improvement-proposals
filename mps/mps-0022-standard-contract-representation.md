---
MPS: "0022"
Title: A Standard, Language-Agnostic Representation of Compiled Compact Contracts
Authors: Rodrigo Quelhas @RomarQ
Status: Proposed
Category: Standards
Created: 16-JUN-2026
Requires: none
Replaces: none
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

Compact, Midnight's official smart-contract language, compiles a contract into on-chain artifacts (ZKIR circuit definitions and proving keys) plus generated **TypeScript** that applications use to interact with the contract: build transactions, call circuits, and read state. Because that interaction layer is generated JS/TS bound to `@midnight-ntwrk/compact-runtime`, only JS/TS applications can interact with Compact contracts. SDKs in other languages have no supported path. The compiler also emits a `contract-info.json` metadata file, but its format is unspecified and does not carry enough information to interpret a contract's circuits. This MPS describes the resulting gap: Midnight has no standardized, language-agnostic representation of a compiled contract that an SDK in any language can read to interpret the contract's circuit bodies and storage layout, and thereby interact with it on-chain without relying on the generated JavaScript. This document defines the problem and is intentionally solution-agnostic. It does not specify a format.

## Vision

The standard is a single, machine-readable representation of a deployed contract that any tool, in any language, can consume to interact with it without the source. Interacting with a Midnight contract requires more than call signatures and encoding. A client must execute the contract's off-chain circuit body to build the transaction's transcript and witness inputs before proving, so the standard must carry that body, not just the surface.

Concretely, an SDK or tool written in any language interacts with any Compact contract by reading a standard, machine-readable representation and interpreting it, with no JavaScript runtime and no per-contract code generation. Supporting a new host language means implementing one interpreter against the documented representation, after which every Compact contract works in that language for free. The generated TypeScript becomes one consumer of a shared standard rather than the only supported way to interact with a contract.

## Problem

Compact's only first-class interaction target is JS/TS. For each contract, the compiler emits an executable TypeScript `Contract` class that imports `@midnight-ntwrk/compact-runtime` and inlines the circuit logic as JavaScript. An application calls into that generated class to construct the on-chain transcript, supply witnesses, and submit transactions.

Two distinct intermediate representations are involved, and only one is the subject of this MPS. The **on-chain IR**, the ZKIR circuit definitions and the VM transcript format, is defined by `midnight-ledger` and is already shared by every language and SDK. The **off-chain IR** is Compact-specific: it is the representation of each circuit's body that an SDK must execute off-chain to produce the on-chain transcript and witness inputs before proving. This off-chain Compact IR has no specification, and it is the representation this MPS argues should be standardized.

To interact with the same contract from another language today, a developer has three options, all poor:

- **Reimplement the code generation** for the target language. This means re-deriving Compact's entire circuit-to-host-code lowering and keeping it in lockstep with the compiler forever.
- **Embed a JavaScript engine** inside the other-language application to run the generated TS. This drags a JS runtime into otherwise-native stacks and is impractical for many environments.
- **Fork the Compact compiler and extend `contract-info.json` with the circuit body.** The compiler already emits this file, but it currently carries only the contract's surface (circuit signatures, witness types, ledger field declarations, versions), not the circuit body, so it is not sufficient on its own to build the on-chain transcript for a circuit call. A maintainer fork can extend the file with an interpretable circuit body that other-language tools then read, at the cost of running and maintaining a compiler fork.

This is not hypothetical. [`midnight-rs`](https://github.com/RomarQ/midnight-rs), a Rust SDK for Midnight, needed exactly this capability and took the third path: it forked the Compact compiler ([`RomarQ/compact`, branch `feat/contract-info-extensions`](https://github.com/RomarQ/compact/tree/feat/contract-info-extensions)) to emit an interpretable circuit body inside `contract-info.json`, then built an interpreter against that extended output. The work proved the approach is viable, but it also demonstrates the gap. Absent a standard, every non-JS SDK must carry and maintain its own fork of the compiler, rebasing it onto upstream Compact as the language evolves. There is no standard representation for the various parties to build interoperable tooling against, and no stability contract, so any change to the compiler's emitted output can break downstream tools without warning.

## Use Cases

- **An SDK in another language interacting with a Compact contract.** A team building Midnight tooling in Rust, Python, Go, or any non-JS language wants to deploy a Compact contract, call its circuits, and read its state. Without a standard they must fork the compiler or re-derive Compact's code generation. With one, they implement a single interpreter against the representation, after which every Compact contract works in that language.
- **Tools that decode contract state.** Services, dashboards, and back-end integrations that need to read and type-decode a contract's storage benefit from a documented, machine-readable representation of that contract's state layout, independent of the JS runtime.

## Goals

- A **documented, versioned** representation of a compiled Compact contract, emitted as a first-class compiler output rather than an internal artifact.
- **Language-agnostic:** the representation makes no assumptions about a JavaScript host and is consumable from any language.
- **Sufficient to interact on-chain by interpretation:** it carries everything an SDK needs to build and submit transactions for a contract's circuits without per-contract code generation. This includes the interpretable circuit bodies, the contract's storage layout, and circuit and witness signatures with their types.
- **Scoped to the off-chain Compact IR:** the standard covers the off-chain, Compact-specific circuit body. The on-chain IR defined by `midnight-ledger` (ZKIR and the VM transcript format) and the proving keys remain the source of truth and are not redefined. The representation references them rather than duplicating them.
- **Stable enough to depend on:** consumers can rely on a versioning scheme so that representation changes are detectable rather than silent.

## Expected Outcomes

- SDKs in any language can interact with Compact contracts without forking the compiler or embedding a JS engine.
- The cost of supporting a new host language drops from "re-derive and maintain a code generator" to "implement one interpreter," and the cost of supporting a new contract is zero for an existing interpreter.
- Format drift between the compiler and its consumers becomes a detectable version mismatch instead of a silent breakage.
- The Midnight ecosystem broadens beyond JS/TS: native back-ends, services, and tooling in other languages become first-class.

## Open Questions

- **What is the versioning scheme**, and how do consumers detect compatible versions?
- **Should the standard extend the existing `contract-info.json` or be a new artifact?** Today that file carries only surface metadata. The standard could grow it or live alongside it as a separate file. This is a decision for the MIP, not the problem.

## Recommended MIPs

- **Standard Compiled-Contract Representation.** A MIP defining the concrete, versioned, language-agnostic representation, including the interpretable circuit body, the storage layout, and circuit/witness signatures. The MIP decides whether this extends the existing `contract-info.json` or defines a new artifact. This directly addresses the gap described above: it is what lets any-language SDKs interpret a contract instead of relying on generated JavaScript or forking the compiler.
- **[Lazy Contract State Query RPC](../mips/mip-NNNN-query-contract-state-rpc.md)** (already drafted). The node-side primitive for reading specific fields of a deployed contract's state by path through its `StateValue` tree. It is complementary to the representation above: that representation tells a consumer *what* state a contract has and how it is typed and located, while this RPC is *how* a client fetches a given path from a node. Together they give a non-JS SDK both halves of state access.

## References

- [Compact compiler](https://github.com/LFDT-Minokawa/compact)
- Evidence of the gap: the [Compact compiler fork](https://github.com/RomarQ/compact/tree/feat/contract-info-extensions) that adds an interpretable off-chain circuit body to `contract-info.json`, consumed by the [`midnight-rs`](https://github.com/RomarQ/midnight-rs) Rust SDK.

## Acknowledgements

To be filled in when reviewers and co-authors are confirmed.

## Copyright

This MPS is licensed under CC-BY-4.0.
