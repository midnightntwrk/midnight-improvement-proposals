# Native Cryptographic Primitives for Prime Stablecoin Integration

## Abstract

This MIP proposes exposing two cryptographic capabilities: **Keccak-256 hashing** and **ECDSA/Secp256k1 signature recovery** as native Compact smart contract primitives to support launching Prime Stablecoin on Midnight.

Launching Prime Stablecoin on Midnight requires that Compact smart contracts can (1) hash a serialized messages using Ethereum-native Keccak-256 and (2) recover a signer address from an ECDSA signature over Secp256k1. These two primitives unlock the a deposit intent lifecycle: serialization → hashing → signing → on-chain verification.

## Motivation

Prime Stablecoin's minting flow is initiated by a message that must be serialized off-chain, hashed, signed by an authorized party, and verified on-chain before minting proceeds. 

These Compact capabilities will allow for a message, as shown below to be processed by a Compact smart contract. As an example in [this](https://gist.github.com/yvonnezhangc/205c2bf28ec7f80ee6dbc7a9001773d9) document details the steps partners must take to serialize, hash, sign, and validate a `DepositIntent` message that initiates the minting of stablecoin. 

## Specification

### Capability 1: Keccak-256 Hashing

Compact smart contracts must support hashing of a message using Keccak-256.

Example snippet: 
```
bytes memory encodedDepositIntent = /* concatenation above */;
bytes32 digest = keccak256(encodedDepositIntent);
// 0x5e4cd56161653333155634cc8565cbaff8254f9dec29bd9b1ff8c864aac569fc
```

---

### Capability 2: ECDSA Signature Recovery

Compact smart contracts must support recovering a signer address from an ECDSA signature over Secp256k1. The verifier requires only the 65-byte signature and re-derives the public key and address via `recover`.

Example snippet: 
```
bytes32 hash = keccak256(blob);          // same as above
address signer = ECDSA.recover(hash, signature);
require(signer == expectedSigner, "invalid sig");
```

### Supporting Issues

- [Compact #91](https://github.com/LFDT-Minokawa/compact/issues/91)
- [Compact #104](https://github.com/LFDT-Minokawa/compact/issues/104)
