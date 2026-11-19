---
MPS: "xxxx" # to be assigned
Title: Post-Quantum Signature Verification in Compact
Authors:
- Quantova Inc (quantova-inc)
Status: Proposed
Category: Libraries and Tooling
Created: 19-Nov-2026
Requires: none
Replaces: none
MIP: none
---

<!--
 Copyright 2026 Midnight Foundation

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

A Compact contract that authorizes a party by signature verifies that signature inside its circuit.
The schemes available or requested for that today are all classical. Custody accounts in
[MIP-0013](/mips/mip-0013-account-authorisation.md) use Schnorr on JubJub,
[MIP-0030](/mips/mip-0030-ecdsa-account-authorisation.md) adds ECDSA, and
[MPS-0010](/mps/mps-0010-sig-verify-ecdsa.md) and [MPS-0009](/mps/mps-0009-sig-rsa.md) ask for
ECDSA and RSA verification. No path exists for a contract to verify ML-DSA
([FIPS 204](https://csrc.nist.gov/pubs/fips/204/final)) or SLH-DSA
([FIPS 205](https://csrc.nist.gov/pubs/fips/205/final)). Once the ledger accepts post quantum
keys under [MPS-0051](/mps/mps-0051-post-quantum-spending-authorization.md), a user can hold
unshielded funds under ML-DSA but cannot be a signer on a custody contract, and a contract cannot
check a credential or attestation signed by an issuer that has moved to a post quantum scheme. The
circuit library has SHA-256, SHA-512 and SHA3-256, which is enough for SLH-DSA with SHA2, but it has no
SHAKE output, which ML-DSA needs.

## Vision

A Compact developer verifies an ML-DSA or SLH-DSA signature with one library call, the same way
MPS-0010 asks for ECDSA, at a known circuit cost. Custody accounts, credential checks and oracle
contracts accept post quantum signers next to classical ones.

## Problem

**Custody and multisig.** MIP-0013 authorizes every asset releasing circuit with a Schnorr
signature on JubJub verified in circuit, and MIP-0030 adds ECDSA. A holder who moves to an ML-DSA
key under MPS-0051 has no way to stay a signer. The contract keys that protect custody balances stay
classical while the user's own ledger keys do not.

**Credentials and attestations.** Identity, credential and oracle contracts check signatures made
outside Midnight. The `midnight-zk` examples already prove an ECDSA signed credential in circuit
(`zk_stdlib/examples/identity/jwt`) and Bitcoin and Ethereum signatures. NIST IR 8547 marks quantum
vulnerable signatures at 112 bit security as deprecated after 2030 and all of them as disallowed
after 2035. Issuers that follow it will sign with ML-DSA or SLH-DSA, and a Midnight contract will not
be able to check what they sign.

**Missing primitive.** ML-DSA verification expands its public matrix from a seed with SHAKE128 and
hashes the message with SHAKE256. `midnight-zk` exposes `keccak_256_digest` and `sha3_256_digest`
on the Keccak chip (`zk_stdlib/src/external/keccak_sha3.rs`) but no SHAKE128 or SHAKE256 output of
arbitrary length. SLH-DSA with SHA2 needs only SHA-256 and SHA-512, which `circuits/src/hash`
already has, but no one has measured a full SLH-DSA verification as a circuit.

**Unknown cost.** An ML-DSA-65 verification works on a 6 by 5 matrix of polynomials of degree 256
over a 23 bit prime, plus the SHAKE calls that produce and check it. An SLH-DSA-SHA2-128s
verification is on the order of two thousand SHA-256 compressions. Neither has a published circuit size, proving
time or memory on Midnight's proof system, so a contract author cannot plan for either.

## Use Cases

- **Post quantum signer on a custody account.** A treasury holds funds in a MIP-0013 account and its
  signers want ML-DSA keys. Today one of them would have to keep a classical key to stay a signer.
- **Post quantum credential.** An issuer signs credentials with ML-DSA-65. A contract that grants
  access on a valid credential cannot verify it.
- **Oracle feed.** A data provider signs price reports with SLH-DSA for long term assurance. A
  contract that settles on those reports cannot check them.

## Goals

1. A Compact library function that verifies ML-DSA-44, ML-DSA-65 and ML-DSA-87 signatures as
   specified in FIPS 204, including the context string, and returns a Boolean. It passes the NIST
   ACVP sigVer vectors.
2. A Compact library function that verifies SLH-DSA signatures as specified in FIPS 205, at least
   for the SHA2 128 bit parameter sets, and passes the ACVP vectors.
3. SHAKE128 and SHAKE256 with arbitrary output length in the circuit library.
4. Published circuit size, proving time, proving memory and verification time for each function.
5. A post quantum signer type for custody accounts that composes with MIP-0013 and MIP-0030.

## Expected Outcomes

Contract authors can accept post quantum signers and post quantum issuers with a known cost.
Custody accounts can move with their users to post quantum keys. Midnight contracts stay able to
check external attestations as issuers follow NIST's schedule.

## Open Questions

- Should verification run in circuit or as a native ledger operation, as
  [MPS-0011](/mps/mps-0011-native-crypto-prims.md) considers for other primitives?
- Should a 1,952 byte ML-DSA-65 public key be a public input, or should the contract store a
  commitment to it and take the key as a private input?
- Can the expanded ML-DSA matrix be cached per key across calls without weakening the binding to
  its seed?
- Which parameter sets are worth their circuit cost for contracts?
- How does batch verification of several signers in one circuit compare with separate calls?

## Recommended MIPs

- **SHAKE in the circuit library.** Adds SHAKE128 and SHAKE256 with arbitrary output length on the
  existing Keccak chip. Addresses goal 3.
- **ML-DSA verification in Compact.** Defines the library function, its inputs and encodings, and its
  test vectors. Addresses goals 1 and 4.
- **SLH-DSA verification in Compact.** The same for SLH-DSA. Addresses goals 2 and 4.
- **Post quantum signers for custody accounts.** Extends MIP-0013 and MIP-0030 with an ML-DSA signer.
  Addresses goal 5.

## References

- [MIP-0013, Multi-key Account Authorisation for Custody Contracts](/mips/mip-0013-account-authorisation.md)
- [MIP-0030, ECDSA Authorisation for Custody Accounts](/mips/mip-0030-ecdsa-account-authorisation.md)
- [MPS-0009, RSA 1024/2048 Signature Verification](/mps/mps-0009-sig-rsa.md)
- [MPS-0010, ECDSA/secp256k1 Signature Verification](/mps/mps-0010-sig-verify-ecdsa.md)
- [MPS-0011, Native Cryptographic Primitives for Prime Stablecoin Integration](/mps/mps-0011-native-crypto-prims.md)
- [MPS-0051, Post-Quantum Spending Authorization for Unshielded Outputs](/mps/mps-0051-post-quantum-spending-authorization.md)
- [FIPS 202, SHA-3 and SHAKE](https://csrc.nist.gov/pubs/fips/202/final), [FIPS 204, ML-DSA](https://csrc.nist.gov/pubs/fips/204/final) and [FIPS 205, SLH-DSA](https://csrc.nist.gov/pubs/fips/205/final)
- [NIST IR 8547, Transition to Post-Quantum Cryptography Standards](https://csrc.nist.gov/pubs/ir/8547/ipd)

## Acknowledgements

None yet.

## Copyright

This MPS is licensed under CC-BY-4.0.
