**MPS:** ????

**Title:** CAIP-2 Compliant Network Identifiers for Wallet Ecosystem Integration

**Category:** Wallet

**Status: **Open
**Authors:**  Bob Blessing-Hartley <bob@shielded.io>

**Created:** 2026-04-07

## Abstract

Midnight currently uses simple string-based network identifiers (`'mainnet'`, `'qanet'`, `'testnet'`) that are not compliant with blockchain industry standards. This creates barriers to integration with multi-chain wallet infrastructure, including the Open Wallet Standard (OWS), WalletConnect, and other CAIP-compliant systems. The lack of standardized network identification prevents Midnight from participating in the broader multi-chain wallet ecosystem and forces developers to build custom integration layers for each wallet provider.

## Problem

### Current State

Midnight's network identification system uses unstructured string values defined in `/midnight-wallet/packages/abstractions/src/NetworkId.ts`:

```typescript
export const NetworkId = {
  MainNet: 'mainnet',
  TestNet: 'testnet',
  DevNet: 'devnet',
  QaNet: 'qanet',
  Undeployed: 'undeployed',
  Preview: 'preview',
  PreProd: 'preprod',
} as const;
```

### Limitations

1. **Ambiguity**: The identifier `'mainnet'` is used by hundreds of blockchains, making it impossible to distinguish Midnight mainnet from Ethereum mainnet, Solana mainnet, etc., without additional context.

2. **Incompatibility with Standards**: The [CAIP-2 specification](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md) defines blockchain identifiers as `namespace:reference` (e.g., `ethereum:1`, `solana:5eykt4...`), which Midnight does not follow.

3. **Wallet Integration Barriers**: Multi-chain wallets (MetaMask, WalletConnect, OWS) require CAIP-2 identifiers to route transactions to the correct chain. Midnight's non-standard format requires custom integration code for each wallet provider.

4. **Ecosystem Fragmentation**: Without CAIP-2 compliance, Midnight cannot leverage:
   - CAIP-10 (Account IDs) for unified account addressing
   - CAIP-25 (Wallet Handshake) for dApp-to-wallet connections
   - CAIP-27 (Wallet Registration) for wallet discovery
   - Other Chain Agnostic Improvement Proposals

5. **Developer Experience**: DApp developers must learn Midnight-specific network handling instead of using familiar multi-chain patterns.

### Impact Assessment

**Affected Users:**
- Wallet developers integrating Midnight support
- DApp developers building multi-chain applications
- End users wanting to use popular multi-chain wallets
- Infrastructure providers (block explorers, indexers) building cross-chain tools

