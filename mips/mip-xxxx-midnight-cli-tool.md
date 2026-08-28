---
MIP: X
Title: Midnight CLI tool
Authors:
  - Roman Mazur (mazurroman)
  - Marija Mijailovic (marijamijailovic)
Status: Draft
Category: Standards
Created: 2026-08-28
Requires: none
Replaces: none
MPS: MPS-0002
License: Apache-2.0
---

<!--
 Copyright Midnight Foundation

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

This proposal describes `mdn`, a command-line tool for interacting with Compact contracts on Midnight.
Today, developers write a script for every action.
One script deploys a contract, another one calls a circuit, and another one reads the state.
Each of them sets up the wallet, the providers, and the connection to the proof server again.

With `mdn`, each action will become one command.
A developer will deploy a contract, run a circuit locally without a proof, send a real transaction, and manage the wallet that pays for it.
The same tool will show a contract's circuits and ledger fields, read its decoded state, and turn a failed transaction into a readable message.

Every command will print JSON, and it will exit with success or with a clear error message when it fails, so a coding agent can read the result and choose the next step.
The tool will not change the node, the ledger, the compiler, or the Compact language; it will only put the existing steps behind one interface.

## Motivation

Today, to deploy a Compact contract or to call one of its circuits, a developer has to write a script.
One script deploys the contract, another one calls a circuit, and another one reads its state.
Every script sets up the wallet, the providers, and the connection to the proof server again, so even a small task takes a lot of boilerplate code.

Coding agents have the same problem, because they also have to write a script for every action.

So we propose `mdn`, one command-line tool for the whole workflow: deploy a contract, call a circuit, read its state, and see what a transaction did.
This is useful for coding agents, because an agent can run a command and read the output instead of writing a script first.

## Specification

### 1. Objective

Build one command-line tool, `mdn`, for the whole Compact contract workflow on Midnight.

- Add commands to deploy a contract, call a circuit, submit a transaction, and manage the wallet.
- Add commands to read a contract and its state.
- Add commands that turn a failed transaction into a readable message.
- Add JSON output, so a CI tool or a coding agent can easily read every result.

We are open to adjusting the details with the Midnight developer community and the maintainers of the Midnight developer tools.

### 2. Implementation Mechanics

Every command will pick a network with `--network <name>` for a known one or `--rpc <url>` for any custom RPC.

#### Inspect

- `abi`: a contract's circuits, its witnesses, its ledger fields, and the versions it was built with.
- `read`: the decoded ledger state of a deployed contract.
- `decode-error`: a readable message for the error byte behind an error code (for example `1010`), from the full ledger error table for each version.

#### Execute

- `call`: run a circuit locally, with no proof and no submission, to see the result and the change it would make.
- `send`: build the proof, send the transaction, and print the result.
- `deploy`: deploy a compiled contract.
- `wallet`: manage keys and addresses, show the NIGHT and DUST balance, and register DUST, so that an address can pay for a transaction.

The examples below show what the interface could look like.

```bash
mdn deploy --contract hello-world                   # deploy a compiled contract
mdn call aa6ce704…41688 --circuit storeMessage      # run a circuit locally
mdn send aa6ce704…41688 --circuit storeMessage      # prove and submit a transaction
mdn read aa6ce704…41688                             # a contract's decoded state
mdn decode-error 1010                               # a readable message behind an error code
```

#### JSON output

Every command will take `--json` and print a JSON object with a stable shape.
It will exit with success or with a clear error message when it fails.
Keys and network settings will come from flags or from environment variables, so the same command will work in a terminal, in CI, and inside a coding agent.

The pretty-printed output will be useful for developers, and the JSON output will be useful for CI tools and agents.

```bash
mdn deploy --json
{ "address": "aa6ce704…41688", "tx": "00faa404…df481", "block": 1042 }

mdn read aa6ce704…41688 --json
{ "state": { "message": "hello" }, "block": 1042 }
```

### 3. Architectural Alignment

- The tool will use the existing Midnight libraries for the wallet, the providers, and the ledger.
- It will read the chain through a node and an indexer, like any other client.
- It will read the artifacts of the existing `compact` compiler.
- It will not change the node, the ledger, the compiler, or the Compact language.
- It will keep private keys on the machine of the user, and it will never send a key to a server.
- It will use the existing local development tools for a local network, instead of building a new one.

### 4. Out of scope

Our work will not require any changes to the node, the ledger, the compiler, or the Compact language, and there will not be a need for a hard fork.

We will be happy to continue working with Midnight on some more advanced commands after the initial work is delivered.
`block`, `tx`, and `receipt` read a block, a transaction, and its application result; `decode-tx` reads a raw extrinsic offline; and `trace` shows the Impact operations of a transaction.

## Rationale

