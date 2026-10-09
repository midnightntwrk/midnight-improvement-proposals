---
MPS: <Number> # assigned by editors  
Title: Multi-environment Compact Releases  
Authors: Parisa Ataei (@pataei) 
Status: Proposed  
Category: Libraries and Tooling  
Created: 05-Oct-2026  
Requires: none  
Replaces: none  
MIP: none
---

<!--
 Copyright [YEAR] Midnight Foundation
 
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

The **Compact compiler** is used to compile smart contracts for the Midnight Network. The **Compact developer tools** (or "devtools") is used to install the compiler, to switch between compiler versions, and to invoke the compiler. The Midnight Network has [multiple deployed **environments**](https://docs.midnight.network/relnotes/network). For example, there is Midnight Mainnet itself, and also several official public testnets such as the Preproduction Testnet and the Preview Testnet. Each environment will normally be running its own specific versions of the Midnight Network software.

Currently, a versioned release of the Compact compiler is only able to compile contracts for a specific configuration of the Midnight Network software, and thus a specific environment. And Compact compiler only has a single release train. Thus, with the current settings, it is impossible to release the Compact compiler for different environments.

This MPS elaborates the issues caused by this restriction and describes the vision for how to address the problem.

## Vision

The Compact toolchain and developer tool provide a single release of the toolchain for different environments with an easy way for the user to choose the environment they need to compile their contract.

## Problem

The lack of a release of a single version of the Compact toolchain compatible with different Midnight networks causes lots of burden to the dApp developers on Midnight:

- They must figure out what version of the Compact toolchain is compatible with their desired network. For example, Compact compiler 0.31 supports the Mainnet environment and Compact compiler 0.33 supports the stagenet environment. The configurations for these environments can also change, meaning Compact compiler 0.31 which supports the Mainnet environment as of this writing may no longer support the Mainnet environment when the it is promoted to use a different version of the ledger. Thus, forcing the burden of picking the right compiler on the user makes developing on Midnight network less user-friendly. 
- They must wait months to access newer off-chain features of the Compact toolchain. Under the current settings, once the Compact compiler supports a newer environment it does not actively develop on the older environments. Thus, the dApp developers won't be able to use the new features until the environment they use is promoted to the new environment supported by a newer version of Compact compiler. Note that this delay is unnecessary as most of these changes are off-chain changes that do not rely on promoting an environment and they should be accessible instantly to Mainnet users.

## Use Cases

Take a partenr that requires standardization and consistency of different serialization formats in the Compact compiler by the time they are launching their product on Midnight's Mainnet. Midnight Mainnet environment relies on ledger 8.0, however, the Compact compiler is being developed actively on ledger 9.x and only security fixes are released on older versions of Compact compiler that are compatible with Mainnet. So if the Compact development team delivers the requested feature of the partner tomorrow, it will be on a non-Mainnet-compatible Compact compiler and the partner is forced to wait until the Mainnet environment is promoted to ledger 9.x. This creates lots of unnecessary friction and waiting on Midnight partners and users.

Right now, the said partner deploy their contracts on the Stagenet environment. But when Stagenet is promoted to say ledger 10.y they must remember that they need to switch their Compact compiler to a compatible version. This becomes harder to keep up with as the environments get promoted to different configurations more consistently and creates lots of unnecessary friction for the user.

The ideal user experience should be that they don't think about the compatibility of a network and Compact compiler. They simply tell the Compact developer tool that they want to compile their contract for a specific environment and the developer tool picks a compatible Compact compiler for them. If the user does not have a compatible Compact compiler installed they will get a clear message asking them to run a command to get the Compact compiler they need. This way the user does not have to carry the mental load of finding the compatibility matrix for each network every time they just need to compile a Compact contract. More importantly, Midnight partners and users do not have to wait to benefit from off-chain features until an environment is promoted to a certain configuration. This will unblock lots of essential features that have been developed or are being developed in the Compact compiler recently. 

## Goals

Deliver a release of Compact developer tool and toolchain that allows users to pick a compatible Compact compiler for any of the available Midnight environments without having to know what version of Compact is compatible with their desired environment. 

## Expected Outcomes

Partners and users won't have to wait for an environment promotion to access off-chain features of the Compact compiler. They also won't have to carry the burden of figuring out what version of the Compact compiler they need to compile their contract for a specific environment. This makes developing for the Midnight network more accessible and attractive.

## Recommended MIPs

The potential solution is addressed in this [CoIP](https://github.com/LFDT-Minokawa/compact/blob/main/coips/coip-0005.md) (Compact Improvement Process).

## Acknowledgements

- Kevin Millikin

## Copyright

This MPS is licensed under CC-BY-4.0.