**Current Workarounds:**
- Manual string mapping in each integration
- Hardcoded context assumptions (e.g., "mainnet" means Midnight mainnet if we're in Midnight code)
- Custom wallet connector protocols
- Limited to Midnight-specific wallets only

## Use Cases

### UC1: Multi-Chain Wallet Integration

**Scenario**: A developer wants to add Midnight support to an existing multi-chain wallet that already supports Ethereum, Solana, and Cosmos using the Open Wallet Standard.

**Current Approach**: The developer must:
1. Fork or extend OWS to handle Midnight's non-standard network IDs
2. Build custom translation layer between `'mainnet'` and `'midnight:mainnet'`
3. Maintain separate code paths for Midnight vs other chains
4. Risk ambiguity bugs when multiple chains use `'mainnet'` string

**Limitations**:

- Significant additional integration effort per wallet
- Ongoing maintenance burden for custom integration code
- Cannot use standard CAIP tooling and libraries
- Testing complexity increases significantly

**Desired Outcome**: Midnight network IDs work seamlessly with OWS and other CAIP-2 compliant systems using standard integration patterns (2-3 days effort instead of 2-3 weeks).

### UC2: DApp Multi-Network Support

**Scenario**: A DApp developer wants to deploy their application on multiple Midnight networks (mainnet, testnet, preview) and allow users to switch networks, similar to how Ethereum DApps handle mainnet/Sepolia/Goerli.

**Current Approach**: The developer must:
1. Hardcode network string comparisons: `if (network === 'mainnet')`
2. Build custom network switching UI
3. Cannot use standard libraries like wagmi, viem, or web3modal
4. Risk of string typos causing bugs (`'mainnet'` vs `'Mainnet'` vs `'main-net'`)

**Limitations**:
- No type safety for network identifiers
- Cannot leverage existing multi-chain DApp frameworks
- Custom error handling for network mismatches
- Poor UX compared to standard multi-chain DApps

**Desired Outcome**: Use standard CAIP-2 identifiers (`midnight:mainnet`, `midnight:testnet`) that work with existing multi-chain DApp frameworks and provide compile-time type safety.

### UC3: Cross-Chain Analytics Platform

**Scenario**: A blockchain analytics platform wants to aggregate data from Midnight alongside Ethereum, Cardano, and other chains, providing unified portfolio tracking and transaction history.

**Current Approach**: The platform must:
1. Build Midnight-specific data ingestion pipeline
2. Manually namespace all Midnight data to avoid collisions
3. Cannot use standard multi-chain indexing tools
4. Risk of data corruption if `'mainnet'` from different chains gets mixed

**Limitations**:
- Cannot use CAIP-compliant indexing infrastructure
- Higher development and maintenance costs
- Slower time-to-market for Midnight integration
- Data model inconsistencies across chains

**Desired Outcome**: Midnight data can be ingested using standard CAIP-2 chain identifiers, allowing reuse of existing multi-chain infrastructure and preventing namespace collisions.

### UC4: WalletConnect Integration

**Scenario**: Users want to connect their Midnight wallet to DApps using WalletConnect, the industry-standard wallet connection protocol used by 400+ wallets and 2000+ DApps.

**Current Approach**:
1. WalletConnect requires CAIP-2 chain IDs for session establishment
2. Midnight must build custom WalletConnect bridge
3. DApps cannot discover Midnight support automatically
4. Users see "unsupported chain" errors

**Limitations**:
- WalletConnect integration requires protocol extensions
- Cannot participate in WalletConnect ecosystem (400+ wallets)
- Missed network effects from standardized wallet connections
- Fragmented user experience

**Desired Outcome**: Midnight wallets work natively with WalletConnect using standard CAIP-2 identifiers, enabling automatic discovery and connection to WalletConnect-enabled DApps.

## Goals

### Primary Goals

1. **Enable CAIP-2 compliance** for Midnight network identifiers to support integration with standards-based wallet infrastructure.

2. **Maintain backward compatibility** during transition period to avoid breaking existing wallets, DApps, and infrastructure.

3. **Provide clear migration path** for wallet developers, DApp developers, and node operators to adopt CAIP-2 identifiers.

4. **Minimize ecosystem disruption** by avoiding changes to core ledger state, address formats, or transaction structures if possible.

### Requirements

#### Performance
- Network identifier parsing must add < 1ms overhead to transaction validation
- No impact on ZK proof generation time
- No additional storage overhead for ledger state

#### Security
- Network identifier cannot be spoofed or manipulated to route transactions to wrong chain
- Must prevent namespace collisions with other blockchains
- Migration must not create vulnerability window

#### Privacy
- Network identifier changes must not affect zero-knowledge proof structures
- No privacy degradation during migration period
- Shielded and unshielded transactions equally supported

#### Compatibility
- Support both legacy (`'mainnet'`) and CAIP-2 (`'midnight:mainnet'`) formats during transition
- Provide automatic migration for existing wallet storage
- No breaking changes to external APIs during migration period
- Coordinate with ledger upgrade if blockchain state changes required

#### Usability
- Clear documentation and migration guides
- Automated migration tools for wallet developers
- Transparent to end users (no manual intervention required)
- Type-safe APIs for developers

### Success Metrics

Solutions will be evaluated based on:

- **Integration Effort**: < 3 days to add Midnight to CAIP-2 compliant wallet (vs current 2-3 weeks)
- **Ecosystem Adoption**: Midnight supported in ≥3 major multi-chain wallets within 6 months
- **Migration Success Rate**: ≥95% of existing wallets migrate without data loss
- **Developer Satisfaction**: Positive feedback in developer surveys on migration process
- **Standard Compliance**: 100% conformance to CAIP-2 specification
- **Breaking Changes**: Zero breaking changes to user-facing wallet functionality

### Non-Goals

The following are explicitly OUT of scope:

- **Changing address format**: Bech32m address encoding should remain unchanged (`mn_addr1...`, `mn_shield-addr1...`)
- **Multi-chain support**: This MPS focuses on network identification, not cross-chain transactions or bridges
- **CAIP-10 adoption**: Account identifier standards are a separate concern
- **Governance changes**: Network selection or upgrade governance is out of scope
- **Genesis hash computation**: Not required if network name suffices as CAIP-2 reference
- **Testnet consolidation**: The number and naming of test networks is out of scope

## Open Questions

Solutions must address the following design considerations:

### 1. CAIP-2 Reference Format

**Question**: What should the `reference` portion of `midnight:{reference}` be?

**Options**:
- a) Network name: `midnight:mainnet`, `midnight:qanet` (simple, human-readable)
- b) Genesis hash: `midnight:000000000019d668...` (Bitcoin-style, immutable)
- c) Chain ID: `midnight:1`, `midnight:2` (Ethereum-style, numeric)
- d) Hybrid: `midnight:mainnet-{hash-prefix}` (readable + verifiable)

