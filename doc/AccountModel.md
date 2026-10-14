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
