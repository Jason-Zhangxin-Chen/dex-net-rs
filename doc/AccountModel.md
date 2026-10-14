# Account Models

There three account models each of which provides different finance primitives.

## External Account

External account represents the user who owns the private key of the corresponding account address.
It can be used by a human, a trade agent to issue the transfer and trade intent. It can be used by
 a licensed party to mint or burn assets from the RWA Pool Account that derived from it as well. It
 can also be used by a regulation party to manage the RWA Pool or the external account who owns the
 RWA by according to the regulation policy.

 ```Rust
 type Balance: u64;
 type Nonce: u64;
 trait TokenID {};

 /// The external account and the state to be managed by QMDB:
 /// key: "u/<alice>/nonce" -> value: {Nonce}
 /// key: "u/<alice>/native_balance" -> value: {Balance}
 /// key: "u/<alice>/blocked" -> value: {bool}
 #[repr(C)]
 pub ExternalAccount {
    /// The nonce to prevent replay attack.
    nonce: Nonce,
    /// The native token balance of the account.
    native_balance: Balance,
    /// The account blocked flag.
    blocked: bool,
    /// The populated tokens from the underlying QMDB:
    /// key: "u/<alice>/tokens" -> value: {Set<TokenID>};
    /// key: "u/<alice>/token/<TokenID>" -> value: {Balance};
    tokens: Option<Map<TokenID, Balance>>,
 }
 ```

## RWA Pool Account

RWA pool account stores the meta data of a digital asset pool, it is similar as an ERC-20 token contract account
in ETH, but it does not track the proportions of each user of that asset, as the proportions are distributed
into each external account for parallel processing.
```Rust
/// The RWA Pool Account and the state to be managed by QMDB:
/// key: pool/<PoolID>/issuer -> value: {Address};
/// key: pool/<PoolID>/regulator -> value: {Address};
/// key: pool/<PoolID>/symbol -> value: {Symbol};
/// key: pool/<PoolID>/total_supply -> value: {Balance};
/// key: pool/<PoolID>/reserve -> value: {Balance};
/// key: pool/<PoolID>/blocked -> value: {bool};
trait Address {};
type Symbol: [u8; 32];

#[repr(C)]
pub RWAPoolAccount{
    /// The asset issuer account address which can mint or burn the asset from ths pool.
    issuer: Address,
    /// The asset regulation party account address which can block/unblock the pool and the asset owner account.
    regulator: Address,
    /// The symbol of the asset.
    symbol: Symbol,
    /// The total supply of the asset.
    total_supply: Balance,
    /// The resting reserve in this pool.
    reserve: Balance,
    /// The asset pool blocked flag.
    blocked: bool,
}
```

## CLOB Account

CLOB account stores the orderbook of a specific market with initial configurations.

```text
// Book metadata (one key per book)
ob/<book_id>/meta → {
    symbol: "BTC-USDC",
    base_token: id,
    quote_token: id,
    config: { tick_size, lot_size, maker_fee, taker_fee, ... },
    best_bid_price: u64 | null,
    best_ask_price: u64 | null,
    bid_level_count: u32,
    ask_level_count: u32,
    total_bid_qty: u128,
    total_ask_qty: u128,
}

// Price level (one key per level per side)
ob/<book_id>/bid/<price> → {
    price: u64,
    total_qty: u128,
    order_count: u32,
    head_order_id: u64,   // oldest order (time priority)
    tail_order_id: u64,   // newest order
    next_price: u64 | null,  // next lower price (for bids)
    prev_price: u64 | null,  // next higher price (for bids)
}

ob/<book_id>/ask/<price> → {
    price: u64,
    total_qty: u128,
    order_count: u32,
    head_order_id: u64,
    tail_order_id: u64,
    next_price: u64 | null,  // next higher price (for asks)
    prev_price: u64 | null,  // next lower price (for asks)
}

// Order (one key per order)
ob/<book_id>/order/<order_id> → {
    order_id: u64,
    side: Bid | Ask,
    price: u64,
    quantity: u128,       // remaining quantity
    original_qty: u128,
    owner: Address,
    next_order_id: u64 | null,  // next in time queue at this level
    prev_order_id: u64 | null,  // prev in time queue at this level
    timestamp: u64,
    status: Open | PartiallyFilled | Filled | Cancelled,
}
```