**Considerations**:
- Genesis hash is cryptographically verifiable but hard to remember
- Network name is user-friendly but could change (unlikely for mainnet)
- Midnight doesn't currently expose genesis hash as public constant
- Compatibility with existing tooling preferences

### 2. Namespace Registration

**Question**: How will `midnight` namespace be registered in the official CAIP-2 registry?

**Process**:
- Submit proposal to [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces)
- Define resolution method for reference portion
- Specify which networks use the namespace
- Timeline: 2-4 weeks for review and approval

**Considerations**:
- Can development proceed before official approval?
- Risk of namespace collision during interim period
- Need coordination with CAIP-2 editors

### 3. Backward Compatibility Strategy

**Question**: How will solutions support both legacy and CAIP-2 formats during transition?

**Approaches**:
- a) **Adapter layer**: Translate at boundaries, internal format unchanged
- b) **Dual support**: Accept both formats, canonicalize to CAIP-2 internally
- c) **Hard cutover**: Require migration, deprecate legacy format immediately
- d) **Phased migration**: 6-month grace period with warnings, then deprecation

**Considerations**:
- Impact on existing wallets and DApps
- Testing burden for dual support
- User communication and migration timeline
- Risk of extended technical debt

### 4. Ledger State Impact

**Question**: Does `network_id` embedded in ledger state need to change?

**Context**:
- `network_id: String` is serialized in `StandardTransaction` (Rust ledger)
- Changing format may invalidate historical ZK proofs
- May require hard fork at specific block height

**Scenarios**:
- a) **No ledger changes**: Network ID separate from blockchain consensus
- b) **Ledger fork**: Add version field, validate based on block height
- c) **Dual validation**: Accept both formats in proof verification

**Considerations**:
- Complexity of coordinated network upgrade
- Impact on historical transaction verification
- Validator upgrade requirements

### 5. Address Encoding Interaction

**Question**: Should CAIP-2 NetworkId affect Bech32m address format?

**Current**: Addresses use network-aware prefixes: `mn_addr1...` (mainnet), potential for `mn_addr_testnet1...`

**Options**:
- a) **Decouple**: NetworkId is metadata, address format unchanged
- b) **Embed**: Parse NetworkId from address (risks breaking validation regex)
- c) **Hybrid**: Address prefix implies network, CAIP-2 for API/storage only

**Considerations**:
- Bech32m validation currently rejects `:` character
- 50+ address test vectors would break if format changes
- User experience with long vs short addresses

