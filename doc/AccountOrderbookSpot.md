# Spot Orderbook Account

## Use Case
Spot orderbook account holds the state of the spot market.

## Schema and Key of QMDB

```Rust

//! Spot Market Order Book Account
//!
//! Spot books settle atomically: a matched fill moves base and quote tokens
//! between the two counterparties' `ExternalAccount`s in the same write.
//! There is no position account, no funding, and no resolution step, so the
//! book only needs the two price trees and the order queue.
//!
//! The book is identified by its own account `Address` (the account that
//! stores this struct). Each order is identified by the hash of the
//! transaction that carried it, so no separate order id allocator or
//! sequence counter is needed.
//!
//! QMDB keys:
//!   ob/<address>/meta          -> {SpotBookAccount}
//!   ob/<address>/bid/<price>   -> {PriceLevel}
//!   ob/<address>/ask/<price>   -> {PriceLevel}
//!   ob/<address>/order/<tx_hash> -> {Order}

// ---------------------------------------------------------------------------
// Shared types (move to `clob/common.rs` when the other books are split out)
// ---------------------------------------------------------------------------

/// The price of the order.
type Price = u64;
/// The quantity of the order.
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

/// The order state
#[repr(u8)]
pub enum OrderStatus {
    /// New order opened.
    Open = 0,
    /// PartiallyFilled order.
    PartiallyFilled = 1,
    /// Filly filled order.
    Filled = 2,
    /// Cancelled order. Cancellation comes from User or from the match engine
    /// in an lazy way, for example the expiration of an order.
    Cancelled = 3,
}

/// The STP policy.
#[repr(u8)]
pub enum STPMode {
    /// When STP is enabled, a self-trade event will cancel the maker order.
    CancelMaker,
    /// When STP is enabled, a sef-trade event will cancel the taker order.
    CancelTaker,
    /// When STP is enabled, a sef-trade event will cancel both the taker and maker orders.
    CancelBoth,
}

/// The spot book config, when the field is None, the engine will not run the rule.
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
}

// ---------------------------------------------------------------------------
// Spot book account
// ---------------------------------------------------------------------------

/// The Spot book account. The account address itself is the book id.
/// key: ob/<address>/meta -> value: {SpotBookAccount}; It could be refined by breaking it into individual fields to
/// avoid the write amplification on the QMDB side.
#[repr(C)]
pub struct SpotBookAccount {
    /// Symbol of the book.
    symbol: Symbol,
    /// The token being bought or sold.
    base_token: TokenID,
    /// The token used to price and pay for the base token.
    quote_token: TokenID,

    /// Book config, once loaded would rarely be updated. Store it as an object in QMDB.
    config: BookConfig,

    // ---- Top-of-book / aggregate state ----
    /// The best bid price, it links to the price level with the price level key.
    best_bid_price: Option<Price>,
    /// The best ask price, it links to the price level the the price level key.
    best_ask_price: Option<Price>,
    /// The num of bid levels.
    bid_level_count: u32,
    /// The num of ask levels.
    ask_level_count: u32,
    /// Total bid quantity.
    total_bid_qty: Qty,
    /// Total ask quantity.
    total_ask_qty: Qty,
    /// Is the market place be paused.
    is_paused: bool,
    // ---- Reference data for display and analytics ----
    last_traded_price: Option<Price>,
    /// Cumulative base-token volume over the book's lifetime.
    total_base_volume: Qty,
    /// Cumulative quote-token volume over the book's lifetime.
    total_quote_volume: Qty,
}

// ---------------------------------------------------------------------------
// Price level and order
// ---------------------------------------------------------------------------

/// Price level:
/// It could be break into individual field k-v to avoid write amplification.
/// key: ob/<address>/bid/<price> -> value: {PriceLevel};
/// key: ob/<address>/ask/<price> -> value: {PriceLevel};
///
/// For bid levels, `next_price` is the next lower price and `prev_price` is
/// the next higher price. For ask levels, `next_price` is the next higher
/// price and `prev_price` is the next lower price.
#[repr(C)]
pub struct PriceLevel {
    price: Price,
    total_qty: Qty,
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
/// key: ob/<address>/order/<tx_hash> -> value: {Order}; It can be refined to avoid write amplification.
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