---
MPS: xxx
Title: Off-Chain Token Metadata Registry for Midnight
Category: Ecosystem
Status: Open
Authors:
  - Robert Blessing-Hartley
Proposed Solutions:
  - MIP-xxx to be drafted
Discussions: []
Created: 2026-05-13
License: CC-BY-4.0

---

## Abstract

Midnight tokens are identified on-chain by opaque 32-byte color values. Wallets, explorers, and DApps have no standardized way to resolve these colors into human-readable names, symbols, logos, or operational metadata. The problem is compounded by Midnight's four-quadrant token model (shielded/unshielded x ledger/contract), where a single logical token may operate across multiple quadrants with different privacy characteristics. Without a metadata standard, every application must invent its own token identification scheme, fragmenting the ecosystem and degrading the user experience.

## Problem

Cardano solved this problem with CIP-26, an off-chain metadata registry keyed by `policyId + assetName`. Midnight cannot reuse CIP-26 directly for several reasons:

- **Different identity model.** Midnight tokens are identified by a `Bytes<32>` color derived from `blake2b_256(domainSeparator, contractAddress)`, not by policy script hashes. There is no native script equivalent and no asset name concatenation.
- **Four token quadrants.** A single token color may represent shielded ledger UTXOs, unshielded ledger UTXOs, private contract balances, or transparent contract balances. CIP-26 has no concept of token quadrants because Cardano tokens are uniformly transparent UTXOs.
- **Privacy leakage from metadata queries.** On Cardano, all transfers are public, so querying metadata for a specific asset reveals nothing new. On Midnight, shielded transfers hide which tokens a user holds. A metadata query for a specific color partially defeats this privacy guarantee.
- **Contract-held tokens lack uniform interfaces.** Cardano native tokens all share the same UTXO-based transfer semantics. Midnight contract tokens use diverse Compact circuits -- a wallet needs to know which circuit to call for `transfer`, `approve`, or `burn` operations.
- **Cross-chain bridging.** NIGHT exists on both Midnight and Cardano (as cNIGHT). Metadata must map between the two identity systems.

The absence of a standard means:

- Wallets display raw hex color strings instead of token names
- Explorers cannot label or categorize tokens
- DApps hard-code token metadata, creating maintenance burdens and fragmentation
- Users cannot distinguish between tokens with similar names issued by different contracts
- No mechanism exists to verify that metadata actually corresponds to the claimed token

## Use Cases

### UC1: Wallet Token Display

**Scenario**: A user opens their Midnight wallet and sees three token balances: one NIGHT balance, one shielded game token, and one unshielded governance token.

**Current Approach**: The wallet displays raw 64-character hex strings for each token color. The user cannot tell which token is which without memorizing hex prefixes or cross-referencing external documentation.

**Limitations**: No standardized source of token names, tickers, decimals, or logos. Each wallet vendor must maintain their own curated list or display raw data.

**Desired Outcome**: The wallet queries a metadata registry by color, receives `name`, `ticker`, `decimals`, and `logo` for each token, and displays a human-readable portfolio view.

### UC2: Explorer Token Pages

**Scenario**: A block explorer indexes a transaction that mints a new shielded token. The explorer wants to display the token name, its supply characteristics, and which quadrants it operates in.

**Current Approach**: The explorer can extract the color from the transaction but has no way to resolve it to a name or determine its operational characteristics (shielded-only, unshielded-only, or both).

**Limitations**: Without `tokenOrigin` metadata, the explorer cannot even determine which contract issued the token or what domain separator was used.

**Desired Outcome**: The explorer queries the registry, retrieves the full metadata entry including `tokenOrigin`, and renders a token detail page with verified provenance.

### UC3: DApp Token Integration

**Scenario**: A DeFi DApp wants to list available tokens for a swap interface. It needs to know each token's name, decimals, and -- for contract tokens -- which circuits to call for transfers and approvals.

**Current Approach**: The DApp maintainer manually curates a token list with hard-coded metadata. Adding a new token requires a code change and redeployment.

**Limitations**: Hard-coded lists become stale, cannot cover the long tail of community tokens, and require trust in the DApp maintainer's curation.

**Desired Outcome**: The DApp queries the registry for all tokens matching certain criteria (e.g., quadrant = `unshielded-contract`, standard = `MidnightFungible-v1`), displays them with verified metadata, and discovers circuit names for programmatic interaction.

### UC4: Privacy-Safe Token Lookup

**Scenario**: A user holds shielded tokens and their wallet needs to resolve metadata. The wallet must not reveal which specific tokens the user holds to the metadata server.

