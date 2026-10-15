# Orderbook Accounts

## Use Case
Orderbook account holds the state of the orderbook of different types of market, spot, perp, and prediction product.

## Schema and Key of QMDB

```Rust
/// CLOB Accounts

type BookId = u64;
type OrderId = u64;
type Price = u64;
type Qty = u128;
type EventId = u64;
type OutcomeId = u8;

#[repr(u8)]
pub enum Side {
    Bid = 0,
    Ask = 1,
}

#[repr(u8)]
pub enum OrderStatus {
    Open = 0,
    PartiallyFilled = 1,
    Filled = 2,
    Cancelled = 3,
}

#[repr(u8)]
pub enum Resolution {
    Unresolved = 0,
    Yes = 1,
    No = 2,
    Invalid = 3,
}

#[repr(C)]
pub struct BookConfig {
    /// Minimum price increment.
    tick_size: Price,
    /// Minimum quantity increment.
    lot_size: Qty,
    /// Maker fee in basis points.
    maker_fee_bps: u16,
    /// Taker fee in basis points.
    taker_fee_bps: u16,
    /// Minimum order quantity.
    min_qty: Qty,
    /// Maximum resting orders per book.
    max_orders: u32,
}

#[repr(C)]
pub struct OrderFlags {
    /// Perp only: order may only reduce an existing position.
    reduce_only: bool,
    /// Order must rest as maker; cancel if it would take.
    post_only: bool,
    /// Perp only: order originates from the liquidation engine.
    is_liquidation: bool,
}

/// Market-specific state. Common book state lives in `CLOBAccount`.
///
/// The variant resolves the settlement precompile contracts and carries
/// the fields that only make sense for that market type.
pub enum MarketType {
    /// Spot market: atomic settlement, no position risk.
    Spot {
        base_token: TokenID,
        quote_token: TokenID,
    },

    /// Perp market: needs funding, mark/index prices, open interest, and
    /// risk-engine coupling.
    Perp {
        settlement_token: TokenID,
        index_oracle: Address,
        funding_interval: u64,
        /// Scaled integer, e.g. 1e8.
        funding_rate: i64,
        next_funding_time: u64,
        open_interest: Qty,
        mark_price: Price,
        index_price: Price,
        max_leverage: u32,
    },

    /// Prediction market book. One book per outcome token.
    /// `complement_token` is the opposite outcome token for the same event.
    /// Price is interpreted as implied probability in [0, 1].
    Prediction {
        settlement_token: TokenID,
        event_id: EventId,
        outcome_id: OutcomeId,
        outcome_token: TokenID,
        complement_token: TokenID,
        expiry: u64,
        resolution: Resolution,
    },
}

/// The CLOB Account and the state to be managed by QMDB:
/// key: ob/<book_id>/meta -> value: {CLOBAccount};
#[repr(C)]
pub struct CLOBAccount {
    /// Unique book id.
    book_id: BookId,
    /// Human-readable symbol, e.g. "BTC-USDC".
    symbol: Symbol,
    /// Market type plus market-specific state.
    market_type: MarketType,
    /// Common book configuration.
    config: BookConfig,
    /// Best bid price, if any.
    best_bid_price: Option<Price>,
    /// Best ask price, if any.
    best_ask_price: Option<Price>,
    /// Number of active bid price levels.
    bid_level_count: u32,
    /// Number of active ask price levels.
    ask_level_count: u32,
    /// Cached total bid quantity across all levels.
    total_bid_qty: Qty,
    /// Cached total ask quantity across all levels.
    total_ask_qty: Qty,
    /// Next order id to allocate.
    next_order_id: OrderId,
    /// Monotonic sequence for deterministic replay.
    sequence: u64,
    /// Paused flag.
    is_paused: bool,
}

/// Price level:
/// key: ob/<book_id>/bid/<price> -> value: {PriceLevel};
/// key: ob/<book_id>/ask/<price> -> value: {PriceLevel};
///
/// For bid levels, `next_price` is the next lower price and `prev_price` is
/// the next higher price.
/// For ask levels, `next_price` is the next higher price and `prev_price` is
/// the next lower price.
#[repr(C)]
pub struct PriceLevel {
    price: Price,
    total_qty: Qty,
    order_count: u32,
    /// Oldest order at this level (time priority head).
    head_order_id: OrderId,
    /// Newest order at this level (time priority tail).
    tail_order_id: OrderId,
    next_price: Option<Price>,
    prev_price: Option<Price>,
}

/// Order:
/// key: ob/<book_id>/order/<order_id> -> value: {Order};
#[repr(C)]
pub struct Order {
    order_id: OrderId,
    book_id: BookId,
    side: Side,
    price: Price,
    /// Remaining quantity.
    remaining_qty: Qty,
    original_qty: Qty,
    owner: Address,
    /// Next order in the FIFO queue at this price level.
    next_order_id: Option<OrderId>,
    /// Previous order in the FIFO queue at this price level.
    prev_order_id: Option<OrderId>,
    timestamp: u64,
    status: OrderStatus,
    flags: OrderFlags,
}
```