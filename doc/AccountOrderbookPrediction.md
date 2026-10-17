# Prediction Orderbook Account

## Use Case

Prediction orderbook account holds the state of one prediction market. A market lists a
set of mutually exclusive outcomes (two for a binary market); each outcome trades as a
share of the settlement token priced in `[0, 1]`. A share pays out 1 unit at resolution
if its outcome is the reported winner, and 0 otherwise.

Unlike the perp book, there is no leverage, no funding, and no liquidation anywhere in
the model:

- **A buy is fully collateralized** — the buyer pays `price` per share up front.
- **A sell is fully collateralized** — the seller either holds the shares, or mints the
  complete set first: depositing 1 unit of collateral mints one share of *every* outcome,
  so selling one side locks the rest of the set. Selling YES at price `p` costs `1 - p`
  net and wins `p` if YES loses — the prediction-market "short".
- **The market can never be undercollateralized** — the invariant that every outcome's
  outstanding shares equal the minted complete sets means the locked collateral always
  covers the maximum possible payout of any single outcome.

Trades on the outcome books settle atomically like spot: shares move between the two
counterparties' `OutcomeBalance` entries while settlement tokens move in the opposite
direction in the same write. Resolution is the only non-trading state change: the oracle
reports the winner, winning shares redeem 1:1 against the locked collateral, and losing
shares are burned.

The absence of the perp machinery (clearinghouse, margin summaries, liquidation sweeps)
is deliberate: the share-balance model needs no global account structure, so each market
is self-contained.

## Schema and Key of QMDB

The market is identified by its own account `Address`. Each outcome is a sub-book under
the market, addressed by its `OutcomeID`. Each order is identified by the hash of the
transaction that carried it, so no order id allocator or sequence counter is needed.

