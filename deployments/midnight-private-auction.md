# Midnight Private Auction

## Description

A sealed-bid auction DApp built on Midnight using Compact's private state 
primitives. Bids are kept confidential during the auction period using 
zero-knowledge proofs — no participant or node operator can observe competing 
bids. After the deadline, the auctioneer closes the auction and the winner 
is revealed on the public ledger.

This is a developer demonstration project showcasing chain-native privacy 
primitives. It is not intended for production use with real financial assets 
at this stage.

## Risk Self-Assessment

### Privacy-at-risk: 1

Bid amounts are stored as private state and are never exposed on-chain. 
In the event of a ZK fault, the worst case is potential disclosure of a 
bid amount submitted by the affected user. No identity data, credentials, 
or external assets are involved.

### Value-at-risk: 1

This DApp holds no real financial value. It is a developer showcase with 
no token transfers or locked funds. There is nothing of monetary value 
that could be lost or extracted in an exploit scenario.

### State-space-at-risk: 1

The contract creates a small, bounded set of ledger entries: one auction 
record and one entry per bid. State growth is linear and predictable, 
controlled entirely by the auctioneer. There is no mechanism by which an 
attacker could cause unbounded state expansion.

## Repository

https://github.com/pplmaverick/midnight-private-auction

## Mitigations

No category scored above 1, so no additional mitigations are required 
at this stage.
