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
    /// Funding interval in blocks, e.g. a period of 8h in blocks.
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

    // ---- Embedded clearing vault book ----
    clearing_vault: ClearingVault,
}

// ---------------------------------------------------------------------------
// Embedded clearinghouse vault book
// ---------------------------------------------------------------------------

/// The token used as collateral in this book.
type Balance = u128;

/// Last event applied to a vault entry.
#[repr(u8)]
pub enum VaultEvent {
    /// Margin deposited into the book.
    Deposited = 0,
    /// Margin withdrawn from the book.
    Withdrawn = 1,
    /// Margin locked as isolated margin for a new or increased position.
    MarginLocked = 2,
    /// Margin released when an isolated position is reduced or closed.
    MarginReleased = 3,
    /// Realized PnL credited or debited.
    PnlRealized = 4,
    /// Funding credited or debited.
    FundingSettled = 5,
    /// Position liquidated; margin seized to cover the shortfall / surplus.
    Liquidated = 6,
    /// ADL executed; margin and PnL settled at bankruptcy price.
    AdlSettled = 7,
    /// Insurance fund credited from a liquidation surplus.
    InsuranceCredited = 8,
    /// Insurance fund debited to cover a liquidation shortfall.
    InsuranceDebited = 9,
}

/// The clearinghouse vault book. Traders deposit the settlement token into
/// the book; the vault bookkeeps each trader's claim on the pool as
/// `free_margin` + `locked_isolated_margin` + `pending_withdrawal`.
///
/// Persistent state lives in the doubly-linked list (`head_owner` /
/// `tail_owner` plus `prev_owner` / `next_owner` on each entry). The
/// `vaults` map is populated from that list at load time so settlement,
/// funding, and risk can do O(1) random access by owner.
#[repr(C)]
pub struct ClearingVault {
    /// Number of non-empty vault entries.
    vault_owner_count: u32,

    // ---- Aggregate balances ----
    /// Sum of `free_margin` across all vaults.
    total_free_margin: Balance,
    /// Sum of `locked_isolated_margin` across all vaults. Should equal
    /// the sum of `isolated_margin` across all open isolated positions.
    total_locked_margin: Balance,
    /// Sum of `pending_withdrawal` across all vaults.
    total_pending_withdrawal: Balance,

    // ---- Lifetime flow counters ----
    /// Cumulative settlement tokens deposited over the book's lifetime.
    total_deposited: Balance,
    /// Cumulative settlement tokens withdrawn over the book's lifetime.
    total_withdrawn: Balance,
    /// Net realized PnL credited to vaults over the book's lifetime.
    net_realized_pnl: i128,
    /// Net funding paid/received across vaults over the book's lifetime.
    net_funding: i128,

    // ---- Insurance fund ----
    /// Balance held by the book as the insurance fund. Funded by
    /// liquidation surpluses and debited on liquidation shortfalls and ADL.
    insurance_fund_balance: Balance,

    // ---- Persistent linked list (source of truth in QMDB) ----
    /// Head of the vault linked list. `None` when no non-empty vaults.
    head_owner: Option<Address>,
    /// Tail of the vault linked list. `None` when no non-empty vaults.
    tail_owner: Option<Address>,

    // ---- Populated in-memory view ----
    /// Built by walking the linked list from `head_owner` and following
    /// `next_owner`. Not persisted as a standalone structure.
    /// key: perp/<address>/vault/<owner> -> value: {VaultEntry};
    vaults: Map<Address, VaultEntry>,
}

/// Clearing vault entry:
/// key: perp/<address>/vault/<owner> -> value: {VaultEntry};
///
/// The `free_margin + locked_isolated_margin + pending_withdrawal` sum is
/// the trader's claim on the book's pooled settlement-token balance. It
/// must always equal the actual per-owner balance attributable in the
/// book's settlement-token ledger.
#[repr(C)]
pub struct VaultEntry {
    /// The trader whose margin this entry tracks. Also the key under which
    /// the entry is stored.
    owner: Address,