We are proposing one binary; the commands will share the same core: reading the chain, decoding the state with the ABI, and running circuits locally.
Foundry works the same way: `forge`, `cast`, and `anvil` are one toolchain widely used by Ethereum developers.
We want to bring the same developer experience to Midnight.
A command line will also give an agent a closed loop, because the agent can write a contract, run `mdn deploy` and `mdn call`, read the failure, and fix the code without a human in the middle.

This proposal covers recommendations 4 and 5 of [MPS-0002](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-0002-developer-tooling.md) in one place, together with the coding-agent integration it asks for.
It also meets recommendation 6, because all commands are documented together.
The tool will print JSON, because that lets a test, a CI tool, or a coding agent take a value from one command and use it in the next one.

## Path to Active

The `mdn` tool will be fully open source.
We will publish binaries for macOS and Linux, together with thorough documentation.
A developer will be able to install the tool and then create, deploy, and invoke a contract without writing a custom script.

### Acceptance Criteria

1. `mdn` allows developers to deploy a contract, call a circuit, and read the state without a script.
2. Every command works with `--network <name>` and with `--rpc <url>`.
3. Every command prints JSON and exits with success or with a clear error message when it fails.
4. A developer can deploy a contract, call a circuit, and read the state without writing a script, on a local network and on a public network.
5. A failed transaction shows a readable message instead of an error byte, for every supported ledger version.
6. Every command has a documentation page with an example.

### Implementation Plan

The work is split into four milestones.
Each milestone ends with commands that a developer can already use, so the tool is useful and available as soon as Milestone 1 finishes and improves with every subsequent Milestone.

#### Milestone 1: Deploy, Call, and Failure Decoding

**Estimated weeks:** 4

**Budget:** $25,000 (USD)

**Goal:** Take a compiled contract, deploy it, run a circuit locally, and get a readable message when something fails, all from the terminal and without writing a script.

**Deliverables / Value Metrics:**

- One tool, `mdn`, with a subcommand for every action, so a developer installs one thing and gets the whole workflow.
- `mdn abi`: reads the artifacts of `compact compile` and parses `contract-info.json` into circuits, witnesses, ledger fields, and the compiler, language, and runtime versions.
- `mdn deploy`: deploys a compiled contract and writes a deployment record with the address, the transaction, and the compiler version.
- `mdn call`: simulates a circuit locally against the current chain state, with the same runtime the chain uses. No proof is built, and nothing is submitted.
- `mdn decode-error`: maps the error byte behind an error code, for example `1010`, to a readable message, with one table per ledger version.
- `--json` on every command, with a stable shape and a schema version, plus exit code `0` for success, one code for a user error, and one code for a chain or network error.
- Network selection with `--network <name>` and `--rpc <url>`. Networks, keys, and endpoints come from flags or from environment variables, so no project file is needed.
- Documentation for every command in this milestone, with an example against a local network.

**Acceptance Criteria:**

- A developer compiles the example contract with `compact compile` and deploys it with `mdn deploy` alone, with no TypeScript file.
- `mdn deploy --json` prints the contract address, and that value can be passed straight into `mdn call`.
- `mdn call` returns the result of a circuit without producing a proof or submitting a transaction.
- `mdn abi` lists the circuits, witnesses, and ledger fields of the example contract, with the versions it was built with.
- A failed transaction shows a readable message instead of an error byte, for every supported ledger version.
- The same command shape works against a local network and against a public network.
- Every command in this milestone has unit tests, and integration tests cover the whole flow.

#### Milestone 2: Wallet and Send

**Estimated weeks:** 4

**Budget:** $25,000 (USD)

**Goal:** Manage the account that pays for a transaction and send a real, proven transaction to the chain.

**Deliverables / Value Metrics:**

- `mdn wallet`: creates a new wallet, or imports one from a seed phrase or a hex seed, shows the address and the NIGHT and DUST balance, funds an address on a local network, and registers DUST, so the address can pay for a transaction.
- `mdn send`: builds the proof with the proof server, submits the transaction, and prints the result.
- `mdn deploy` moves to the same key handling as `mdn wallet`, so one wallet pays for every command.
- `--json` on every command in this milestone, with the same rules as Milestone 1.
- Documentation for every command in this milestone, with an example against a local network.

**Acceptance Criteria:**

- A developer creates or imports a wallet, funds it on a local network, registers DUST, and sees the NIGHT and DUST balance with `mdn wallet`.
- `mdn send` builds the proof, submits the transaction, and prints the transaction hash and the application result.
- A transaction that fails during `mdn send` shows the readable message from `mdn decode-error`.
- Deploy and send use the same wallet, on a local network and on a public network.
- Every command in this milestone has unit tests, and integration tests cover the whole flow.

#### Milestone 3: Contract State and JSON Output

**Estimated weeks:** 4

**Budget:** $25,000 (USD)

**Goal:** Read the decoded state of a deployed contract and make the output stable enough for CI and for a coding agent.

**Deliverables / Value Metrics:**

