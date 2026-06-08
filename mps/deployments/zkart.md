**dApp name:** zkArt

**Contract repository:** https://github.com/jonathanfishbein1/zkArt-demo

**Brief description:**

zkArt is a privacy-preserving digital art auction platform on Midnight. Sellers
mint a single NFT per artwork and escrow it into a sealed-bid auction contract.
Bidders submit a shielded zkArt token during a time-bounded bidding window, with
the bid amount committed via hash and revealed in a separate window to determine
the winner. Bidder identities are derived from local secret keys and never
published; the seller identity is stored in sealed contract state. After the
auction closes the winner claims the shielded NFT, the auction house claims a
10 % platform fee, the seller claims the remaining pot, and losing bidders
reclaim their escrowed tokens.

Three Compact contracts make up a deployed auction:

- **Token contract** (deployed once per platform instance) — a mint-capped
  shielded fungible token used as the bid denomination. Distribution for this
  demo is via a faucet circuit; the token has no on-chain peg to any external
  asset and is intended only for in-platform bidding.
- **NFT contract** (one per auction) — mints a single shielded coin whose color
  uniquely identifies the artwork; stores artwork metadata (name, description,
  image URI, media type, edition) on the public ledger.
- **Auction contract** (one per auction) — manages the full auction lifecycle:
  NFT deposit, shielded-token bid escrow into a single accumulated `bid_pot`,
  commit-reveal winner determination, NFT transfer to winner, payment split to
  seller and auction house, and refunds to non-winners.

---

| Category            | Self-assessed score (1–3) |
| ------------------- | :-----------------------: |
| Privacy-at-risk     |             1             |
| Value-at-risk       |             2             |
| State-space-at-risk |             1             |

---

### Privacy-at-risk — 1

**Rationale:**

Bidder and seller identities are sealed. The on-chain `winning_bidder_key` is not
a wallet address — it is a `persistentHash` of a random 32-byte secret (generated
via `crypto.getRandomValues`, stored only in the user's `localStorage`) mixed with
an auction-specific tag (`"auction:pk:"` + `nft_color`). This key is unlinkable to
the user's wallet, real-world identity, or their participation in other auctions.
Seller identity, NFT color, the auction house public key, and the bid-token color
are sealed contract fields. Bid amounts are hidden during the commit phase via
hash commitments and become visible during the reveal phase, but art auction bids
are pseudonymous financial signals in a public auction context, devoid of
real-world harm. Artwork metadata (name, image URI, edition) is public by design.
Bid token transfers themselves are shielded — escrow, refunds, fee payout, and
seller payment all flow as shielded coin operations against the contract's
`bid_pot`, so amounts and counterparties are not visible in the transfer
graph. No data exposed by the contract links to real-world identities or reveals
private behavioral patterns.

**Mitigations:**

- Per-auction key derivation (`persistentHash([tag, auction_id, sk])`) ensures
  on-chain keys are auction-scoped and unlinkable across contracts. The
  underlying secret key never leaves the browser's `localStorage`.
- Bid amounts are hidden during the commit phase via hash commitments, and even
  once revealed they pose no privacy risk beyond what any public auction
  inherently discloses.
- Bids are denominated in a shielded zkArt token, so the escrow flow into
  `bid_pot` (and the subsequent refund / fee / payout sends) uses shielded coin
  ops — bid-amount visibility on-chain is limited to the public reveal step,
  not to the underlying transfers.

---

### Value-at-risk — 2

**Rationale:**

Bidders' shielded zkArt tokens are escrowed in the contract for a bounded period
between bid submission and auction close. The escrow window is defined by
`end_time` and `winner_check_end_time` (set at contract deployment). All bidders'
deposits are accumulated into a single `bid_pot` (a
`QualifiedShieldedCoinInfo` updated via `mergeCoinImmediate`). After the auction
closes, the auction house's fee and the seller's payment are sent from the pot,
and losing bidders reclaim their full deposit. Funds do not accumulate across
auctions — each auction is an independent contract instance. The bid token is
purpose-minted for the platform and not redeemable against any external asset,
so the absolute value at risk is bounded by the token's in-platform utility.

**Mitigations:**

- Time-bounded escrow with explicit deadlines enforced on-chain.
- Seller escape hatch (`reclaim_nft`) prevents permanent NFT lock if no valid
  bids are placed (gated on `highest_bid_amount == 0`).
- Two-phase refund (`prepare_refund` zeroes the bidder's commitment + records
  the owed amount; `claim_refund` sends from `bid_pot`) prevents double-spend
  by clearing state before the coin op.
- Three-phase disbursement (`close_auction` validates and stages the 10 % fee →
  `claim_house_fee` sends to the house key → `prepare_seller_payment` →
  `claim_seller_payment` sends the remainder to the seller) keeps each
  fund-moving circuit within ZSwap's per-circuit effects budget while
  ensuring fee and seller payouts are mutually gated.
- Fee math is checked on-chain (`fee*10 ≤ highest_bid_amount` and
  `(fee+1)*10 > highest_bid_amount`) so the floor-division 10 % split cannot be
  inflated.
- Each auction is a separate contract deployment, so an exploit in one instance
  cannot drain funds from another.

---

### State-space-at-risk — 1

**Rationale:**

Each auction contract is an independent deployment with a bounded lifecycle, and
after the auction closes no new state entries can be created. The only growing
structure is the `bid_commitments` map (one entry per unique bidder key); the
escrowed value itself is collapsed into a single `bid_pot` coin rather than a
per-bidder map. Growth of `bid_commitments` is constrained by three structural
bounds: each per-auction-derived key can create at most one map entry (enforced
by a `bid_commitments.member` check in `reserve_bid_slot`), the bidding window
has a fixed deadline (`end_time`), and each auction is an isolated contract
instance whose state does not accumulate across auctions. A `minimum_bid` adds
modest additional friction — an attacker must temporarily lock tokens per entry
— though since escrowed amounts are refundable after close, the real deterrent
is opportunity cost rather than permanent loss. The NFT contract enforces a
single mint; the bid-token contract is mint-capped at deployment.

**Mitigations:**

- One entry per key: the `bid_commitments.member` check in `reserve_bid_slot`
  ensures each derived key can create exactly one map entry, preventing a
  single wallet from inflating state.
- Single accumulated `bid_pot` (via `mergeCoinImmediate`) means escrowed value
  does not grow a per-bidder map; only the commitment map grows, and it is
  capped by the one-entry-per-key bound.
- Auction time windows impose a deadline on state growth — no entries can be
  added after `end_time`.
- One contract per auction ensures no cross-auction state accumulation.
- Minimum bid enforcement raises the cost of state-spam by requiring attackers
  to temporarily lock tokens per entry, though tokens are refundable after the
  auction closes.
- The single-mint constraint on the NFT contract caps metadata storage at one
  entry; the bid-token contract's `max_supply` caps total token issuance.
- State-clearing is user-initiated: bidders call `prepare_refund` /
  `claim_refund` which zero out their commitment and send the escrowed amount
  out of `bid_pot`.
