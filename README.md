# dex-net-rs
A modern blockchain network in Rust with built-in DEX primitives as the 1st class citizen.

## P2P Network Layer
An ultra fast quic protocol based P2P network layer with support for NAT traversal, peer discovery, and message routing.
 Inspired by [iroh] (https://github.com/n0-computer/awesome-iroh).

## Execution layer

### Structured Gossiping
The structured and latency aware gossiping is a novel gossiping protocol that is designed to improve the efficiency and
scalability of the network. It is based on a structured overlay network, which allows for efficient routing of messages
and reduces the number of redundant messages that are sent across the network. The structured gossiping protocol is
designed to be resilient to network failures and can adapt to changes in the network topology.

### Transaction Propagation
As TXNs are packed by the proposers elected by the pseudo random selection, they are routed only to the proposers of the
coming consensus instances. Without a native gossiping, the TXNs can be mined very quickly.

### Block propagation
To extensively leverage the capacity of a distributed network, the blocks are erasure coded and propagated in a
gossiping manner. The erasure coding is based on [Reed-Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction) algorithm,
which allows the block to be reconstructed from any subset of the erasure coded pieces. It will significantly reduce
the bandwidth consumption and improve the block propagation speed when a block need to be fanout from a proposer. Thus,
the latency of the block propagation can be reduced to a minimum, which is critical for the performance of the consensus
 protocol.

### Freezer and Chain DB
A linear Freezer database stores the ancient blocks, and the linear chain database stores the recent blocks. The Freezer
 database is designed to be append-only, which allows for efficient storage and retrieval of historical data. The Chain
  database, on the other hand, is optimized for fast access to the most recent blocks, enabling quick validation and
  processing of new transactions.

### [Optional] Virtual Machine
The execution environment of smart contract is under discussion, as there are a lots of attack vectors could be exposed
by a flaky smart contract. With or without the supporting of smart contract is now under discussion.

### State Engine
A deterministic state engine leverages [QMDB](https://arxiv.org/pdf/2501.05262)
inspired by LayerZero Labs' paper: `QMDB: Quick Merkle Database` to replace the legacy Merkle Patricia Trie (MPT) with
a more efficient and scalable data structure for global state management.

### Asset Primitives
Native digital asset primitives are implemented in the execution layer, which allows for the creation and management of
digital assets on the blockchain. These primitives include support for regulated stable coins, fungible tokens,
non-fungible tokens (NFTs), and other digital assets, enabling a wide range of use cases such as decentralized finance
(DeFi), gaming, and digital collectibles.

### Exchange Primitives
- CLOB
The execution layer also includes native exchange primitives, which enable the creation and management of decentralized
exchanges (DEXs) on the blockchain. These primitives include support for marketplace which involves margin account
management, order book management, trade execution and state pub&sub. The exchange primitives are designed to be highly
 efficient.
- Ask for Quote
The network also supports an Ask for Quote (AFQ) mechanism, which allows users to request quotes for specific assets or
trading pairs. This mechanism enables users to obtain real-time pricing information and execute trades based on
the quotes received. The AFQ mechanism is designed to be fast and efficient, allowing for quick execution of trades and
 improved liquidity in the market.
- Order Types
Primitives of different types of orders are supported, such as Standard, Iceberg, PostOnly, TrailingStop, Pegged, MarketToLimit, ReserveOrder.

## Consensus Layer
Modern BFT consensus protocol contains a few innovations: optimal view forwarding, pipelined, round-reduced and
parallelized voting.

### Minimmit BFT
Minimmit: Fast Finality with Even Faster Blocks inspired by [Minimmit](https://arxiv.org/pdf/2508.10862) paper. It is
a BFT consensus protocol with 80% quorum model that achieves fast finality with even faster blocks. The protocol is
designed to be super optimal for consensus view forwarding and very friendly for pipeline parallelized execution.
Allowing for quick confirmation and high throughput of transactions and blocks.

### Consensus Messaging
- Message Relaying
The consensus messaging channel uses the structured and latency aware gossiping is a novel gossiping protocol that
is designed to improve the efficiency and scalability of the network. It is based on a structured overlay network,
which allows for efficient routing of messages and reduces the number of redundant messages that are sent across
the network. The structured gossiping protocol is designed to be resilient to network failures and can adapt to
changes in the network topology.
- Message Aggregation
As most of the voting messages shares the same content in a consensus instance, between the messaging layer and the
consensus layer, an batch aggregation layer is introduced to reduce the consumed capacity of message relaying and the
processing computing power of the consensus engine. The aggregation is inspired by [BLS12-381](https://zodl.com/new-snark-curve/)
and [Fast SubGroup Checks for BLS12-381](https://eprint.iacr.org/2019/814.pdf).