### 6. API Versioning

**Question**: How should public APIs (HTTP, RPC) handle NetworkId format changes?

**Endpoints affected**:
- Indexer GraphQL queries
- Node RPC calls
- Proving service requests
- Wallet SDK APIs

**Strategies**:
- a) **API version header**: `X-API-Version: 2` for CAIP-2
- b) **Content negotiation**: Accept both, return client's preferred format
- c) **Deprecation warnings**: Old format works but returns warning headers
- d) **Dual endpoints**: `/v1/` vs `/v2/` paths

**Considerations**:
- Third-party API client breakage
- Documentation and migration guides
- Support burden for multiple API versions

### 7. Type Safety and Validation

**Question**: How should CAIP-2 format be enforced at compile time?

**Current**: NetworkId is `type NetworkId = WellKnownNetworkId | string` (weak typing)

**Improvements**:
- a) **Branded types**: TypeScript/Rust branded types enforce format
- b) **Parse, don't validate**: Constructor function validates CAIP-2 format
- c) **Regex validation**: Runtime check for `namespace:reference` pattern
- d) **Enum limitation**: Restrict to known networks only (lose extensibility)

**Considerations**:
- Developer experience and error messages
- Extensibility for future networks
- Performance of validation checks

## References

- [CAIP-2: Blockchain ID Specification](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md)
- [CAIP-10: Account ID Specification](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md)
- [CAIP-25: Provider Handshake](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-25.md)
- [Open Wallet Standard](https://github.com/open-wallet-standard/core)
- [WalletConnect Protocol](https://docs.walletconnect.com/)
- [Midnight NetworkId Implementation](https://github.com/midnight-ntwrk/midnight-code/blob/main/midnight-wallet/packages/abstractions/src/NetworkId.ts)

## Appendices

### Appendix A: OWS Integration Impact Analysis

Based on analysis of the Open Wallet Standard (OWS) codebase, integrating Midnight requires CAIP-2 compliance for the following reasons:

**OWS Chain Registry** (`ows-core/src/chain.rs`):
```rust
pub const KNOWN_CHAINS: &[Chain] = &[
    Chain { name: "ethereum", chain_id: "eip155:1" },
    Chain { name: "solana", chain_id: "solana:5eykt4..." },
    Chain { name: "bitcoin", chain_id: "bip122:00000000..." },
    // Midnight would require:
    Chain { name: "midnight", chain_id: "midnight:mainnet" },
];
```

Without CAIP-2 format, Midnight cannot be registered in OWS's chain registry, preventing integration.

### Appendix B: CAIP-2 Format Examples from Other Chains

| Chain | Namespace | Reference | Format | Notes |
|-------|-----------|-----------|--------|-------|
| Ethereum Mainnet | `eip155` | `1` | `eip155:1` | Numeric chain ID |
| Ethereum Goerli | `eip155` | `5` | `eip155:5` | Testnet uses different ID |
| Bitcoin | `bip122` | `000000000019d668...` | `bip122:000000000019d668...` | Genesis hash (truncated) |
| Solana | `solana` | `5eykt4UsFv8P8NJd...` | `solana:5eykt4UsFv8P8NJd...` | Genesis hash base58 |
| Cosmos Hub | `cosmos` | `cosmoshub-4` | `cosmos:cosmoshub-4` | Network name |
| Polkadot | `polkadot` | `91b171bb158e2d38...` | `polkadot:91b171bb158e2d38...` | Genesis hash |

**Recommendation for Midnight**: Use network name as reference (`midnight:mainnet`) for readability, following Cosmos pattern.

### Acknowledgements

- Analysis based on codebase exploration of midnight-code repository
- CAIP-2 specification by ChainAgnostic community
- Open Wallet Standard architecture review
- Cardano CIP-9999 (Problem Statements) for MPS template structure

## Copyright

This MPS is licensed under CC-BY-4.0 (Creative Commons Attribution 4.0 International).

---
**MPS Version**: 1.0
**Last Updated**: 2026-05-01