- `mdn read`: fetches the contract state through the indexer and decodes it with the ABI, so the output shows field names and not raw values.
- Integration tests that run the whole flow, from `compact compile` to `deploy`, `call`, `send`, and `read`, on a local network and on a public network.
- CI that runs every test on macOS and on Linux for every change, including a test for the JSON shape.
- Documentation for every command in this milestone, with an example against a local network.

**Acceptance Criteria:**

- A developer deploys a contract, calls a circuit, sends a transaction, and reads the new state using only `mdn` commands, with no TypeScript file.
- `mdn read` shows the decoded ledger state with field names taken from the ABI.
- Every command prints JSON with `--json`, with the same shape and the same exit codes.
- The whole flow runs green in CI on macOS and on Linux.

#### Milestone 4: Documentation, Release, and Adoption

**Estimated weeks:** 4

**Budget:** $25,000 (USD)

**Goal:** Release the tool, so any developer can install it and use it on their own project.

**Deliverables / Value Metrics:**

- Public open-source release, with prebuilt binaries for macOS and Linux, an installer script `mdnup` like `foundryup`, and a documented path to build from source.
- One documentation page per command with an example, with a getting-started guide that goes from an empty folder to a deployed and called contract.
- An example repository with a contract that a new user can deploy and call.
- A short demo video of the full `mdn` tool.
- Outreach to Midnight developers.

**Acceptance Criteria:**

- A new developer can install `mdn` from a binary and deploy and call the example contract by following the guide alone.
- Every command has a documentation page with a working example.
- Fixes for any bug that blocks the documented happy path.

#### Future Milestones

These come in a later proposal, after the commands above exist.

- One entry point for the local network and for a new project. Today those are `midnight-local-dev` and `create-mn-app`. `mdn node` and `mdn init` would wrap them, so a developer learns one tool, not three.
- A test runner, `mdn test`. It runs every test in a project against a fresh contract and prints which ones passed and which ones failed, the way `forge test` does for Solidity.
- Chain inspection. `mdn block`, `mdn tx`, and `mdn receipt` show a block and its transactions, transaction metadata, and what happened when it was applied. `mdn decode-tx` reads a raw transaction offline with the existing Midnight ledger libraries.
- A `trace` command. It shows the Impact operations of a transaction from the public transcript, the contract and the circuit behind them, the proof and application result, and the operation where it failed. A full trace also shows the state before and after every operation, but the node does not keep the state from before a transaction, so the tool has to replay the chain from an earlier block.
- A real debugger. Step through a circuit, go from an operation back to the line in the `.compact` file, fork a network, and change state while testing. This needs debug information from the compiler and deeper work inside the VM.

## Backwards Compatibility Assessment

Nothing will break, because the tool will be new.
It will not change the node, the ledger, the indexer, the Compact compiler, or the Compact language, and it will not need a hard fork.
Like any other client, it will read the chain and send transactions.

Existing projects will keep working.
A developer can use the tool on a project that was built without it, and a team can keep its own scripts and move over step by step.

Every release will say which node, ledger, and compiler versions it supports, because every new version brings its own changes and features.

## Security Considerations

The tool will be a CLI that runs on the machine of the user, so it will bring no new security risk.
It will store nothing, and it will upload nothing.
Private data will stay local, both the wallet seed and the witness data of a circuit.

## Implementation

The `mdn` CLI tool will live in a new repository.
It will use the existing Midnight libraries for the wallet, the providers, the proof server, and the ledger.
It will read the chain through a node and an indexer.
It will not change the node, the ledger, or the compiler.

## Testing

Every command will have unit tests.
Integration tests will run against a local network, and they will deploy a contract, call a circuit, and read the state.
CI will run all tests on macOS and on Linux for every change.
We will also test the JSON output, so the shape stays the same between versions.

## About the Team

Walnut is well suited for this work.
Our team has four years of experience building blockchain debugging and observability tooling.
We partner with leading ecosystems:

- **Canton.** We are building `dpm trace`, an official tracing and debugging tool for Canton.
- **Ethereum Foundation / Argot.** We own debug info generation in [`solc`](https://github.com/argotorg/solidity), the official Solidity compiler. One-year partnership, being extended.
- **Starkware / Starknet.** We build the [Walnut Starknet Debugger](https://walnut.dev/), covering debug info generation, tracing, simulation, verification, and the hosted debugger itself. Three-year partnership.
- **Miden.** We build the compiler and the debugger for [Miden](https://github.com/0xMiden).
- **Tempo.** We work with them on [solar](https://github.com/paradigmxyz/solar), a Solidity compiler written in Rust.
- **Arbitrum / Offchain Labs.** [StylusDB](https://github.com/OffchainLabs/stylus-sdk-rs/blob/main/cargo-stylus/docs/StylusDebugger.md), the official debugger for Stylus.

## Acknowledgements

N/A

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0.
Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
