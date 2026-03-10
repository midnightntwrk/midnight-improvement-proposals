# **Event Emission Support for Compact Smart Contracts- Phase 1**

# Summary

This proposal adds first-class event support to the Compact smart contract language, enabling contracts to emit structured, typed events during execution. Events are captured by the midnight-ledger and exposed via the Midnight Indexer GraphQL API, enabling event-driven DApp development, historical activity queries, and real-time notifications.

Phase 1 delivers **public events** — the `event`, `emit`, and `public` keywords — using the existing `Log` opcode and `ContractLog` infrastructure. No protocol changes are required.

Not part of this MIP: Phase 2 (future work) extends the model with **private events** — `proven` blocks for commitment binding and `encrypt_for` for encrypted delivery — enabling privacy-preserving event patterns. Phase 2 is scoped in Part 2 of this document.

# **Problem Statement**

DApp developers building on Midnight currently have no mechanism to:

- Track historical contract activity (token transfers, swaps, votes)
- Implement event-driven user interfaces with real-time notifications
- Build indexers, analytics tools, or block explorers for custom contracts
- Provide audit trails for governance or compliance

# **Motivation**

**The Inefficiency of State Polling**

Currently, DApps must poll the contract's state tree to detect changes. This is structurally inefficient:

- High Bandwidth Cost: Monitoring a contract for 24 hours via polling consumes ~112 MB of bandwidth vs ~200kb on an event-based approach.
- Transient Data Loss: Polling misses state changes that occur within a block execution

**The Scanning Problem in Private Systems**

Fully encrypted events without metadata force wallets to download and trial-decrypt every event on chain. Phase 2 solves this with **Topic-Based Filtering** achieving 99%+ scanning reduction.

## **Phased Scope**

**Phase 1 — Public Events (MVP)**

Typed event definitions, `emit` statements, public plaintext payloads, GraphQL indexing, fee metering.

# Requirements

**Compact Events (Language + Compiler)**

Developers can define typed event structures and emit them from circuit logic with compile-time validation.

- **Phase 1:** `event` keyword, `emit` statement, compile-time type validation, TypeScript event descriptor generation. Events are public (plaintext).
- **Not Part of this MIP: Phase 2:** `@private` field annotation, `@topic` annotation for filtering, client SDK encryption/decryption, `@zkp` ZK circuit integration.

**Event Indexing & Query API**

Emitted events are ingested, stored, and queryable without special-casing event types.

- **Phase 1:** Ingest `ContractLog` events into dedicated storage. GraphQL query and subscription API filtered by contract address, event type, and block range. Raw + decoded payloads persisted.
- **Not part of this MIP: Phase 2:** Topics column for private event filtering. Wallet-specific decryption with session isolation. `RelevantContractEvent` table for per-wallet relevance.

**Fee Metering**

Prevent spam and align cost with resource usage.

- **Base fee:** Fixed cost per `emit`
- **Data fee:** Variable cost proportional to payload size
- **Encryption fee (Phase 2):** Additional cost for encrypted fields and topic generation