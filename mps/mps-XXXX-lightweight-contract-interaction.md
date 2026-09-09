---
MPS: "XXXX"
Title: Calling a Contract Requires Its Full Compiled Artifacts
Authors: Hector Bulgarini @hbulgarini, Nicolas Di Prima (NicolasDP)
Status: Proposed
Category: Libraries and Tooling
Created: 09-SEP-2026
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

To call a Midnight contract, a client must hold the contract's full compiled output: the executable circuit bodies (to build the transaction's transcript by local execution) and the ZK artifacts for every circuit it touches (prover key, verifier key, ZKIR). These artifacts are large — prover keys in the `midnight-js` test fixtures alone range from 14 KB to 73 MB per circuit, and ten small test contracts total 384 MB — and they must be shipped to, and kept current on, every client. There is no lightweight interaction path: no equivalent of an EVM ABI, where a few kilobytes of interface description are enough to call any deployed contract. The cost lands hardest on tooling that interacts with contracts *outside their own dApp*: wallet SDKs, agent CLIs, external signers, and back-end services, whose footprint today grows by the full artifact bundle of every dApp they support. Cross-contract calls compound the problem, since a caller needs the artifacts of every contract in the call tree. This MPS describes the gap: Midnight has no way to interact with a deployed contract from a small, stable interface description, with the heavy execution and proving material resolved or delegated rather than bundled. It is intentionally solution-agnostic.

## Vision

Interacting with a deployed Midnight contract should require only a small, versioned interface description — comparable in spirit to an EVM ABI, adapted to Midnight's execution model. A wallet SDK, an agent CLI, or a service supports a new dApp by adding kilobytes, not hundreds of megabytes; its footprint is bounded and independent of how many contracts it can reach. The heavy material — executable circuit bodies, prover keys, ZKIR — is resolved on demand from a verifiable source (anchored to what the chain already stores, such as the deployed verifier-key hashes) or the work that needs it is delegated to a component that already holds it, without weakening integrity or the privacy model. A client can discover, at connection time, what a deployed contract exposes and what calling it requires. Composed (cross-contract) transactions inherit the same property: the caller's cost does not multiply by the number of contracts in the call tree.

## Problem

Midnight's execution model makes the client the executor. A contract call transaction does not carry "circuit name plus arguments"; it carries the *transcript* of running the circuit against the current ledger state, plus a proof of that run. Concretely, in `midnight-js` (the official SDK), the `ContractCallPrototype` placed in a transaction is built entirely from the outputs of local circuit execution — the partitioned public transcript, the private transcript outputs, and the input/output alignment — produced by running the compiled contract in-process. This has three consequences for anyone who wants to call a contract:

**1. The caller must hold the contract's executable.** Every call-transaction entry point in `midnight-js` (`submitCallTx`, `createUnprovenCallTx`, `createCircuitCallTxInterface`, `findDeployedContract`, `deployContract`) is parameterized by a `compiledContract` — the generated executable output of the Compact compiler. There is no interface-only path. Even *finding* a deployed contract requires locally holding every circuit's verifier key, because the SDK verifies the local artifacts against the deployed contract state before use.

**2. The caller must hold — and ship — the ZK artifacts.** Each circuit needs three files (`keys/<circuit>.prover`, `keys/<circuit>.verifier`, `zkir/<circuit>.bzkir`). Prover keys dominate. Sizes measured in the `midnight-js` repository's own test fixtures:

| Circuit (test fixture) | Prover key | Verifier key |
|---|---|---|
| `counter/increment` | 14 KB | ~1.3 KB |
| `unshielded/mintUnshieldedToUserTest` | 2.7 MB | ~2 KB |
| `shielded/mintAndSendShielded` | 9.5 MB | ~2 KB |
| `events/emitMisc` | 64.3 MB | ~2 KB |
| `fee-mint/mintWithShieldedFee` | 72.9 MB | 2.1 KB |

Ten small test contracts total **384 MB** of compiled artifacts. The asymmetry is stark: the chain stores only the verifier key (kilobytes), while the caller must hold the prover key (up to tens of megabytes) — a ratio of roughly 35,000× for the largest fixture. And the artifacts are not even parked: the stock proof-server client **uploads the prover key, verifier key, and ZKIR with every proving request**, as an `application/octet-stream` payload. Only protocol builtin circuits are supplied server-side.

