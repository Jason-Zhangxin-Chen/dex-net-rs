# External Account

## Use Case
External account represents the user who owns the private key of the corresponding account address.
It can be used by a human, a trade agent to issue the transfer and trade intent. It can be used by
 a licensed party to mint or burn assets from the RWA Pool Account that derived from it as well. It
 can also be used by a regulation party to manage the RWA Pool or the external account who owns the
 RWA by according to the regulation policy.

## Schema and Key of QMDB
 ```Rust
 type Balance = u64;
 type Nonce = u64;
 trait TokenID {};

 /// The external account and the state to be managed by QMDB:
 /// key: "u/<alice>/nonce" -> value: {Nonce}
 /// key: "u/<alice>/native_balance" -> value: {Balance}
 /// key: "u/<alice>/blocked" -> value: {bool}
 #[repr(C)]
 pub struct ExternalAccount {
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