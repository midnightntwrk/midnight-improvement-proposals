---
MPS: "0042"  
Title: Stable, Versioned and Well-Specified Serialization Format  
Authors: Karmel (Karmoola)  
Status: Proposed    
Category: Standards    
Created: 17-Sept-2026  
Requires: none  
Replaces: none   
MIP: none  
---

<!-- Copyright 2026 Midnight Foundation Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at https://www.apache.org/licenses/LICENSE-2.0 Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License. -->

## **Abstract**

Midnight lacks a single, stable, and formally specified serialization format for Compact values. The current Field-Aligned Binary (FAB) encoding is documented as unstable, yet it underpins persistentHash, a guarantee that promises consistent hashing across protocol upgrades. This is a contradiction. Teams building on Midnight, including cross-chain bridging protocols, serialize data into events and must reproduce that exact serialization off chain to verify signed attestations. With no committed specification, they reverse engineer the encoding from observed behavior, which is fragile and introduces security risk into deployed contracts. Compounding this, the compiler carries two serialized representations, one for ledger storage and one for events, that are not guaranteed identical, the published specification already diverges from the implementation, and the in-circuit encoding is inefficient at scale. The absence of a stable, documented format blocks safe third-party development on Midnight and risks a partner's home-grown implementation becoming the de facto standard.

## **Vision**

Midnight offers one compiler-controlled serialization format that external teams can build on with confidence. It is formally specified, versioned, and stable at each version, with a reference implementation that reproduces in-circuit behavior exactly. Cross-chain protocols and other integrators serialize and hash data with the assurance that their integrations will not silently break across protocol upgrades, and that any incompatibility is detected before production rather than after.

## **Problem**

Serialization on Midnight is split across mechanisms. The Midnight layer uses a custom, informally versioned approach, while the extrinsic side uses scale encoding. Two serialization concerns must be separated: the ledger-internal storage encoding, which is internal and free to evolve, and the compiler-controlled encoding used to serialize data into the ledger and into events, which is the format external teams observe and depend on. Event data falls under the second. Today the compiler carries two representations, one for ledger storage and one for events, and they are not guaranteed identical. persistentHash hashes the ledger encoding.

A stability guarantee rests on an unstable format. FAB is documented as subject to change, while persistentHash promises any non-opaque value hashes to the same digest across future versions. The hash operates only on FAB bytes and carries no version information, so if FAB changes the guarantee silently breaks, with no compatibility policy attached.

The specification itself notes limits can shift with a minor version increment, so the format cannot be safely built on as written.

External teams are forced to reverse engineer the format. With no reproducible specification, teams derive off-chain libraries by observing the in-circuit function's output. In the typical flow, serialized data is rehashed on chain, an attestation signed by a multi-party computation is verified against that hash, and the bundle is deserialized inside the circuit so correctness can be proven on chain. The exact serialization must therefore be reproducible off chain, including for a plain bytes object passed to a hash function. Any mismatch becomes a security issue in a deployed contract.

The in-circuit encoding is inefficient. Serialization and deserialization run inside the circuit and are costly, so large data volumes risk real performance problems and lock the ecosystem into a format that does not scale.

The format is many-to-one. Distinct Compact objects can produce the same serialized output. A trusted schema plus a signed attestation manages this today, but it remains a latent hazard, for example a serialized vector being reinterpreted as a struct.

## **Use Cases**

- A cross-chain bridging protocol serializes transfer data into a Midnight event, has the attestation signed by a multi-party computation, and needs to reproduce the identical serialization in an off-chain service to verify the signature. Today it must reverse engineer the encoding, and a single-byte discrepancy silently breaks verification or, worse, admits invalid data.
- A team migrating an integration across a protocol upgrade needs to know whether a value it hashed before the upgrade still hashes to the same digest after. Without version information on the format, it cannot tell, and cannot plan a safe migration.
- A third-party developer wants an official, documented serialization library rather than depending on another partner's reverse-engineered implementation, which may be inefficient or insecure yet become the de facto standard by default.

## **Goals**

- A single compiler-controlled serialization format, with the events and ledger-storage encodings unified intentionally.
- A version indicator on the format: the current format is designated version one, known versions are processed, and unknown versions are rejected rather than misinterpreted, surfacing during testnet before production. A single byte suffices, with a reserved value signaling that a longer word follows.
- A real stability guarantee per version: a given version is committed and does not change.
- A formal specification, as a written description, a reference implementation that replicates in-circuit behavior exactly, or both, covering the type-configured nature of serialization and the encoding of a bytes object.
- An efficient in-circuit encoding, addressed after the initial versioning and specification work.
- A defined migration path, with announced breaking changes, best practices, and a transition period supporting both old and new versions so teams do not lose access to on-chain data.

## **Expected Outcomes**

Cross-chain and other integrating teams build against a committed, reproducible target, removing the reverse-engineering risk and the class of security bugs it creates. persistentHash guarantees become meaningful, because unifying the two encodings makes it immaterial which one is hashed, and versioning makes any format change explicit rather than silent. Incompatibilities are caught at testnet rather than in production. Efficient in-circuit encoding keeps the format viable as data volumes grow. Midnight gains an official serialization standard, avoiding the risk of an inefficient or insecure implementation becoming the default.
