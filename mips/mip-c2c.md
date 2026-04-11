# **Contract-to-Contract Calls (unshielded) phase 1**

## Sentence Summary

This MIP specifies Phase 1 of contract to contract calls, enabling unshielded DeFi interactions on Midnight through dynamic cross-contract invocation without witness access.

Phase 1 ("Without Witness") introduces the foundational cross-contract call capability on Midnight. Contracts can be invoked dynamically at runtime via a sandboxed ZKIR interpreter, restricted to one level of call depth and no access to off-chain private witness data. This enables headless DApps and partner token integrations to interact with the ledger efficiently, prior to the full private state features introduced in Phase 2.

## Motivation

To drive network traffic and developer adoption ahead of Midnight's full privacy stack, partners need to be able to list standard tokens and build basic DeFi primitives (e.g., DEX interactions) with minimal code changes.

## Specification

### Dynamic Address Invocation

A circuit must be passable a contract address at runtime (via public ledger or public input) and successfully invoke a circuit on it.

- **Success:** Circuit invokes the target contract at the provided address.
- **Failure:** Circuit fails if the address does not exist or has no deployed code.

### ZKIR Interpreter Sandbox

The system must provide a secure interpreter that runs the ZKIR representation of the called contract without requiring local source code.

- **Success:** Called contract executes from its ZKIR binary via the sandbox.
- **Failure:** Transaction is rejected if the ZKIR binary is corrupted or malformed.

---

### No-Witness Restriction

The called circuit must be strictly prohibited from accessing any off-chain private witness data.

- **Success:** Execution proceeds with no witness access.
- **Failure:** Interpreter throws an error if a "fetch witness" instruction is encountered during execution.

### Call Depth Limitation

The contract call graph must be restricted to exactly one level deep (e.g., DEX → Token).

- **Success:** Single-depth calls execute successfully.
- **Failure:** Any secondary cross-contract call from within the called circuit triggers a runtime failure.

### Public State Access

The interpreter must successfully perform public state reads and writes for the called contract during execution.

- **Success:** Public state is read and written correctly.
- **Failure:** Writing to a read-only state or unauthorized public state triggers a transaction failure.

### Interface Verification (v3)

If using Version 3 typed ZKIR, the system must be able to statically verify that a contract satisfies a specific interface (e.g., Token Standard).

- **Success:** V3 contract interface is verified prior to invocation.
- **Failure:** V2 contracts that cannot be verified must fail or be managed via off-chain garbage collection.

### Prover Key Generation

The system must be able to dynamically generate prover and verifier keys from the ZKIR representation in a WebAssembly environment.

- **Success:** Keys are generated within acceptable UX latency thresholds for a typical DEX transaction.
- **Failure:** Latency for key generation exceeds acceptable UX thresholds.

**This MIP does not include Contract to Contract alls with Witness.** 

Phase 2 of Contract to Contract calls will be captured in another MIP. Phase 2 represents the mature stage of the Midnight ecosystem, where contracts can share and manipulate private state across contract boundaries.