**3. Cross-contract calls multiply the requirement.** A composed transaction carries one proof per contract in the call tree, so — as the SDK's own artifact registry documents — "proving requires artifacts for several compiled contracts". The runtime fetches each callee's *state* from the indexer on demand mid-execution, but the callee's *artifacts* must already be on the client, discovered only at execution time; a missing or stale bundle fails with `ZKArtifactNotFoundError`. As contract-to-contract composition becomes routine, a caller's artifact set grows with the transitive closure of everything its contracts touch.

**Distribution today is entirely client-side.** The supported models are: bundle the `compactc` output on disk (Node/CLI), fetch it from the dApp's own web server (browser), or regenerate it from source with a pinned compiler. All three place the full per-contract bundle on or behind the caller, and all require the caller's bundle to stay in lockstep with what is deployed — a contract upgraded through its maintenance authority invalidates every client's copy silently until the verifier-key check fails.

**The contrast with other ecosystems** is what makes this a competitiveness problem, not just an inconvenience. On EVM, a few-KB ABI is sufficient to construct a valid call to any deployed contract; on Solana, an IDL plays the same role. On Midnight, the equivalent capability costs three to six orders of magnitude more bytes, plus the obligation to execute the contract locally.

A related problem — how a caller obtains *private state* and *witness implementations* for contracts that use them — is real but deliberately **out of scope** here. Most dApps do not yet lean heavily on private state, and practices around it are still forming; it deserves its own problem statement once they settle. This MPS is about the public-artifact footprint, which affects every contract interaction regardless of private state.

## Use Cases

- **A wallet SDK that executes on behalf of many dApps.** A portable identity/wallet SDK (e.g. Passport) must call the contracts of the dApps its users interact with. Bundling each dApp's artifacts was the original plan; with realistic per-contract bundles, a handful of dApps pushes the SDK beyond 1–2 GB — unshippable in a browser or mobile context. Fetching from each dApp's server helps distribution but not footprint, and makes the wallet's correctness depend on every dApp's static-file hosting.
- **Agent tooling and CLIs.** An autonomous agent defines strategies and executes them through a CLI that makes contract calls (agentic trading is the immediate example). If the CLI must carry the artifacts of every dApp the agent may touch, it scales into the gigabytes exactly like the wallet case. Routing the agent through a browser-extension flow instead is both operationally awkward and more error-prone for an LLM-driven agent than a typed CLI.
- **External signers and policy engines.** Tooling that gates and signs contract interactions (e.g. an OWS-style policy-gated signer) needs to understand *what a call is* — contract, entry point, arguments — from a compact description. Today even validating that a call matches a deployed contract requires the full artifact bundle.
- **Back-end services and integrations.** Indexing enrichers, monitoring, and server-side integrations that touch many contracts pay the same per-contract cost, multiplied across everything they observe or call.

## Goals

- **A small, stable interface artifact.** A caller can learn what a deployed contract exposes — circuits, argument types, state layout surface — from kilobytes, not the compiled bundle.
- **Bounded client footprint.** The bytes a client must permanently hold do not grow with the number of contracts it can interact with; heavy material is resolved on demand or delegated.
- **Verifiable artifact resolution.** Whatever resolves or hosts artifacts is anchored to what the chain already commits (the deployed verifier-key hashes), so a stale or substituted artifact is detected, not trusted.
- **Delegable heavy work.** Where execution or proving is delegated (e.g. a proof server that already holds keys, keyed by reference instead of per-request upload), the delegation must not silently expand what third parties learn — the privacy boundary must be explicit.
- **Call-tree proportionality.** Composed transactions do not require the caller to pre-assemble the transitive artifact closure; callee requirements are resolvable the same way the callee's *state* already is.
- **Upgrade legibility.** When a contract's implementation changes at the same address, callers find out through a version/identity check, not through gigabytes of re-shipped artifacts or a late runtime error.

## Expected Outcomes

