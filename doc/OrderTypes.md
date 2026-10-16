# Order Types
This is the order types shared by different marketplace within this protocol.

## Schemas

```Rust
pub enum OrderType {
    /// Standard limit order.
    #[default]
    Standard,

    /// Iceberg order with hidden quantities.
    Iceberg { hidden_quantity: Quantity },

    /// Post only order that won't match immediately.
    PostOnly,

    /// Trailing stop order that adjusts with market movement
    TrailingStop {
        /// Amount to trail the market price, in price ticks.
        trail_amount: Price,

        /// Last reference price.
        last_ref_price: Price,
    },

    /// Pegged order that adjusts based on reference price
    Pegged {
        /// Offset from the reference price.
        reference_price_offset: i64,

        /// Type of reference price to track.
        reference_price_type: PegReferenceType,
    },

    /// Market-to-limit order that converts to limit after initial execution
    MarketToLimit,

    /// Reserve order with custom replenishment
    /// if `replenish_amount` is None, it uses DEFAULT_RESERVE_REPLENISH_AMOUNT
    /// if `auto_replenish` is false, and visible quantity is below threshold, it will not replenish
    /// if `auto_replenish` is false and visible quantity is zero it will be removed from the book
    /// if `auto_replenish` is true, and replenish_threshold is 0, it will use 1
    ReserveOrder {
        /// The hidden quantity of the order.
        hidden_quantity: Quantity,

        /// Threshold at which to replenish
        replenish_threshold: Quantity,

        /// Optional amount to replenish by, in quantity units. If `None`, uses
        /// [`DEFAULT_RESERVE_REPLENISH_AMOUNT`]. A replenish amount is
        /// structurally non-zero ([`NonZeroU64`]): a zero replenish would draw
        /// an empty visible tranche from hidden.
        replenish_amount: Option<NonZeroU64>,

        /// Whether to replenish automatically when below threshold.
        /// If false, only replenish on next match
        auto_replenish: bool,
    },
}```