    /// Previous non-empty vault owner in the linked list. `None` when head.
    prev_owner: Option<Address>,
    /// Next non-empty vault owner in the linked list. `None` when tail.
    next_owner: Option<Address>,

    // ---- Balances ----
    /// Free, unlocked margin. Available for new isolated positions, for
    /// cross positions to draw on, or for withdrawal.
    free_margin: Balance,
    /// Margin currently locked as isolated margin on positions. Must equal
    /// Σ `isolated_margin` of this owner's open isolated positions.
    locked_isolated_margin: Balance,
    /// Withdrawals submitted but not yet finalized.
    pending_withdrawal: Balance,

    // ---- Lifetime flow counters (per owner) ----
    /// Cumulative settlement tokens deposited by this trader.
    total_deposited: Balance,
    /// Cumulative settlement tokens withdrawn by this trader.
    total_withdrawn: Balance,
    /// Net realized PnL credited to this vault over its lifetime.
    realized_pnl: i128,
    /// Net funding paid/received by this trader over its lifetime.
    net_funding: i128,

    // ---- Metadata ----
    /// Last event applied to this vault.
    last_event: VaultEvent,
    /// Block height of first deposit.
    opened_at: u64,
    /// Block height of the last update.
    updated_at: u64,
}


// ---------------------------------------------------------------------------
// Embedded position book
// ---------------------------------------------------------------------------

/// Aggregate + entry view of the perp book's position state.
///
/// Persistent state lives in the doubly-linked list (`head_owner` /
/// `tail_owner` plus `prev_owner` / `next_owner` on each position). The
/// `positions` map is populated from that list at load time so settlement,
/// funding, and risk can do O(1) random access by owner without walking
/// the list on every lookup.
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
    /// Cumulative open interest in contracts.
    open_interest: Qty,
    /// Sum of all positive unrealized PnL across positions.
    total_unrealized_pnl: i128,
    /// Sum of all isolated margin locked by positions in this book.
    total_isolated_margin: Balance,
    /// Number of positions currently under liquidation.
    liquidation_count: u32,
    /// Number of positions that have been ADL'd.
    adl_count: u32,

    // ---- Persistent linked list (source of truth in QMDB) ----
    /// Head of the position linked list. `None` when no open positions.
    head_owner: Option<Address>,
    /// Tail of the position linked list. `None` when no open positions.
    tail_owner: Option<Address>,

    // ---- Populated in-memory view ----
    /// The populated position entries, built by walking the linked list
    /// from `head_owner` and following `next_owner`. Not persisted as a
    /// standalone structure; it mirrors the linked list for O(1) lookup.
    /// key: perp/<address>/position/<owner> -> value: {PerpPosition};
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
    /// The position owner. Also the key under which this entry is stored.
    owner: Address,
    /// Previous open position owner in the linked list. `None` when head.
    prev_owner: Option<Address>,
    /// Next open position owner in the linked list. `None` when tail.
    next_owner: Option<Address>,

    /// The position side.
    side: PositionSide,
    /// The net quantity of the aggregated positions.
    size: Qty,
    /// The entry price of the net positions, weighted average cost of the opening contracts.
    entry_price: Price,
    /// The margin mode of the position.
    margin_mode: MarginMode,
    /// The isolated margin when with isolated mode.
    isolated_margin: Balance,
    /// The value of cumulative_funding_index at the last time this position was touched.
    funding_index: i128,
    /// The block height that last funding happens on this position.
    last_funding_time: u64,
    /// The mark price at which a position's equity falls to exactly its maintenance margin requirement.
    liquidation_price: Price,
    /// The realized PnL of the trader.
    realized_pnl: i128,
    /// The un-realized PnL of the trader.
    unrealized_pnl: i128,
    /// The leverage rate derived per position.
    leverage: u32,
    /// If the trader is liquidated by the exchange.
    is_liquidated: bool,
    /// The last position event type.
    last_event: PositionEvent,
    /// The block height on which the 1st position happens.
    opened_at: u64,
    /// The block height on which the last update happens.
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
```