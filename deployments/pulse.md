# Deployment Request: Pulse

**Pulse** is a hybrid OrderBook-AMM DEX for Midnight.

The current deployment is focused on the AMM and AMM orders, through which users can swap assets, add liquidity, remove liquidity, and receive LP tokens. The broader architecture is designed so orderbook-style settlement can be added later without changing the deployed AMM and order contracts.

The open-source contract is available here: https://github.com/pulse-finance/midnight-dex-contract

## Architecture

### Hybrid OrderBook-AMM

Pulse separates user-facing order placement from AMM settlement.

These are the 4 compact contracts we will deploy to mainnet:

- `MarketOrder.compact`, reusable *order contract* for token swaps
- `MintLpOrder.compact`, reusable *order contract* for adding liquidity and receiving LP tokens in return
- `BurnLpOrder.compact`, reusable *order contract* for removing liquidity by burning LP tokens
- `Amm.compact` for pool state, swap validation, LP minting and burning, fee accounting, and payouts.

Each *order contract* stores one active order at a time. A user opens an order by depositing the required shielded coin or coins into an empty order slot. The batcher then advances the order through the AMM lifecycle: reserve a pool slot, move the order coin into the AMM, validate the AMM state transition, pay the result back to the order contract, and close the order back to the user's return shielded address.

The same order and AMM contract interface can support later orderbook-style settlement. Orderbook matching and routing can be added in additional settlement logic while continuing to use the same AMM pool and reusable order-slot contracts.

The Pulse contracts only involve *shielded* assets.

### DApp

The DApp provides the user interface, wallet integration, transaction construction, proof generation/submission flow, pool discovery, order status display, and operational coordination for batch processing.

The DApp is not the source of truth for balances, pool reserves, or order settlement. Midnight contract state is the source of truth. DApp-side state is used to keep the interface responsive and to coordinate background processing, but settlement correctness is enforced by the contracts.

### Batcher and trust model

Pulse uses a batcher to advance multi-step AMM and order lifecycles.

The batcher is authorized in the AMM by a private `batcherSecret` witness whose persistent hash is stored as the on-chain `batcherCommitment`. Batcher-only circuits include AMM order placement, funding, merge operations, validation, payouts, fee reward collection, and fee/treasury updates.

The batcher is trusted for liveness, sequencing, and operational policy. It is not given any custody over user funds. It cannot arbitrarily withdraw user assets from the AMM or order contracts because the contracts enforce token colors, order states, AMM arithmetic, LP accounting, owner commitments, callback wiring, and payout routing.

If the batcher stalls, in-flight orders may wait for completion. This is treated as an operational liveness dependency, not as an admin custody issue.

## Data

Pulse stores protocol state required for AMM settlement:

- AMM token colors.
- AMM X/Y liquidity values.
- AMM LP circulating supply.
- AMM fee basis points and treasury destination.
- AMM accumulated X-side rewards.
- AMM pending order slots and active order state.
- Order contract state, order parameters, AMM slot assignment, and shielded coin positions.
- Owner commitments derived from user secrets for order-owner actions.

The contracts do not store real-world identity data, profile data, private messages, user names, email addresses, or other off-chain personal information.

Each user's full shielded address and order secrets are though persisted in the backend to be able to finalize user orders correctly in the background.

## Self-assessment

### Privacy-at-risk: 2

Pulse does not store real-world identities, profile data, messages, email addresses, or other personal records on-chain.

DEX activity does expose public protocol state needed for AMM operation, including token colors, pool reserves, order slot state, lifecycle state, fee configuration, and AMM settlement outputs. From this public state, swap sizes can be infered, but never be linked to specific individuals.

Full user shielded addresses are stored in the DApp backend, which means worst-case information leak from our DB leads to user's full shielded addresses becoming exposed. We think this is an acceptable risk given the improved UX offered by allowed backend order settlement (so users only have to sign the ingress transactions when performing swaps on Pulse, and not also the egress transactions). 

### Value-at-risk: 2

Pulse handles user assets through protocol-constrained AMM and order flows. The contracts are designed so users remain protected by contract logic throughout the order lifecycle rather than any admin custodying funds.

The batcher can advance order settlement and update AMM fee/treasury configuration, but it cannot arbitrarily withdraw user funds. User funds are fully protected by contract checks.

In-flight user funds sit in order contracts or AMM coin positions while orders are being processed, but those funds remain governed by the contract state machine. A stalled batcher only creates a liveness issue.

### State-space-at-risk: 2

AMM state grows with pending order slots, but AMM slot identifiers are bounded by `Uint<14>`. Reusable order contracts each hold one order at a time, and practical deployment uses a finite DApp-managed set of AMM pools and order slots.

State growth is therefore bounded by deployed contract count and AMM slot capacity rather than unbounded per-user ledger expansion.

## Testing and review summary

The contract repository includes unit and integration test coverage for AMM and order behavior:

- `test/unit/Amm.test.ts`
- `test/unit/MarketOrder.test.ts`
- `test/unit/MintLpOrder.test.ts`
- `test/unit/BurnLpOrder.test.ts`
- `test/integ/`

The repository also contains prior review notes under `audits/`.

## Requested outcome

Pulse is **mainnet ready** and we are requesting guarded mainnet deployment permission for the current AMM-based deployment.

Orderbook-style settlement is not part of the current deployment. It can be added later without changing the AMM and order contracts described in this request.

If this deployment request passes, we will use the provided deployment keys to deploy a limited number AMM and order contracts. End-users will only interact with these predeployed contracts, and will not need to deploy additional contracts themselves. 