**Current Approach**: No approach exists. A naive per-color query leaks the user's token portfolio to the server.

**Limitations**: Any per-subject query pattern is a privacy leak for shielded token holders.

**Desired Outcome**: The wallet either syncs the full registry locally (eliminating per-query leakage) or uses batch queries with decoy subjects to obscure the real lookup targets.

### UC5: Bridged Contract Token Identity Mapping

**Scenario**: A DeFi protocol issues a token on Midnight that is also bridged to Cardano. Cardano wallets need to display the token with consistent branding. Midnight wallets receiving bridged-back tokens need to recognize them.

**Current Approach**: The Cardano side uses CIP-26 with a `policyId + assetName` subject. The Midnight side has no corresponding standard. The mapping between the two identity systems is undocumented. (Note: NIGHT/cNIGHT is a special case -- both sides are protocol-level constants that clients hardcode. This use case applies to contract-issued tokens that bridge across chains.)

**Limitations**: No standard way to declare that a Midnight contract token color corresponds to a specific Cardano CIP-26 subject.

**Desired Outcome**: The `bridge` property in the Midnight metadata entry contains the Cardano `policyId` and `assetName`, enabling wallets on both chains to display consistent metadata for bridged contract tokens.

## Goals

### Primary Goals

1. **Define a metadata schema** that covers all four token quadrants (shielded/unshielded x ledger/contract), including quadrant-specific fields like `contractToken` for account-model tokens and `privacy` for disclosure characteristics.
2. **Enable verifiable provenance** by requiring `tokenOrigin` fields that allow clients to independently verify that a metadata entry corresponds to the claimed on-chain token via color derivation.
3. **Preserve shielded token privacy** by specifying query patterns (batch decoys, full sync) that prevent metadata servers from learning which tokens a user holds.
4. **Provide a server API** for registry read/write operations with validation rules, sequence number enforcement, and attestation verification.
5. **Support cross-chain metadata mapping** between Midnight token colors and Cardano CIP-26 subjects via the `bridge` property.

### Requirements

- **Performance**: Metadata lookups must complete in < 200ms for single-subject queries and < 1s for batch queries of up to 100 subjects.
- **Security**: Attestation signatures must use Ed25519 over Blake2b-256 hashes. Color derivation verification must be mandatory for contract-issued tokens.
- **Privacy**: Read queries must not require authentication. Batch query responses must include uniform-length entries for all requested subjects (including unknowns) to prevent timing inference.
- **Compatibility**: The `bridge` property must contain sufficient information to cross-reference CIP-26 entries on Cardano. The schema must be extensible without breaking existing entries.
- **Usability**: A wallet developer should be able to integrate metadata lookups using only this specification and a single HTTP client. No Compact compilation or ZK proof knowledge required.

### Success Metrics

- **Adoption**: 80% of tokens listed on Midnight explorers have registry entries within 6 months of launch.
- **Verification rate**: 95% of registry entries pass automated color derivation verification.
- **Query privacy**: No metadata server can determine a user's token portfolio from query logs with probability > random baseline, assuming the client follows the batch/sync protocol.
- **Cross-chain coverage**: Bridged token representations on Cardano (e.g., cNIGHT) have corresponding CIP-26 entries. The Midnight-side native token metadata is hardcoded by clients, not registered.

### Non-Goals

- **On-chain metadata storage.** This standard is exclusively off-chain. On-chain metadata (if ever needed) is a separate problem.
- **Token discovery or indexing.** The registry maps known colors to metadata. It is not a search engine for finding tokens by name or category.
- **DUST metadata.** DUST is a network resource, not a token. It has no color and no transferable representation. It does not belong in the token registry.
- **Native token (NIGHT) metadata.** NIGHT has protocol-defined, immutable properties (name, ticker, decimals, atomic unit, cNIGHT bridge mapping). Clients must hardcode these. Registering NIGHT would add a failure dependency on an external service for the one token every client already knows about.
- **NFT metadata (content).** This standard covers identity metadata (name, ticker, logo). Rich NFT content metadata (traits, media, royalties) is a separate concern for a future MIP.
- **Token standard enforcement.** The `contractToken.standard` field is informational. The registry does not verify that a contract actually implements the claimed standard.

## Open Questions

1. **Should the registry support non-fungible token metadata?** NFTs have per-token-ID metadata (traits, media URIs) that does not fit the one-entry-per-color model. Should the proposed MIP extend to handle this, or should a separate MIP define NFT-specific metadata?

