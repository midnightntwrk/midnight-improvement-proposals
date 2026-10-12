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

The **Compact compiler** is used to compile smart contracts for the Midnight Network. The **Compact developer tools** (or "devtools") is used to install the compiler, to switch between compiler versions, and to invoke the compiler. The Midnight Network has [multiple deployed **environments**](https://docs.midnight.network/relnotes/network). For example, there is Midnight Mainnet itself and also several official public testnets such as the Preproduction Testnet and the Preview Testnet. Each environment runs a specific **configuration** of the Midnight Network software.

In Midnight Network, smart contracts are compiled off chain. What ties them to the chain is the data and its structure in the generated code of the compiler. But not every compiler feature touches the part of the generated code that is tied to the chain which we call  **off-chain features**. Unfortunately, the current release of the Compact compiler is tied to a single configuration. Thus, a single release of the Compact compiler cannot be used for different environments if they use different configurations. This delays when a user can access a feature, even an off-chain feature, depending on which environment they require and whether a Compact compiler implementing their required feature exists for their configuration.

This MPS elaborates on the issues caused by this restriction and describes the vision for how to address the problem.

## Vision

Releases of the Compact compiler supporting multiple environments and the devtools providing an easy way for the user to choose the environment for each run of the compiler.

## Problem

The lack of a release of the Compact compiler compatible with different Midnight environments burdens Midnight dApp developers:

- They must figure out what version of the Compact compiler is compatible with their desired network. For example, Compact compiler 0.31 supports the Mainnet environment and Compact compiler 0.33 supports the stagenet environment. The configurations for these environments can also change, meaning Compact compiler 0.31 which supports the Mainnet environment as of this writing may no longer support the Mainnet environment when it is promoted to use a different version of the ledger. Thus, forcing users to pick the right compiler makes developing on Midnight Network less user-friendly. 
- They must wait months to access newer off-chain features of the Compact compiler. Under the current settings, once the Compact compiler supports a newer environment it no longer actively develops for older configurations. Thus, the dApp developers won't be able to use the new features until the environment they use is promoted to the new environment supported by a newer version of Compact compiler. Note that this delay is unnecessary as most of these changes are off-chain changes that do not rely on promoting an environment and they should be accessible immediately to Mainnet users.

## Use Cases

One of our partners requires the off-chain feature of standardization and consistency of different serialization formats in the Compact compiler by the time they launch their product on Midnight's Mainnet. The Midnight Mainnet environment relies on ledger 8.0; however, the Compact compiler is being actively developed on ledger 9.x. Once Compact compiler development moves to a newer version of the ledger, older versions of the Compact compiler that support Mainnet receive only bug fixes. So if the Compact development team delivers the partner's requested feature tomorrow, it will land on a non-Mainnet-compatible Compact compiler, and the partner will be forced to wait until Mainnet is promoted to ledger 9.x. This creates unnecessary friction and waiting for Midnight partners and users. It is already forcing Mainnet users to wait months to access newer Compact features such as various cryptographic curves, contract-to-contract calls, and emitting events.

Right now, the partner deploys their contracts on Stagenet, which uses ledger 9.x. But when Stagenet is promoted to, say, ledger 10.y, they must remember to switch their Compact compiler to a compatible version. This becomes harder to keep up with as environments are promoted to different configurations more frequently, and it creates a lot of unnecessary friction for users.

The ideal user experience should be that users do not think about the compatibility of a network and Compact compiler. They simply tell the Compact developer tool that they want to compile their contract for a specific environment and the developer tool picks a compatible Compact compiler for them. If the user does not have a compatible Compact compiler installed, they will get a clear message asking them to run a command to get the Compact compiler they need. This way the user does not have to carry the mental load of finding the compatibility matrix for each network every time they just need to compile a Compact contract. More importantly, Midnight partners and users do not have to wait to benefit from off-chain features until an environment is promoted to a certain configuration. This will unblock many essential features that were recently developed or are under development in the Compact compiler recently. 

## Goals

- Deliver a release infrastructure for the Compact compiler and devtools that allows users to pick a compatible Compact compiler for any of the available Midnight environments without having to know what version of Compact is compatible with their desired environment. 
- Deliver the first release of the Compact compiler that supports multiple configurations. 

## Expected Outcomes

Partners and users won't have to wait for an environment promotion to access off-chain features of the Compact compiler. They also won't have to carry the burden of figuring out what version of the Compact compiler they need to compile their contract for a specific environment. This makes developing for the Midnight Network more accessible and attractive.

## Recommended MIPs

The potential solution is addressed in this [CoIP](https://github.com/LFDT-Minokawa/compact/blob/main/coips/coip-0005.md) (Compact Improvement Process).

## Acknowledgements

Kevin Millikin and Kent Dybvig for useful discussions on the problem and potential solutions. 

## Copyright

This MPS is licensed under CC-BY-4.0.