```Rust
//! QMDB keys:
//! pred/<address>/meta -> {PredictionBookAccount}
//! pred/<address>/outcome/<outcome_id>/meta -> {OutcomeBook}
//! pred/<address>/outcome/<outcome_id>/bid/<price> -> {PriceLevel}
//! pred/<address>/outcome/<outcome_id>/ask/<price> -> {PriceLevel}
//! pred/<address>/outcome/<outcome_id>/order/<tx_hash> -> {Order}
//! pred/<address>/outcome/<outcome_id>/balance/<owner> -> {OutcomeBalance}

/// The price of an outcome share. Scaled by 1e6 and bounded to [0, 1]:
/// 500000 means a 0.5 probability of the outcome. The bounds are enforced
/// by `BookConfig::min_price` / `max_price`.
type Price = u64;
/// The quantity of the order (shares).
type Qty = u64;
/// The notional amount of the order.
type Notional = u128;
/// The settlement token balance. Shares redeem 1:1 against it.
type Balance = u128;

/// Transaction hash that carries an order.
type TxHash = [u8; 32];
/// The TokenID is the RWA pool account address.
type TokenID = Address;

/// The outcome of a prediction market. Index into the market's outcome
/// sub-books: 0 and 1 for a binary market, 0..n for categorical.
type OutcomeID = u8;

/// Trade side.
#[repr(u8)]
pub enum Side {
    /// Buyer side.
    Bid = 0,
    /// Seller side.
    Ask = 1,
}

/// The order state.
#[repr(u8)]
pub enum OrderStatus {
    /// New order opened.
    Open = 0,
    /// Partially filled order.
    PartiallyFilled = 1,
    /// Fully filled order.
    Filled = 2,
    /// Cancelled order. Cancellation comes from the user or from the match
    /// engine in a lazy way, for example the expiration of an order.
    Cancelled = 3,
}

/// The STP policy.
#[repr(u8)]
pub enum STPMode {
    /// When STP is enabled, a self-trade event will cancel the maker order.
    CancelMaker,
    /// When STP is enabled, a self-trade event will cancel the taker order.
    CancelTaker,
    /// When STP is enabled, a self-trade event will cancel both the taker
    /// and maker orders.
    CancelBoth,
}

/// The lifecycle of a prediction market.
#[repr(u8)]
pub enum MarketStatus {
    /// Orders can be placed and matched.
    Trading = 0,
    /// The winner has been reported; books are closed and shares resolve.
    Resolving = 1,
    /// The redemption sweep is complete; the market is settled.
    Resolved = 2,
}

/// Last event applied to an outcome share balance.
#[repr(u8)]
pub enum BalanceEvent {
    /// Shares minted from a complete set.
    Minted = 0,
    /// Shares burned into a complete set.
    Merged = 1,
    /// Shares bought on the book.
    Bought = 2,
    /// Shares sold on the book.
    Sold = 3,
    /// Winning shares redeemed 1:1 against locked collateral.
    Redeemed = 4,
    /// Losing shares burned at resolution.
    Burned = 5,
}

/// The prediction book config. When a field is `None`, the engine will not
/// run the corresponding rule.
#[repr(C)]
pub struct BookConfig {
    /// Minimum price increment.
    tick_size: Option<Price>,
    /// Minimum quantity increment.
    lot_size: Option<Qty>,
    /// Maker fee in basis points.
    maker_fee_bps: Option<u16>,
    /// Taker fee in basis points.
    taker_fee_bps: Option<u16>,
    /// Minimum order quantity.
    min_qty: Option<Qty>,
    /// Maximum order quantity.
    max_qty: Option<Qty>,
    /// Maximum notional amount.
    max_notional: Option<Notional>,
    /// Maximum resting orders per outcome book.
    max_orders: Option<u32>,
    /// STP policy.
    stp: Option<STPMode>,
    /// Minimum order price. Defaults to 0; bounds the share price from
    /// below.
    min_price: Option<Price>,
    /// Maximum order price. Defaults to 1 scaled (1e6); bounds the share
    /// price from above. A buy priced above the payout would be a
    /// donation, so the bound is a safety rule.
    max_price: Option<Price>,
    /// Creator reward on trading fees, in basis points, if set.
    creator_fee_bps: Option<u16>,
}

// ---------------------------------------------------------------------------
// Prediction market account
// ---------------------------------------------------------------------------

/// The prediction market account. The account address itself is the market
/// id. It carries the market-wide state: resolution data, the complete-set
/// mint/merge ledger, and the outcome count. The order books and share
/// balances live per outcome, under the sub-keys.
/// key: pred/<address>/meta -> value: {PredictionBookAccount};
/// It could be refined by breaking it into individual fields to avoid
/// write amplification on the QMDB side.
#[repr(C)]
pub struct PredictionBookAccount {
    /// Symbol of the market.
    symbol: Symbol,
    /// Hash of the off-chain question that this market resolves.
    question_hash: TxHash,
    /// The token used to price shares and pay out winners.
    settlement_token: TokenID,
    /// Oracle account address that reports the winning outcome.
    resolution_oracle: Address,
    /// Block height at or after which the oracle may report the winner.
    resolution_time: u64,

    /// The market lifecycle state.
    market_status: MarketStatus,
    /// The winning outcome, set when the market enters `Resolving`.
    /// `None` while trading.
    resolved_outcome: Option<OutcomeID>,

    /// Number of outcomes in this market (2 for binary, 0..n for
    /// categorical). The outcome sub-books are addressed 0..outcome_count.
    outcome_count: u8,

    /// Book config, once loaded would rarely be updated. Store it as an
    /// object in QMDB.
    config: BookConfig,

    // ---- Complete set (mint/merge) ledger ----
    /// Collateral locked by minted complete sets, one unit per set. This
    /// is the payout reserve: the maximum possible payout of any single
    /// outcome equals `minted_sets`, which never exceeds this balance.
    collateral_locked: Balance,
    /// Number of complete sets minted. Must equal the outstanding share
    /// total of every outcome.
    minted_sets: Qty,

    // ---- Reference data for display and analytics ----
    /// Cumulative collateral collected as trading fees over the market's
    /// lifetime.
    total_fees_collected: Balance,
    /// Cumulative notional traded across all outcomes.
    total_notional_volume: Notional,
    /// Is the market paused.
    is_paused: bool,
}

// ---------------------------------------------------------------------------
// Outcome sub-book
// ---------------------------------------------------------------------------

/// The outcome sub-book, one per outcome:
/// key: pred/<address>/outcome/<outcome_id>/meta -> value: {OutcomeBook};
///
/// This is the book half of the market for one outcome: a spot-style order
/// book (bids, asks, orders) plus the share-balance ledger of its holders.
/// Fills move shares and settlement tokens between the two counterparties'
/// `OutcomeBalance` entries; no position or margin state is involved.
#[repr(C)]
pub struct OutcomeBook {
    /// The outcome this book trades. Part of the key.
    outcome_id: OutcomeID,

    // ---- Top-of-book / aggregate order state ----
    /// The best bid price, links to the price level with the price level key.
    best_bid_price: Option<Price>,
    /// The best ask price, links to the price level with the price level key.
    best_ask_price: Option<Price>,
    /// The number of bid levels.
    bid_level_count: u32,
    /// The number of ask levels.
    ask_level_count: u32,
    /// Total bid quantity.
    total_bid_qty: Qty,
    /// Total ask quantity.
    total_ask_qty: Qty,

    // ---- Reference data for display and analytics ----
    last_traded_price: Option<Price>,
    /// Cumulative share volume over the book's lifetime.
    total_share_volume: Qty,
    /// Cumulative notional volume over the book's lifetime.
    total_notional_volume: Notional,

    // ---- Share-balance ledger ----
    /// Number of holders with a non-zero balance in this outcome.
    holder_count: u32,
    /// Total outstanding shares of this outcome. Invariant: equals the
    /// market's `minted_sets` at all times.
    outstanding_shares: Qty,

    // ---- Persistent linked list (source of truth in QMDB) ----
    /// Head of the holder linked list. `None` when no holders.
    head_holder: Option<Address>,
    /// Tail of the holder linked list. `None` when no holders.
    tail_holder: Option<Address>,

    // ---- Populated in-memory view ----
    /// The populated holder balances, built by walking the linked list
    /// from `head_holder` and following `next_owner`. Not persisted as a
    /// standalone structure; it mirrors the linked list for O(1) lookup.
    /// Redemption sweeps walk the list, not this map.
    /// key: pred/<address>/outcome/<outcome_id>/balance/<owner> -> value: {OutcomeBalance};
    balances: Map<Address, OutcomeBalance>,
}

/// Outcome share balance:
/// key: pred/<address>/outcome/<outcome_id>/balance/<owner> -> value: {OutcomeBalance};
///
/// One entry per holder per outcome. Shares are fully paid: there is no
/// margin, no liquidation, and no funding, so the entry is just a share
/// quantity plus lifetime cash-flow counters. The entry is removed when
/// `shares` reaches zero.
#[repr(C)]
pub struct OutcomeBalance {
    /// The share holder. Also the key under which the entry is stored.
    owner: Address,
    /// Previous holder in the linked list. `None` when head.
    prev_owner: Option<Address>,
    /// Next holder in the linked list. `None` when tail.
    next_owner: Option<Address>,

    /// Shares held in this outcome.
    shares: Qty,
    /// Cumulative collateral paid by this holder for these shares (buys
    /// and mints).
    total_paid: Balance,
    /// Cumulative collateral received by this holder (sells, merges, and
    /// redemptions).
    total_received: Balance,

    /// Last event applied to this balance.
    last_event: BalanceEvent,
    /// Block height of first entry.
    opened_at: u64,
    /// Block height of the last update.
    updated_at: u64,
}

// ---------------------------------------------------------------------------
// Price level and order
// ---------------------------------------------------------------------------

/// Price level:
/// It could be broken into individual field k-v to avoid write amplification.
/// key: pred/<address>/outcome/<outcome_id>/bid/<price> -> value: {PriceLevel};
/// key: pred/<address>/outcome/<outcome_id>/ask/<price> -> value: {PriceLevel};
///
/// For bid levels, `next_price` is the next lower price and `prev_price` is
/// the next higher price. For ask levels, `next_price` is the next higher
/// price and `prev_price` is the next lower price.
#[repr(C)]
pub struct PriceLevel {
    price: Price,
    total_qty: Qty,
    /// Sum of notional at this level, used for depth analytics.
    total_notional: Notional,
    order_count: u32,
    /// Oldest order at this level (time-priority head).
    head_order_hash: TxHash,
    /// Newest order at this level (time-priority tail).
    tail_order_hash: TxHash,
    /// Pointer to the next price level.
    next_price: Option<Price>,
    /// Pointer to the previous price level.
    prev_price: Option<Price>,
}

/// Order:
/// key: pred/<address>/outcome/<outcome_id>/order/<tx_hash> -> value: {Order};
/// It can be refined to avoid write amplification.
#[repr(C)]
pub struct Order {
    /// Side of the order: Ask or Bid.
    side: Side,
    /// Price of the order.
    price: Price,
    /// Type of the order.
    type: OrderType,
    /// Remaining quantity.
    remaining_qty: Qty,
    /// Original quantity.
    original_qty: Qty,
    /// The trader's account address.
    owner: Address,
    /// Next order in the FIFO queue at this price level.
    next_order_hash: Option<TxHash>,
    /// Previous order in the FIFO queue at this price level.
    prev_order_hash: Option<TxHash>,
    /// Timestamp of the block height on which the order's carrier TXN is mined.
    timestamp: u64,
    /// The status of the order.
    status: OrderStatus,
}
```