2. **How should multi-contract tokens be handled?** A token that is minted by one contract but managed by another (e.g., a bridge contract that wraps a DeFi token) has a color derived from the minting contract. Should the managing contract be listed in the metadata? What happens if ownership of the metadata entry is disputed?

3. **What is the governance model for the canonical registry?** Cardano uses GitHub PRs reviewed by the Cardano Foundation. Should Midnight follow the same model, or should registration be permissionless with attestation-based trust? What prevents squatting on token names?

4. **Should viewing key metadata be included?** If a token supports selective disclosure via viewing keys, should the registry describe the key format and derivation path? This would help auditors discover how to request access, but it also increases the attack surface.

5. **How should deprecated or revoked tokens be handled?** If a contract is abandoned or a token is migrated to a new contract, should the old metadata entry be marked as deprecated? What prevents the original deployer from removing metadata for a token that others still hold?

6. **What is the minimum viable privacy protocol for lightweight clients?** Full registry sync eliminates query leakage but requires significant storage. Batch decoy queries reduce leakage but do not eliminate it. Is there a practical middle ground -- perhaps a Bloom filter download -- that provides strong privacy guarantees with minimal client overhead?

7. **Should the schema support multi-language metadata?** Token names and descriptions may need localization. Should `name` and `description` support language-tagged variants, or should localization be handled by a separate mechanism?

## References

- [CIP-26: Cardano Off-Chain Metadata](https://cips.cardano.org/cip/CIP-26) -- The Cardano standard this work adapts
- [Cardano Token Registry (Testnet)](https://github.com/input-output-hk/metadata-registry-testnet) -- Reference implementation of CIP-26 registry
- [Cardano Off-Chain Metadata Tools](https://github.com/input-output-hk/offchain-metadata-tools) -- Server and tooling for CIP-26
- Midnight Tokenomics Whitepaper -- NIGHT/DUST token model, Star/Speck denominations
- Midnight Compact Standard Library -- `mintShieldedToken`, `mintUnshieldedToken`, `tokenType`, `nativeToken` function signatures

## Appendices

### Appendix A: Token Color Derivation

```
color = blake2b_256(domainSeparator ‖ contractAddress)

Where:
  domainSeparator: Bytes<32>  -- chosen by contract developer
  contractAddress: Bytes<32>  -- kernel.self() at deployment
  ‖ denotes byte concatenation
```

For the native token (NIGHT), the color is the 32-byte zero value: `0x0000...0000`.

A single contract can issue multiple token types by using different domain separators:

```
goldColor   = blake2b_256(pad(32, "game:gold:")   ‖ contractAddress)
silverColor = blake2b_256(pad(32, "game:silver:") ‖ contractAddress)
```

### Appendix B: Comparison of Token Quadrants

| Quadrant            | Location               | Privacy                                    | Value Model     | Metadata Needs                                            |
| ------------------- | ---------------------- | ------------------------------------------ | --------------- | --------------------------------------------------------- |
| Shielded Ledger     | Zswap UTXO set         | Private (sender, recipient, amount hidden) | UTXO            | name, ticker, decimals, logo, privacy info                |
| Unshielded Ledger   | Transparent UTXO set   | Transparent                                | UTXO            | name, ticker, decimals, logo                              |
| Shielded Contract   | Compact contract state | Private (balances hidden via ZK)           | Account (`Map`) | name, ticker, decimals, logo, privacy info, circuit names |
| Unshielded Contract | Compact contract state | Transparent                                | Account (`Map`) | name, ticker, decimals, logo, circuit names, standard     |

### Appendix C: CIP-26 to MIP-026 Field Mapping

| CIP-26 Field                     | MIP-026 Equivalent    | Notes                                                        |
| -------------------------------- | --------------------- | ------------------------------------------------------------ |
| `subject` (policyId + assetName) | `subject` (color hex) | Different derivation                                         |
| `preimage`                       | `tokenOrigin`         | Richer structure, includes quadrants                         |
| `name`                           | `name`                | Same constraints                                             |
| `description`                    | `description`         | Same constraints                                             |
| `ticker`                         | `ticker`              | Same constraints                                             |
| `decimals`                       | `decimals`            | Same constraints                                             |
| `url`                            | `url`                 | Same constraints                                             |
| `logo`                           | `logo`                | Same constraints                                             |
| `policy` (CBOR native script)    | N/A                   | Midnight uses Compact contracts, not native scripts, this may change in the future |
| N/A                              | `privacy`             | New: Midnight-specific                                       |
| N/A                              | `bridge`              | New: cross-chain mapping                                     |
| N/A                              | `contractToken`       | New: account-model token metadata                            |

## Copyright

This MPS is licensed under CC-BY-4.0.
