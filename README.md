[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE-MIT)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](./LICENSE-APACHE)
[![CI](https://github.com/Jason-Zhangxin-Chen/dex-net-rs/actions/workflows/ci.yml/badge.svg)](https://github.com/Jason-Zhangxin-Chen/dex-net-rs/actions/workflows/ci.yml)

# dex-net-rs

A modern blockchain network in Rust with built-in DEX primitives as first-class citizens.

> **Status: early development.** The workspace currently holds the foundation crates only;
> the layers described below are the design target.

## P2P Network Layer

An ultra-fast, QUIC-based P2P network layer with support for NAT traversal, peer discovery,
and message routing, inspired by [iroh](https://github.com/n0-computer/iroh).

## Execution Layer

### Structured Gossiping

Structured, latency-aware gossiping is a novel gossip protocol designed to improve the
efficiency and scalability of the network. It is built on a structured overlay network that
routes messages efficiently and reduces the number of redundant messages sent across the
network. The protocol is resilient to network failures and adapts to changes in network
topology.

### Transaction Propagation

Transactions are packed by proposers elected through pseudo-random selection and routed only
to the proposers of upcoming consensus instances. By bypassing native gossiping, transactions
can be included in blocks very quickly.

### Block Propagation

To fully leverage the capacity of a distributed network, blocks are erasure-coded and
propagated through gossiping. The erasure coding is based on the
[Reed–Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction) algorithm,
which allows a block to be reconstructed from any subset of its erasure-coded pieces. This
significantly reduces bandwidth consumption and speeds up block propagation when a block is
fanned out from a proposer, keeping block propagation latency to a minimum — critical for
consensus performance.

### Freezer and Chain DB

An append-only Freezer database stores older blocks, and a linear chain database stores the
recent ones. The Freezer is designed for append-only access, enabling efficient storage and
retrieval of historical data. The chain database, on the other hand, is optimized for fast
access to the most recent blocks, enabling quick validation and processing of new
transactions.

### Virtual Machine (optional)

The smart contract execution environment is under discussion, as flawed smart contracts can
expose many attack vectors. Whether to support smart contracts at all is still being debated.

### State Engine

A deterministic state engine replaces the legacy Merkle Patricia Trie (MPT) with a more
efficient and scalable data structure for global state management, inspired by LayerZero
Labs' paper [QMDB: Quick Merkle Database](https://arxiv.org/pdf/2501.05262).

### Asset Primitives

Native digital asset primitives are implemented in the execution layer, allowing the creation
and management of digital assets on the blockchain. These primitives include support for
regulated stablecoins, fungible tokens, non-fungible tokens (NFTs), and other digital assets,
enabling a wide range of use cases such as decentralized finance (DeFi), gaming, and digital
collectibles.

### Regulation Primitives

The regulation primitives are built-in as well, a network configuration enables the regulation primitives from the
genesis of the network, it is used by the licensed parties to control the system risk which is under monitoring and
regulation. The primitives should involve the account Locking/Unlocking, asset Locking/Unlocking,
market Opening/Closing/ etc...

### Exchange Primitives

Native exchange primitives enable the creation and management of decentralized exchanges
(DEXs) on the blockchain.

#### CLOB

These primitives support a marketplace involving margin account management, order book
management, trade execution, and state pub/sub. The exchange primitives are designed to be
highly efficient.

#### Request for Quote

The network also supports a Request for Quote (RFQ) mechanism, which allows users to request
quotes for specific assets or trading pairs. This mechanism enables users to obtain real-time
pricing information and execute trades based on the quotes received. The RFQ mechanism is
designed to be fast and efficient, allowing quick execution of trades and improved liquidity
in the market.

#### Order Types

Order primitives support several types of orders, including Standard, Iceberg, PostOnly,
TrailingStop, Pegged, MarketToLimit, and ReserveOrder.

## Consensus Layer

A modern BFT consensus protocol with several innovations: optimal view forwarding, pipelined
and round-reduced execution, and parallelized voting.

### Minimmit BFT

[Minimmit: Fast Finality with Even Faster Blocks](https://arxiv.org/pdf/2508.10862) is a BFT
consensus protocol with an 80% quorum model that achieves fast finality with even faster
blocks. The protocol is highly optimized for consensus view forwarding and well suited to
pipelined parallel execution, allowing quick confirmation and high throughput for
transactions and blocks.

### Consensus Messaging

#### Message Relaying

The consensus messaging channel runs over the structured, latency-aware gossiping protocol
described above, inheriting its efficient routing, reduced redundancy, and resilience to
network failures and topology changes.

#### Message Aggregation

Because most voting messages in a consensus instance share the same content, a batch
aggregation layer is introduced between the messaging layer and the consensus layer to reduce
the relaying capacity consumed and the processing power required by the consensus engine. The
aggregation is inspired by [BLS12-381](https://zodl.com/new-snark-curve/) and
[Fast Subgroup Checks for BLS12-381](https://eprint.iacr.org/2019/814.pdf).
