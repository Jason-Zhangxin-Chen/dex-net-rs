# Perpetual Orderbook Account

## Use Case

Perpetual orderbook account holds the state of the perpetual future market.
It has two coupled sub-structures:

- The **order book** holds unmatched orders (bids and asks) plus the
  aggregate top-of-book state.
- The **position book** holds crossed orders that have formed contracts
  between traders. Positions persist across matches until closed,
  liquidated, ADL'd, or settled.

Unlike spot, a fill in the perp book does not move tokens between accounts.
Instead, it creates or updates a `PerpPosition` for each side. Settlement
against the settlement token only happens at close, liquidation, or ADL.

## Schema and Key of QMDB

The book is identified by its own account `Address`. Each order is
identified by the hash of the transaction that carried it, so no order id
allocator or sequence counter is needed. Positions are sub-keys under the
book namespace, keyed by the trader's `Address`.

```Rust
//! QMDB keys:
//! perp/<address>/meta -> {PerpBookAccount}
//! perp/<address>/bid/<price> -> {PriceLevel}
//! perp/<address>/ask/<price> -> {PriceLevel}
//! perp/<address>/order/<tx_hash> -> {Order}
//! perp/<address>/position/<owner> -> {PerpPosition}

/// The price of the order.
type Price = u64;
/// The quantity of the order (contracts).
type Qty = u64;
/// The notional amount of the order.
type Notional = u128;

/// Transaction hash that carries an order.
type TxHash = [u8; 32];
/// The TokenID is the RWA pool account address.
type TokenID = Address;

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

/// Margin mode for a perp position.
#[repr(u8)]
pub enum MarginMode {
    /// Margin is isolated to a single position.
    Isolated = 0,
    /// Margin is shared across the account's positions.
    Cross = 1,
}

/// Direction of a locked perp contract.
#[repr(u8)]
pub enum PositionSide {
    /// Long position.
    Long = 0,
    /// Short position.
    Short = 1,
}

/// Last event applied to a position.
#[repr(u8)]
pub enum PositionEvent {
    Opened = 0,
    Increased = 1,
    Decreased = 2,
    Closed = 3,
    Liquidated = 4,
    ADL = 5,
    Settled = 6,
}

/// The perp book config. When a field is `None`, the engine will not run
/// the corresponding rule.
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
    /// Maximum resting orders per book.
    max_orders: Option<u32>,
    /// STP policy.
    stp: Option<STPMode>,
    /// Maximum leverage allowed for any position in this book, scaled by 1e4.
    /// E.g. 200000 means 20x.
    max_leverage: Option<u32>,
    /// Maintenance margin ratio in basis points.
    maintenance_margin_bps: Option<u16>,
    /// Initial margin ratio in basis points.
    initial_margin_bps: Option<u16>,
    /// Liquidation fee in basis points.
    liquidation_fee_bps: Option<u16>,
}

// ---------------------------------------------------------------------------
// Perp market type
// ---------------------------------------------------------------------------

/// Perp market metadata. The index oracle and funding schedule are the two
/// things that make a perp distinct from a spot book.
#[repr(C)]
pub struct PerpMarket {
    /// The token used to settle PnL and margin.
    settlement_token: TokenID,
    /// Oracle account address that publishes the index price.
    index_oracle: Address,
    /// Funding interval in seconds, e.g. 8h.
    funding_interval: u64,
    /// Current funding rate, scaled, signed.
    funding_rate: i64,
    /// Block height of the next funding application.
    next_funding_time: u64,
    /// Cumulative funding index, used for position-level attribution.
    cumulative_funding_index: i128,
    /// Mark price used for margin and liquidation. Refreshed by the risk
    /// engine, not by matches. The median of (index_price+basis, index_price+MA)
    mark_price: Price,
    /// The oracle account of the index price.
    index_oracle: Address,
}

// ---------------------------------------------------------------------------
// Perp book account
// ---------------------------------------------------------------------------

/// The Perp book account. The account address itself is the book id.
/// key: perp/<address>/meta -> value: {PerpBookAccount};
/// It could be refined by breaking it into individual fields to avoid write
/// amplification on the QMDB side.
#[repr(C)]
pub struct PerpBookAccount {
    /// Symbol of the book.
    symbol: Symbol,
    /// Perp market metadata.
    market: PerpMarket,

    /// Book config, once loaded would rarely be updated. Store it as an
    /// object in QMDB.
    config: BookConfig,

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
    /// Is the market place paused.
    is_paused: bool,

    // ---- Reference data for display and analytics ----
    last_traded_price: Option<Price>,
    /// Cumulative contract volume over the book's lifetime.
    total_contract_volume: Qty,
    /// Cumulative notional volume over the book's lifetime.
    total_notional_volume: Notional,

    // ---- Embedded position book ----
    //
    // `position_book` carries the aggregate counters plus the map handle
    // into the per-owner entries. Reading `position_book.positions` gives
    // the L3 position view; reading only the counters gives the OI view
    // without touching the entry sub-tree.
    position_book: PositionBook,
}

// ---------------------------------------------------------------------------
// Embedded position book
// ---------------------------------------------------------------------------

/// Aggregate + entry view of the perp book's position state.
#[repr(C)]
pub struct PositionBook {
    /// Number of non-zero positions currently open in this book.
    position_count: u32,
    /// Total open long contracts.
    total_long_size: Qty,
    /// Total open short contracts.
    total_short_size: Qty,
    /// Sum of long notional at entry.
    total_long_notional: Notional,
    /// Sum of short notional at entry.
    total_short_notional: Notional,
    /// Cumulative open interest in contracts. Equal to
    /// `total_long_size` when the book is balanced, kept separately so
    /// mismatches surface during reconciliation.
    open_interest: Qty,
    /// Sum of all positive unrealized PnL across positions. Refreshed by
    /// the risk engine.
    total_unrealized_pnl: i128,
    /// Sum of all isolated margin locked by positions in this book.
    total_isolated_margin: Balance,
    /// Number of positions currently under liquidation.
    liquidation_count: u32,
    /// Number of positions that have been ADL'd.
    adl_count: u32,

    /// The populated position entries from the underlying QMDB:
    /// key: "perp/<address>/position/<owner>" -> value: {PerpPosition};
    ///
    /// Present once the book is created. Every crossing fill can lock a
    /// contract for any owner, so the map is always `Some`.
    positions: Map<Address, PerpPosition>,
}

/// Perp position entry:
/// key: perp/<address>/position/<owner> -> value: {PerpPosition};
///
/// One net position per (book, owner). A trader cannot be both long and
/// short in the same perp market; opposing fills reduce or flip the
/// position. The entry is removed when `size` reaches zero.
#[repr(C)]
pub struct PerpPosition {
    /// The position owner.
    owner: Address,
    /// Long or Short. Flipped when an opposing fill exceeds `size`.
    side: PositionSide,
    /// Open contract count, always non-negative. Zero means closed.
    size: Qty,
    /// Average entry price.
    entry_price: Price,
    /// Margin mode.
    margin_mode: MarginMode,
    /// Isolated margin locked for this position. Unused when cross.
    isolated_margin: Balance,
    /// Cumulative funding index paid or received by this position.
    funding_index: i128,
    /// Last funding time applied.
    last_funding_time: u64,
    /// Current liquidation price under the current margin mode.
    liquidation_price: Price,
    /// Realized PnL over the position's lifetime.
    realized_pnl: i128,
    /// Mark-to-market unrealized PnL, refreshed by the risk engine.
    unrealized_pnl: i128,
    /// Effective leverage under the current margin, scaled by 1e4.
    leverage: u32,
    /// True once the position has been liquidated or fully ADL'd.
    is_liquidated: bool,
    /// Last event applied to this position.
    last_event: PositionEvent,
    /// Block height when the position was opened.
    opened_at: u64,
    /// Block height of the last update.
    updated_at: u64,
}

// ---------------------------------------------------------------------------
// Price level and order
// ---------------------------------------------------------------------------

/// Price level:
/// It could be broken into individual field k-v to avoid write amplification.
/// key: perp/<address>/bid/<price> -> value: {PriceLevel};
/// key: perp/<address>/ask/<price> -> value: {PriceLevel};
///
/// For bid levels, `next_price` is the next lower price and `prev_price` is
/// the next higher price. For ask levels, `next_price` is the next higher
/// price and `prev_price` is the next lower price.
#[repr(C)]
pub struct PriceLevel {
    price: Price,
    total_qty: Qty,
    /// Sum of open interest at this level, used for depth analytics.
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
/// key: perp/<address>/order/<tx_hash> -> value: {Order};
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
    /// True if the order may only reduce an existing position.
    reduce_only: bool,
    /// True if the order originated from the liquidation engine.
    is_liquidation: bool,
}