## Engine Notes

- **Fill** — spot-style atomic transfer: the buyer's `OutcomeBalance.shares` increases
  and the seller's decreases by the matched quantity, while the settlement tokens move
  from buyer to seller in the same write; fees are taken in settlement tokens and the
  outcome counters and market volumes are updated. The buyer's coverage is their own
  settlement-token balance, and the seller's coverage is their share balance — there is
  no margin check, because both sides are fully collateralized by construction.
- **Mint complete set** — the caller deposits `qty` units of the settlement token and is
  credited one share of *every* outcome per unit: `collateral_locked += qty` and
  `minted_sets += qty`. Minting is how a sell without holdings is opened: mint the set,
  sell the outcome you are bearish on, keep the rest as locked collateral.
- **Merge complete set** — the caller burns one share of *every* outcome per unit and is
  released `qty` units of the settlement token. The reverse of minting.
- **Resolution** — at or after `resolution_time`, the oracle sets `resolved_outcome` and
  the market enters `Resolving`; the books stop taking orders. The winning outcome's
  holder list is swept: each holder redeems 1:1 from `collateral_locked`; losing
  outcomes burn their shares. After the sweep, `Resolved`.
- **Solvency invariant** — for every outcome `i`,
  `Σ owners balances(i).shares == minted_sets`, and `collateral_locked == minted_sets`.
  The payout of any single outcome is therefore fully reserved at all times: a
  prediction market cannot be undercollateralized, by construction rather than by a
  liquidation engine. The invariant is checked on every mint, merge, and fill, and at
  resolution.
- **No clearinghouse** — the perp machinery (global clearinghouse, margin summaries,
  liquidation sweeps) is deliberately absent. Non-leveraged per-outcome balances are
  self-contained, so there is no global account structure to size or cache.

## Prediction State Machine

```Text
Market:
  ┌─────────┐  oracle reports winner  ┌───────────┐  redemption sweep done  ┌──────────┐
  │ Trading │ ──────────────────────▶ │ Resolving │ ──────────────────────▶ │ Resolved │
  └─────────┘                         └───────────┘                         └──────────┘

Shares of an outcome (per holder):
                 mint complete set / buy             sell / merge complete set
        ┌──────┐ ────────────────────────────▶ ┌─────────┐ ────────────────────────────▶ ┌──────┐
        │ Flat │                               │ Holding │                               │ Flat │
        └──────┘ ◀──────────────────────────── └─────────┘ ◀──────────────────────────── └──────┘
                     redeem 1:1 (winning outcome) / burn (losing outcome) at resolution
```
