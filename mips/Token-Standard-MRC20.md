##   Abstract

This MIP proposes a standard interface for fungible tokens within Midnight Network's `compact-contracts `framework. The standard provides basic functionality to transfer tokens, allow third-party spending of tokens, and query token data. It is modeled after Ethereum's successful ERC-20 standard but is specifically designed to leverage Midnight's core privacy features. By using shielded public keys (`string pk`) for account identifiers, this standard enables private ownership and transactions, which is essential for protecting commercial and personal freedom on the network. This proposal includes specifications for state variables, functions, and events that a compliant contract must implement.

##   Motivation

A standardized fungible token interface is a foundational primitive for a healthy and interoperable decentralized application ecosystem. Without a common standard, dApps, wallets, and exchanges would need to implement bespoke logic for every new token, creating significant friction for developers and users.

This standard directly supports Midnight's strategic vision of targeting enterprise use cases in FinTech and Digital ID. These sectors require the ability to transact with digital assets (e.g., stablecoins, security tokens, real-world assets) while preserving the confidentiality of the participants. By providing a privacy-first fungible token standard, this MIP unlocks critical applications such as:
- **Confidential Payments**: Enabling private peer-to-peer and business-to-business transactions.
- **Private Asset Tokenization**: Creating digital representations of real-world assets where ownership is recorded on-chain without exposing sensitive holder information.
- **Decentralized Finance (DeFi)**: Building privacy-preserving financial applications like decentralized exchanges, lending protocols, and yield aggregators.

This standard lowers the barrier to entry for developers familiar with ERC-20, while providing the unique data protection guarantees native to Midnight.

##   Specification

A Midnight Fungible Token contract must be implemented using the `Compact `language and expose the following public interface. The keywords `MUST`, `SHOULD`, and `MAY` are to be interpreted as described in RFC 2119.

**State Variables**
A compliant contract MUST include the following state variables:
- `name: string` - The human-readable name of the token (e.g., "Midnight Token").
- `symbol: string` - The symbol of the token (e.g., "NIGHT").
- `decimals: uint8` - The number of decimal places the token uses.
- `totalSupply: uint128` - The total amount of tokens in existence.
- `balances: Map<string pk, uint128>` - A mapping of shielded account identifiers (accountId) to their token balance.
- `allowances: Map<string pk, Map<string pk, uint128>>` - A nested mapping where` allowances[ownerId][spenderId]` stores the amount an `ownerId` has approved a `spenderId` to withdraw.

**Functions**
A compliant contract MUST implement the following functions:
- `name() public view returns (string)` Returns the name of the token.
- `symbol() public view returns (string)` Returns the symbol of the token.
- `decimals() public view returns (uint8)` Returns the number of decimals used to display token amounts.
- `totalSupply() public view returns (uint128)` Returns the total token supply.
- `balanceOf(accountId: string pk) public view returns (uint128)` Returns the token balance of the account with the address `accountId`.
- `transfer(recipientId: string pk, amount: uint128) public returns (boolean)` Transfers `amount` of tokens from the caller's account to `recipientId`. The caller's identity is proven via a Zero-Knowledge Proof. MUST trigger the `Transfer` event.
- `approve(spenderId: string pk, amount: uint128) public returns (boolean)` Allows `spenderId` to withdraw from the caller's account multiple times, up to the `amount`. If this function is called again, it overwrites the current allowance with amount. MUST trigger the `Approval` event.
- `allowance(ownerId: string pk, spenderId: string pk) public view returns (uint128)` Returns the amount that `spenderId` is still allowed to withdraw from `ownerId`.
- `transferFrom(senderId: string pk, recipientId: string pk, amount: uint128) public returns (boolean)` Transfers amount of tokens from `senderId` to `recipientId`. The `transferFrom` method is used for a withdraw workflow, allowing contracts to transfer tokens on your behalf. This can only be called by an approved `spenderId`. MUST trigger the `Transfer` event.

**Events**
A compliant contract MUST emit the following events:
- `Transfer(from: string pk, to: string pk, value: uint128)` MUST be triggered when tokens are transferred, including zero value transfers.
- `Approval(owner: string pk, spender: string pk, value: uint128)` MUST be triggered on any successful call to `approve`.



##   Rationale

The design of this standard is heavily influenced by Ethereum's ERC-20, which is the most widely adopted and battle-tested fungible token standard in the industry. This familiarity significantly reduces the learning curve for developers migrating from other ecosystems.

The key design distinction lies in the representation of identity. The use of `string pk` for all account identifiers (`ownerId`, `spenderId`, `accountId`) is a deliberate choice to align with Midnight's privacy-by-default architecture. This abstraction allows the underlying system to use Zero-Knowledge Proofs (ZKPs) to verify ownership and permissions for actions like `transfer` and `transferFrom` without revealing the identity of the transacting parties on-chain. This approach provides a powerful combination of public verifiability (of the transaction's validity) and private association (of who was involved).

An alternative considered was to create a single, unified token standard (akin to ERC-1155) from the outset. However, introducing a dedicated fungible token standard first provides a simpler, more focused building block that directly addresses the most common use case. It establishes a clear foundation upon which more complex standards can be built later.

##   Backwards Compatibility Assessment

This MIP introduces a new standard and does not replace any existing ones. As such, it does not create any backwards compatibility issues. It is a foundational proposal intended to set the standard for all future fungible tokens on the Midnight Network.

##   Security Considerations

While this standard benefits from the battle-tested design of ERC-20, its implementation on Midnight introduces unique security considerations, primarily related to privacy.
1. **Correct ZKP Implementation**: The primary security risk is a flaw in the underlying ZK circuits used to prove transaction validity (e.g., proving sufficient balance for a `transfer`). An incorrect implementation could potentially lead to privacy leaks (e.g., transaction linkability) or invalid state transitions (e.g., double-spending or unauthorized token creation). All reference implementations must undergo rigorous security audits.
2. **Standard Token Vulnerabilities**: Implementers must be aware of common token pitfalls, such as integer overflows and underflows. The use of vetted safe math libraries is strongly recommended.
3. **Approval Race Conditions**: The `approve` function can be subject to a race condition where a user changes an allowance and the spender front-runs the transaction. Implementers should be aware of this vector and may consider adding functions like `increaseAllowance` and `decreaseAllowance` to mitigate it, though they are not required by this base standard.

##   Implementation

The reference implementation of this standard is provided in the OpenZeppelin `compact-contracts` repository, specifically within the `FungibleToken.compact` file. Developers intending to create a fungible token on Midnight should use this contract as the canonical reference for implementation. The implementation leverages Midnight's native data types and its `Compact` smart contract language to ensure optimal performance and security.

##   Testing

A comprehensive test suite must accompany any implementation of this standard. The testing procedures should include:
- **Unit Tests:** Each function (`transfer`, `approve`, `balanceOf`, etc.) must be tested in isolation to verify its logic against various inputs and edge cases (e.g., zero-value transfers, transfers to self, maximum value approvals).
- **Integration Tests**: The token contract must be tested in conjunction with other contracts (e.g., a mock decentralized exchange) to ensure interoperability and correct behavior within the broader ecosystem.
- **Property-Based Tests:** Automated tests should be used to check for invariants, such as ensuring that the `totalSupply` never changes as a result of transfers.
- **Privacy Tests**: Specific tests must be designed to verify that ZK proofs function as intended and that no private information (such as the link between a sender and receiver) is leaked on-chain.

##   Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0. Submission requires agreement to the Midnight Foundation Contributor License Agreement [Link to CLA], which includes the assignment of copyright for your contributions to the Foundation.
