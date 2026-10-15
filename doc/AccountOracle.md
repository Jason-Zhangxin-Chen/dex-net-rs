# Data Oracle Account

## Use Case
Data oracle account contains the data point index of the external world. The data point comes from different data sources,
such as CEX, stock exchanges, the regional government federations.


## Schema and Key of QMDB
```Rust
type DataSource = [u8; 128];

/// The Oracle Account and the state to be managed by QMDB:
/// key: oracle/<Address>/issuer -> value: {Address};
/// key: oracle/<Address>/regulator -> value: {Address};
/// key: oracle/<Address>/index -> value: {u64};
/// key: oracle/<Address>/last_updated -> value: {u64};
/// key: oracle/<Address>/symbol -> value: {symbol};
/// key: oracle/<Address>/bonded -> value: {balance};
/// key: oracle/<Address>/reward -> value: {balance};
/// key: oracle/<Address>/is_paused -> value: {bool};
/// key: oracle/<Address>/data_source -> value: {DataSource};
#[repr(C)]
pub OracleAccount {
    /// The data issuer account address.
    issuer: Address,
    /// The regulator account address.
    regulator: Address,
    /// The data index, coupled with the system's unified precision.
    index: u64,
    /// The last updated block height.
    last_updated: u64,
    /// The symbol of the data index.
    symbol: Symbol,
    /// The bonded native token balance, it will be unlocked and reclaimable for the issuer with an un-bonding period.
    bonded: Balance,
    /// The accumulated reward in native token. It comes from those TXNs which use this data.
    /// It is reclaimable for the issuer account after an un-bonding period.
    reward: Balance,
    /// The paused flag which indicates the issuer is not going to update the point anymore.
    /// Set at the un-bond request.
    is_paused: bool,
    /// The data source of the data point.
    data_source: DataSource,
}
```