- Wallet SDKs, agent CLIs, and services support new dApps at near-zero marginal footprint, making "interact with any contract" tooling practical on Midnight.
- The developer experience of calling a deployed contract approaches what EVM/Solana developers take for granted, while keeping Midnight's proof-based execution model intact.
- dApp developers stop being responsible for hand-hosting artifact bundles per client type; distribution becomes an ecosystem capability rather than a per-project chore.
- Agent-driven usage (CLI/MCP-based) becomes viable without funneling agents through browser extensions.

## Open Questions

- **Where does transcript construction move?** Options span a standard interpretable representation executed by lightweight clients (see MPS-0022), a delegated execution service, node- or indexer-side assistance, or a hybrid. Each has different trust and privacy consequences — notably, whoever executes sees the circuit's inputs.
- **Who hosts artifacts, and how are they addressed?** Content-addressed distribution anchored to on-chain verifier-key hashes is the obvious shape, but the operational model (foundation-run, dApp-run, decentralized) is open.
- **Can the proof server hold keys by reference as the norm?** The stock flow uploads prover keys per request; a server-side key store addressed by the on-chain key identity would remove the largest per-call transfer, at the cost of a warming/registration step.
- **What is the minimal interface artifact?** Whether it extends the compiler's existing `contract-info.json` surface metadata, aligns with the MPS-0022 representation, or is a new artifact is a MIP-level decision.
- **How far can argument-only calldata go?** Some circuit calls' transcripts may be deterministically derivable from public state plus arguments, making server-side or node-side construction safe; others depend on witnesses and cannot leave the client. Where that line sits shapes every solution.

## Recommended MIPs

- **Contract interface artifact.** A MIP defining the small, versioned interface description of a deployed contract (circuits, types, identity anchored to deployed verifier-key hashes) sufficient for tooling to describe, validate, and request calls — the ABI-equivalent. Should be co-designed with the MPS-0022 representation so the interface artifact is its surface subset.
- **Verifiable artifact resolution.** A MIP defining how compiled bundles (executables, prover keys, ZKIR) are discovered and fetched by reference — content-addressed, keyed by the on-chain verifier-key identity — so clients resolve on demand instead of bundling, and integrity is checked against the chain rather than a co-shipped manifest.
- **Proof-server key store (keys by reference).** A MIP allowing proving requests to reference keys the server already holds (registered/warmed via the resolution mechanism above), eliminating the per-request upload of multi-MB prover keys.
- **Delegated transcript construction (exploratory).** A MIP examining which circuit calls can be safely constructed away from the client (node, indexer, or service), with an explicit statement of what the delegate learns, for callers that cannot or should not run contract executables.

## References

- `midnight-js` (official SDK): call construction requires the compiled contract (`packages/contracts/src/call.ts`, `unproven-call-tx.ts`, `find-deployed-contract.ts`); transaction bodies are local execution transcripts (`packages/contracts/src/utils/ledger-utils.ts`); per-request key upload (`packages/http-client-proof-provider/src/http-client-proving-provider.ts`); artifact registry and cross-contract note (`packages/types/src/zk-config-registry.ts`); artifact layout and integrity (`packages/node-zk-config-provider`, `packages/fetch-zk-config-provider`). Sizes measured from `testkit-js/testkit-js-e2e/src/contract/compiled/`.
- [MPS-0022: A Standard, Language-Agnostic Representation of Compiled Compact Contracts](./mps-0022-standard-contract-representation.md) — complementary: it addresses *which languages* can interpret a contract; this MPS addresses *how many bytes and whose execution* any caller needs. A shared representation is a plausible building block for both.
- [MPS-0021: Phase 2 contract-to-contract](./mps-0021-phase2-contract-to-contract.md) — composition multiplies the per-contract artifact cost described here.
- [MPS-0004: Trusted proof serving](./mps-0004-trusted-proof-serving.md) — the proving-delegation trust model that a key-by-reference proof server would build on.
- EVM contract ABI and Solana IDL — the interaction-surface baseline in adjacent ecosystems.

## Acknowledgements

To be filled in when reviewers and co-authors are confirmed.

## Copyright

This MPS is licensed under CC-BY-4.0.
