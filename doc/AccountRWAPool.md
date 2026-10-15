# RWA Pool Account

## Use Case
RWA pool account stores the meta data of a digital asset pool, it is similar as an ERC-20 token contract account
in ETH, but it does not track the proportions of each user of that asset, as the proportions are distributed
into each external account for parallel processing.

## Schema and Key of QMDB
```Rust
/// The RWA Pool Account and the state to be managed by QMDB:
/// key: pool/<PoolID>/issuer -> value: {Address};
/// key: pool/<PoolID>/regulator -> value: {Address};
/// key: pool/<PoolID>/symbol -> value: {Symbol};
/// key: pool/<PoolID>/total_supply -> value: {Balance};
/// key: pool/<PoolID>/reserve -> value: {Balance};
/// key: pool/<PoolID>/blocked -> value: {bool};
trait Address {};
trait Hash {};
type Symbol = [u8; 32];

#[repr(C)]
pub struct RWAPoolAccount{
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