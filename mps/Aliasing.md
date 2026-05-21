# Aliasing

Midnight users are exposed to three addresses derived from one seed: Shielded, Unshielded, and Dust. Asking users to manage three addresses produces high error rates and friction during onboarding. Several ecosystem partners (Midnames, 1AM, ..) are independently building naming layers, which risks namespace fragmentation: two providers could allow the same human readable name to be registered against different accounts or resolve the same name to different addresses across wallets, indexers, and dApps.

This MPS identifies the absence of a single, authoritative standard for binding human readable names to Midnight accounts and to all wallet addresses derived from those accounts (Shielded, Unshielded, Dust, and Sig.Network-derived external-chain addresses).

## Vision

A user who completes Midnight onboarding receives one human readable name for example, Alice under the canonical Midnight namespace that resolves uniformly across every Midnight wallet, dApp, indexer, block explorer, and partner service. The name is the user's single public handle. It routes to the correct underlying address for the operation being performed: shielded transfers go to the Shielded address, unshielded operations to the Unshielded address, DUST sponsorship to the Dust address, and cross-chain deposits to the Sig.Network-derived address on the destination chain. The user doesn't see, copy, or distinguish between these underlying addresses unless they explicitly opt into power user mode.

Two users cannot hold the same name simultaneously. When a user recovers their account on a new device, their existing name re-points to the recovered account without becoming available for re-registration by anyone else.

## Problem

The current state of Midnight account addressing creates four compounding problems.

**1. Multiple addresses per user**

**2. Uncoordinated naming providers:** Without a network-level standard, each may publish a registry contract with its own uniqueness rules, its own resolver interface, and its own dispute semantics. Wallets and dApps will then have to choose which provider(s) to support and a user named Alice in one resolver may not be the same Alice in another.

**3. No authoritative duplicate prevention:** Uniqueness of human readable names must hold at any point in time. If two providers each maintain independent state, "uniqueness" only holds within a provider, not across the network. There is no current network-level primitive that guarantees a name is bound to exactly one account globally, on-chain.

## Use Case

**First-time receive.** Alice completes onboarding, receives the name Alice and shares it with Bob over a messaging app. Bob pastes Alice into his Midnight wallet and sends her shielded tokens. He never sees a raw address. The wallet resolves alice to Alice's Shielded address, regardless of whether Bob is using Lace, 1am, or any other conforming wallet.

## Goals

1. **One name per user, resolving everywhere:** A single human-readable name MUST be sufficient for any send-receive-identify operation a user performs on Midnight, abstracting over Shielded/Unshielded/Dust and Sig.Network-derived external-chain addresses.
2. **Global uniqueness, enforced on-chain:** At any point in time, a given name MUST resolve to at most one account, and this property MUST be verifiable from chain state.
3. **Recovery-safe:** Account recovery flows MUST preserve name ownership across device loss
4. **Phishing resistant rendering.** The standard MUST give wallets and dApps enough information to safely render names in UI without each implementer reinventing the rules.