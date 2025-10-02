<!--
 Copyright 2025 Midnight Foundation

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

---
MIP: <MIP Number> (assigned by a MIP Editor)
Title: Contract Metadata Convention
Authors: Andrew Fleming <@andrew-fleming>
Status: Proposed
Category: Standards
Created: 2025-10-02
Requires: [None]
Replaces: [None]
---

## Abstract

This proposal establishes a standardized naming convention for exposing contract ledger state to off-chain services on the Midnight network.
Due to Midnight's architecture requiring proof generation for circuit calls,
traditional "free" getter functions are infeasible for wallets and explorers to query contract metadata.
This standard solves this problem by defining a prefix-based naming pattern for ledger fields that allows off-chain services to discover and read contract state from the ledger without generating proofs.
This convention provides the foundation for token standards (such as fungible and non-fungible tokens) and any other contracts requiring discoverable metadata.

## Motivation

In many blockchain ecosystems, off-chain services can freely query contract state through view functions—for example,
a wallet calling `ERC20.balanceOf(address)` to display a user's token balance.
Midnight's architecture fundamentally differs: circuit calls require proof generation,
making it impractical for wallets, block explorers, and other off-chain services to query metadata for every user interaction.

Without a standardized approach, each contract could expose metadata differently,
forcing wallet developers to implement custom logic for every token or contract type.
This fragmentation would create significant barriers to ecosystem growth and poor user experience.

This standard establishes a uniform ledger naming convention that:

- Enables wallets to automatically discover and display token balances.
- Allows block explorers to identify and properly format contract interactions.
- Provides a foundation for cross-chain bridges to integrate with Midnight tokens.
- Ensures forward compatibility as new contract standards emerge.
- Reduces integration burden for ecosystem developers.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Naming Convention

All standard-compliant ledger fields **MUST** follow this pattern:

```ts
<STANDARD_NAME>__<fieldName>
```

Where:

- `<STANDARD_NAME>` is the identifier for the standard e.g. `MRC123`, `MRC456`, `MRC789`.
- `<STANDARD_NAME>` MUST use uppercase letters e.g. `MRC` not `mrc`.
- `__` is a double underscore separator.
- `<fieldName>` is the specific field name.

### Examples

```ts
// MIP-X Fungible Token Standard
export ledger MRCX__balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>;
export ledger MRCX__allowances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>>;
export sealed ledger MRCX__name: Opaque<"string">;
export sealed ledger MRCX__symbol: Opaque<"string">;

// MIP-Y NFT Standard
export ledger MRCY__owners: Map<Uint<128>, Either<ZswapCoinPublicKey, ContractAddress>>;
export ledger MRCY__tokenURIs: Map<Uint<128>, Opaque<"string">>;

// MIP-Z Custom protocol
export ledger MRCZ__liquidityPools: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>;
```

### Reserved Prefixes

The following naming pattern is RESERVED and MUST NOT be used except as specified:

1. `MRC<number>__` - Reserved for official Midnight Request for Comment standards.
    - Example: `MRC22__`, `MRC333__`, `MRC4444__`

### Discovery Mechanism

Off-chain services can discover which standards a contract implements by:

- Querying the contract's ledger state.
- Inspecting field names for standard prefixes.
- Matching against known standard patterns.

```ts
// Pseudo-code for off-chain service
function getImplementedStandards(contractState) {
  const standards = new Set();
  for (const fieldName in contractState) {
    const match = fieldName.match(/^(MRC\d+)__/);
    if (match) standards.add(match[1]);
  }
  return standards;
}
```

A contract implementing multiple standards will have ledger fields with multiple prefixes:

```ts
// A wrapped token contract might implement multiple standards
// (MRCX and MRCY are placeholder names for illustration—exact MRC number TBD)
export ledger MRCX__balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>;
export ledger MRCX__name: Opaque<"string">;
export ledger MRCY__deposits: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>;
export ledger MRCY__depositTimestamps: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<64>>;
```

### Field Visibility

Standards SHOULD specify whether fields are:

- `sealed ledger` - Immutable after initialization.
- `ledger` - Mutable contract state.

This information helps off-chain services optimize caching strategies.

### Type Compatibility

Standards MUST specify exact types for ledger fields to ensure consistent parsing by off-chain services.

## Rationale

Several separators were considered.

### Single Underscore (MRCX_fieldName)

Single underscores are commonly used within variables.
This can easily conflict with existing naming conventions.
Furthermore, a single underscore is less visually distinctive.

### Dot Notation (MRCX.fieldName)

Dot notation may cause parsing issues in some contexts.
This may also create semantic conflicts with property access in some languages.

### Double Underscore (MRCX__fieldName)

The double-underscore prefix is visually distinctive and provides clear separation.
Using double underscores is uncommon in typical variable naming thus reduces conflicts.
Programmatically, this approach is easy to parse.
Finally, it follows established conventions such as name mangling in Python.

### Prefix (MRCX__fieldName) > Suffix (fieldName__MRCX)

Prefixes are the better choice because:

- Off-chain services can quickly filter fields by standard.
- Standards will be grouped together when alphabetically sorted.
- Easier to read.

### Alternative Approaches Considered

#### Special character prefix

The use of `$` or other special character domains e.g. `$fieldName`.
This may cause semantic conflicts with other languages.

#### Dedicated Storage Struct

Contracts export a ledger API interface which enforces the required fields:

```ts
struct MRCX_Storage {
  balances: Map<Either<ZswapCoinPublicKey, ContractAddress>, Uint<128>>
  name: Opaque<"string">
  ...
}

export ledger Storage: MRCX_Storage;
```

Struct fields cannot be ADT types.

This is an interesting option though as it could easily enable abstraction strategies and improve dx such as config files and/or decorators.

## Copyright Waiver

All contributions (code and text) submitted in this MIP must be licensed under the Apache License, Version 2.0
