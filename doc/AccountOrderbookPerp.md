# Perpetual Orderbook Account

## Use Case

Perpetual orderbook account holds the state of one perpetual future market. It has two
coupled sub-structures:

- The **order book** holds unmatched orders (bids and asks) plus the aggregate
  top-of-book state.
- The **position book** holds crossed orders that have formed contracts between traders.
  Positions persist across matches until closed, liquidated, ADL'd, or settled.

Unlike spot, a fill in the perp book does not move tokens between accounts. It creates or
updates a `Position` for each side, and PnL against the settlement token is realized only
at close, liquidation, ADL, or funding.

Margin, however, is **not** scoped to the book. All perp books that settle in the same
token share one global **Clearinghouse**: a trader deposits the settlement token once,
into their clearinghouse `Account`, and that balance backs every position they hold in
every market (cross margin) or locks isolated margin per position. The book keeps only
the market-local aggregates it needs (open interest, the market's position list); the
authoritative per-trader state lives under the clearinghouse.

This is the HyperLiquid-style portfolio-margin model. Hedge mode and multiple strategies
are expressed as subaccounts, each of which is simply another `Account` in the
clearinghouse.

## Schema and Key of QMDB

The book is identified by its own account `Address`. Each order is identified by the hash
of the transaction that carried it, so no order id allocator or sequence counter is
needed. Positions and margin state live under the clearinghouse, keyed by owner and
market, so the match engine resolves either side of a match with an O(1) key-value read
and cross-margin risk can enumerate an account's positions across all markets.

```Rust
//! QMDB keys:
//! perp/<address>/meta -> {PerpBookAccount}
//! perp/<address>/bid/<price> -> {PriceLevel}
//! perp/<address>/ask/<price> -> {PriceLevel}
//! perp/<address>/order/<tx_hash> -> {Order}
//! clearinghouse/<ch>/meta -> {Clearinghouse}
//! clearinghouse/<ch>/account/<owner> -> {Account}
//! clearinghouse/<ch>/account/<owner>/position/<market> -> {Position}

/// The price of the order.
type Price = u64;
/// The quantity of the order (contracts).
type Qty = u64;
/// The notional amount of the order.
type Notional = u128;
/// The token used as collateral. The settlement token of the clearinghouse.
type Balance = u128;

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

/// Direction of a locked perp contract. With the signed `Position::szi` this
/// is derivable (positive = long, negative = short); the enum is kept for
/// API clarity.
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

/// Last event applied to a clearinghouse account.
#[repr(u8)]
pub enum VaultEvent {
    /// Margin deposited into the clearinghouse.
    Deposited = 0,
    /// Margin withdrawn from the clearinghouse.
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
    /// The token used to settle PnL and margin. Determines which
    /// clearinghouse the book registers with.
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
    /// engine, not by matches: the median of the index+basis and index+MA
    /// oracle prices.
    mark_price: Price,
}

// ---------------------------------------------------------------------------
// Perp book account
// ---------------------------------------------------------------------------

/// The Perp book account. The account address itself is the book id.
/// key: perp/<address>/meta -> value: {PerpBookAccount};
/// It could be refined by breaking it into individual fields to avoid write
/// amplification on the QMDB side.
///
/// The book owns no per-trader state: positions and balances live in the
/// global clearinghouse. The embedded `position_aggregate` keeps only the
/// market-level counters and the market-level position list used by funding
/// and liquidation scans.
#[repr(C)]
pub struct PerpBookAccount {
    /// Symbol of the book.
    symbol: Symbol,
    /// Perp market metadata.
    market: PerpMarket,

    /// Book config, once loaded would rarely be updated. Store it as an
    /// object in QMDB.
    config: BookConfig,

    /// The clearinghouse that holds this book's positions and margin. One
    /// clearinghouse exists per settlement token; the book registers with
    /// the clearinghouse of its `market.settlement_token`.
    clearinghouse: Address,

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

    // ---- Market-level position aggregate ----
    //
    // Counters plus the market-level position linked list. The position
    // entries themselves live in the clearinghouse, keyed by
    // (owner, market); this struct holds no per-owner map. Reading the
    // counters gives the OI view without touching position entries.
    position_aggregate: PositionAggregate,
}

// ---------------------------------------------------------------------------
// Market-level position aggregate
// ---------------------------------------------------------------------------

/// Market-level aggregate of the positions opened against this book.
///
/// This is the counter/list half of the old embedded position book. There
/// is no map of positions here: entries are stored under the clearinghouse
/// as `clearinghouse/<ch>/account/<owner>/position/<market>` and enumerated
/// by walking `head_position_owner` -> `mkt_next_owner`. Funding and
/// liquidation scans use the list; the match engine reads positions by the
/// (owner, market) key directly.
#[repr(C)]
pub struct PositionAggregate {
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
    /// Cumulative open interest in contracts, Σ |szi| across positions.
    open_interest: Qty,
    /// Sum of all positive unrealized PnL across positions. Maintained
    /// during the funding/risk sweep, not on every mark update.
    total_unrealized_pnl: i128,
    /// Sum of all isolated margin locked by positions in this book.
    total_isolated_margin: Balance,
    /// Number of positions currently under liquidation.
    liquidation_count: u32,
    /// Number of positions that have been ADL'd.
    adl_count: u32,

    // ---- Persistent linked list (source of truth in QMDB) ----
    /// Head of the market's position list. `None` when no open positions.
    head_position_owner: Option<Address>,
    /// Tail of the market's position list. `None` when no open positions.
    tail_position_owner: Option<Address>,
}

// ---------------------------------------------------------------------------
// Global clearinghouse
// ---------------------------------------------------------------------------

/// The global clearinghouse account for one settlement token. Every perp
/// book that settles in that token points at this account, and each
/// trader's margin for those books is a single `Account` here.
/// key: clearinghouse/<ch>/meta -> value: {Clearinghouse};
///
/// Persistent state lives in the global account linked list
/// (`head_account` / `tail_account` plus `prev_account` / `next_account` on
/// each entry). A map of all traders does not fit in memory at scale, so
/// `accounts_cache` is an explicitly bounded hot cache, not the source of
/// truth: misses are QMDB direct reads by key, and the list provides
/// persistent enumeration for liquidation sweeps without a global in-memory
/// map.
#[repr(C)]
pub struct Clearinghouse {
    /// Number of non-empty accounts.
    account_count: u32,

    // ---- Aggregate balances ----
    /// Sum of `free_margin` across all accounts.
    total_free_margin: Balance,
    /// Sum of `locked_isolated_margin` across all accounts. Should equal
    /// the sum of `isolated_margin` across all open isolated positions.
    total_locked_margin: Balance,
    /// Sum of `pending_withdrawal` across all accounts.
    total_pending_withdrawal: Balance,

    // ---- Lifetime flow counters ----
    /// Cumulative settlement tokens deposited over the clearinghouse's
    /// lifetime.
    total_deposited: Balance,
    /// Cumulative settlement tokens withdrawn over the clearinghouse's
    /// lifetime.
    total_withdrawn: Balance,
    /// Net realized PnL credited to accounts over the clearinghouse's
    /// lifetime.
    net_realized_pnl: i128,
    /// Net funding paid/received across accounts over the clearinghouse's
    /// lifetime.
    net_funding: i128,

    // ---- Insurance fund ----
    /// Balance held by the clearinghouse as the insurance fund. Funded by
    /// liquidation surpluses and debited on liquidation shortfalls and ADL.
    ///
    /// One global pool per settlement token, deliberately: the risk unit in
    /// this design is the account, not the position. A cross liquidation
    /// unwinds positions across markets, and its shortfall is one
    /// account-level number that cannot be attributed to a single book
    /// without an arbitrary rule, so the pool scopes to the unit it
    /// backstops. Pooling also diversifies rare tail losses — per-market
    /// pools are thinner and exhaust more often, pushing losses to ADL on
    /// that market's traders. Per-market *attribution* (fees contributed vs
    /// gaps consumed per book, fund/OI per book, drawdown caps) can be
    /// layered on top without splitting the pool.
    insurance_fund_balance: Balance,

    // ---- Persistent linked list (source of truth in QMDB) ----
    /// Head of the global account linked list. `None` when no non-empty
    /// accounts.
    head_account: Option<Address>,
    /// Tail of the global account linked list. `None` when no non-empty
    /// accounts.
    tail_account: Option<Address>,

    // ---- Bounded in-memory hot cache ----
    /// LRU cache of recently touched `Account`s, write-through: QMDB is the
    /// source of truth, a miss is a direct read, and eviction cannot lose
    /// dirty state. Not persisted and never enumerated as a full set.
    accounts_cache: Map<Address, Account>,
}

/// Clearinghouse account entry:
/// key: clearinghouse/<ch>/account/<owner> -> value: {Account};
///
/// One entry per trader (or per subaccount — hedge mode is a second
/// `Account`) across all books that settle in this clearinghouse's token.
///
/// Balances are held once, not per market: `free_margin` backs cross
/// positions and seeds new isolated positions; `locked_isolated_margin` is
/// the sum of margin locked by this account's isolated positions; and the
/// `free_margin + locked_isolated_margin + pending_withdrawal` sum is the
/// trader's claim on the clearinghouse's pooled settlement-token balance.
#[repr(C)]
pub struct Account {
    /// The trader whose margin this account tracks. Also the key under
    /// which the account is stored.
    owner: Address,

    /// Previous non-empty account in the global linked list. `None` when
    /// head.
    prev_account: Option<Address>,
    /// Next non-empty account in the global linked list. `None` when tail.
    next_account: Option<Address>,

    /// Head of this account's position list across markets. `None` when the
    /// account has no open positions.
    head_position_market: Option<Address>,
    /// Tail of this account's position list. `None` when no open positions.
    tail_position_market: Option<Address>,

    // ---- Balances ----
    /// Free, unlocked margin. Available for new isolated positions, for
    /// cross positions to draw on, or for withdrawal.
    free_margin: Balance,
    /// Margin currently locked as isolated margin on positions. Must equal
    /// Σ `isolated_margin` of this account's open isolated positions.
    locked_isolated_margin: Balance,
    /// Withdrawals submitted but not yet finalized.
    pending_withdrawal: Balance,

    // ---- Margin summaries ----
    /// Margin summary over all of the account's positions, isolated
    /// included. `account_value` is the account's total equity.
    ///
    /// Both summaries are caches: refreshed when the account is touched
    /// (fill, funding, deposit, withdrawal) and recomputed from the
    /// position list by liquidation sweeps, which never trust the stored
    /// value alone — mark moves in markets the account has not traded in
    /// leave the stored summaries stale.
    margin_summary: MarginSummary,
    /// Margin summary over the account's cross positions only. Used by the
    /// cross-margin check.
    cross_margin_summary: MarginSummary,
    /// Maintenance margin used by cross positions.
    cross_maintenance_margin_used: Balance,
    /// Initial margin used by cross positions. Equals
    /// `cross_margin_summary.total_margin_used`.
    cross_initial_margin_used: Balance,
    /// Withdrawable collateral: `free_margin - cross_initial_margin_used`,
    /// floored at zero. Isolated margin is locked separately and does not
    /// reduce this.
    withdrawable: Balance,

    // ---- Lifetime flow counters (per owner) ----
    /// Cumulative settlement tokens deposited by this trader.
    total_deposited: Balance,
    /// Cumulative settlement tokens withdrawn by this trader.
    total_withdrawn: Balance,
    /// Net realized PnL credited to this account over its lifetime.
    realized_pnl: i128,
    /// Net funding paid/received by this trader over its lifetime.
    net_funding: i128,

    // ---- Metadata ----
    /// Last event applied to this account.
    last_event: VaultEvent,
    /// Block height of first deposit.
    opened_at: u64,
    /// Block height of the last update.
    updated_at: u64,
}

/// HyperLiquid-style margin summary, reused for the account-level totals
/// and the cross-only view. All values are denominated in the
/// clearinghouse's settlement token base units.
#[repr(C)]
pub struct MarginSummary {
    /// Account equity: balances plus unrealized PnL over the summarized
    /// positions. Can be negative.
    account_value: i128,
    /// Initial margin used by the summarized positions.
    total_margin_used: Balance,
    /// Signed net notional of the summarized positions, Σ szi × mark.
    total_ntl_pos: i128,
    /// Gross notional of the summarized positions, Σ |szi| × mark.
    total_raw_usd: Notional,
}

// ---------------------------------------------------------------------------
// Position (owner × market)
// ---------------------------------------------------------------------------

/// Perp position entry:
/// key: clearinghouse/<ch>/account/<owner>/position/<market> -> value: {Position};
///
/// One net position per (owner, market). An account cannot be both long and
/// short in the same market; opposing fills reduce or flip the position,
/// and the entry is removed when `szi` reaches zero.
///
/// The entry carries two independent doubly-linked lists:
/// - `acct_prev_market` / `acct_next_market` link this owner's positions
///   across markets, for the account-wide cross-margin check.
/// - `mkt_prev_owner` / `mkt_next_owner` link all owners with a position in
///   this market, for funding and liquidation scans.
///
/// Closing a position must unlink it from both lists and delete the entry
/// in one atomic QMDB batch.
#[repr(C)]
pub struct Position {
    /// The position owner. Part of the key.
    owner: Address,
    /// The market this position is in. Part of the key; also the unit of
    /// the `acct_prev_market` / `acct_next_market` links.
    market: Address,

    /// Previous position of the same owner, by market. `None` when this is
    /// the account's head position.
    acct_prev_market: Option<Address>,
    /// Next position of the same owner, by market. `None` when tail.
    acct_next_market: Option<Address>,

    /// Previous owner with a position in this market. `None` when this is
    /// the market's head position.
    mkt_prev_owner: Option<Address>,
    /// Next owner with a position in this market. `None` when tail.
    mkt_next_owner: Option<Address>,

    /// The net position size, signed (HyperLiquid convention): positive is
    /// long, negative is short, zero means closed. One signed field makes
    /// netting a plain add and a flip a plain sign change; `PositionSide`
    /// is derivable from the sign.
    szi: i128,
    /// The entry price of the net position. Updated as a size-weighted
    /// average while `szi` grows in magnitude with the same sign, left
    /// unchanged while it shrinks, and reset to the fill price on a flip.
    entry_price: Price,
    /// The margin mode of the position. Per position: one account can hold
    /// Cross in one market and Isolated in another simultaneously. The mode
    /// can only change when `szi` is zero.
    margin_mode: MarginMode,
    /// The isolated margin when in isolated mode, zero for cross positions.
    /// Adding to or releasing from this field is a
    /// `VaultEvent::MarginLocked` / `MarginReleased` on the account.
    isolated_margin: Balance,
    /// The value of cumulative_funding_index at the last time this position was touched.
    funding_index: i128,
    /// The block height that last funding happens on this position.
    last_funding_time: u64,
    /// The price at which an isolated position's equity falls to exactly
    /// its maintenance margin requirement. `None` for cross positions,
    /// whose liquidation boundary depends on the whole account rather than
    /// a single price.
    liquidation_price: Option<Price>,
    /// The realized PnL of the trader.
    realized_pnl: i128,
    /// The un-realized PnL of the trader.
    unrealized_pnl: i128,
    /// The leverage rate derived per position. For isolated positions it is
    /// szi × mark / isolated_margin; for cross positions it reflects the
    /// account's cross initial margin usage, not a per-position lock.
    leverage: u32,
    /// True while the liquidation engine holds the position before it is
    /// closed and removed.
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

## Engine Notes

- **Fill** — the match engine resolves the counterparties' `Account`s and `Position`s by
  key from the clearinghouse (hot cache first, QMDB direct read on miss). It updates
  `szi` (reduce/flip per the entry-price rules), the account balances and summaries, and
  the market's `PositionAggregate` counters. There is no per-market copy of position
  state.
- **Cross-margin check** — iterates the account's position list (`head_position_market`
  → `acct_next_market`), recomputing margin requirements from marks. The stored
  `cross_margin_summary` is refreshed here; the list walk is the authoritative check.
  This was impossible before, when the vault was per-market and could see only one
  market's positions.
- **Isolated liquidation** — uses only the position's own `isolated_margin`; the rest of
  the account is untouched. Cross liquidation evaluates the whole account and closes
  positions until maintenance margin is met.
- **Funding** — iterates the market's position list (`PositionAggregate`
  head → `mkt_next_owner`), settling per-position funding against account balances with
  `funding_index` attribution.
- **List maintenance** — opening a position links it into both lists; closing unlinks it
  from both and deletes the entry in one atomic QMDB batch. An account with no balance,
  no pending withdrawal, and no positions is unlinked from the global account list.
- **Cache policy** — `accounts_cache` and the matching engine's position hot cache are
  bounded LRU caches, write-through, never the source of truth. Eviction cannot lose
  dirty state, and enumeration (funding, liquidation sweeps) never relies on the cache.
- **Subaccounts** — hedge mode or multiple strategies open a second `Account`; nothing
  else needs special casing.
- **Insurance fund scope** — one global pool per settlement token, matching the risk
  unit: cross-liquidation shortfalls are account-level events spanning markets and
  cannot be cleanly attributed to one book, and a shared pool diversifies rare tail
  losses. Per-market attribution counters (contributions vs consumption per book,
  fund/OI per book, drawdown caps) can be layered on later without splitting the pool.

## Perp State Machine
```Text
                            fill (increase / partial reduce / flip)
                         ┌──────────────────────────────────────────┐
                         │                                          │
                         ▼                                          │
   ┌──────┐  fill opens  ┌──────┐  fill size→0   ┌────────┐  delete  ┌──────┐
   │ Flat │ ───────────▶ │ Open │ ─────────────▶ │ Closed │ ───────▶ │ Flat │
   └──────┘              └──────┘                └────────┘          └──────┘
      ▲                     │
      │                     │ equity < MM
      │                     ▼
      │              ┌─────────────┐  liq fill size→0  ┌────────────┐  delete
      │              │ Liquidating │ ────────────────▶ │ Liquidated │ ───────▶ Flat
      │              └─────────────┘                   └────────────┘
      │                     │
      │                     │ insurance fund insufficient
      │                     ▼
      │              ┌─────────┐  delete
      │              │   ADL   │ ───────▶ Flat
      │              └─────────┘
      │
      │                     │ market delisted
      │                     ▼
      │              ┌─────────┐  delete
      └───────────── │ Settled │ ───────▶ Flat
                     └─────────┘

```