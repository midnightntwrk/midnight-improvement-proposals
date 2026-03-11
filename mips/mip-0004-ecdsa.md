# ECDSA Signature Support to enable custodians integrations

 ECDSA Signature Integration for prime custodians support 

This MIP proposes extending the Midnight ledger to accept ECDSA signatures and BIP-44 address derivation, enabling institutional custody providers to hold and transact Midnight assets.

Midnight must support ECDSA signatures at the protocol level to unblock institutional adoption. This requires extending the ledger to accept both Schnorr and ECDSA signatures for transaction authorization, implementing BIP-44 compatible address derivation, and ensuring that all unshielded transaction types mint, burn, and transfer  are compatible with the ECDSA signature model.

## Motivation

Institutional asset managers, exchanges, and token issuers require Prime Custodians to hold and transact Midnight assets, including minting, burning, and transferring both shielded and unshielded tokens. Without ECDSA support, these institutions cannot use their existing custody infrastructure (ex. Fireblocks, Bitgo) with Midnight. Enabling this integration removes a critical adoption blocker and positions Midnight as enterprise ready.

## Specification

### Dual Signature Scheme Support (Schnorr + ECDSA)

Extend the Midnight ledger to accept both Schnorr and ECDSA signatures for transaction authorization. The ledger must determine the signature type from the transaction metadata and route validation accordingly.

### ECDSA Address Derivation (BIP-44)

Implement ECDSA-compatible address derivation following the BIP-44 standard path `m/44'/coin_type'/account'/change/index`. This enables Fireblocks to derive and manage Midnight addresses using its existing wallet infrastructure.

### Unshielded Transaction Workflow Support

All unshielded Midnight transaction types — mint, burn, and transfer — must be compatible with the ECDSA signature model.

**Example existing transaction workflow:** 

https://developers.fireblocks.com/docs/transaction-flows

Resources:
- ECDSA/EdDSA Algorithms: ncw-developers.fireblocks.com/docs/multiple-algorithms
- Create Transaction API: developers.fireblocks.com/reference/create-transactions
- API Reference: https://developers.fireblocks.com/reference/api-overview
- MPC-CMP Protocol: developers.fireblocks.com/docs/what-is-fireblocks