---
sip: 20
title: "[SIP20] JSONRPC"
author: "@lerencao"
type: API
category: API
status: Alpha
created: 2020-11-17
weight: 20
---

# 统一 JSONRPC 接口的参数和返回值类型

目前 Starcoin 的 json  rpc 接口返回值类型直接使用了内部的数据结构，比如 Block, Header, Transaction, TransactionInfo 等等。
但是在接口的实际使用过程中，这些内部结构的字段无法满足外部使用需求，往往是缺乏必要的关联字段，比如外部使用可能需要 TransactionInfo 里包含 txn 被打包的 block id，亦或是某个 u64 类型的字段序列化成 json 后，无法被 javascript 端的正确的解析出来。
为了解决诸如此类的问题，该 RFC 定义了一系列 jsonrpc 接口返回值类型，试图将内部使用的数据结构和对外接口的数据结构解耦。


## 类型定义

经过梳理，接口中使用的以下几类核心数据结构需要替换成自定义的对外返回的类型。

- Block, 以及关联的 BlockHeader, BlockBody。
- Transaction，以及关联的 RawUserTransaction，BlockMetadata 等。
- TransactionInfo：Transaction 在链上的结果数据。
- Transaction 在链上产生的 Events 数据。

以下部分列出了新的接口返回类型定义。

### Block

```rust
pub struct BlockView {
    pub header: BlockHeaderView,
    pub body: BlockTransactionsView,
    pub uncles: Vec<BlockHeaderView>,
}

pub struct BlockHeaderView {
    /// block hash
    pub block_hash: HashValue,
    /// Parent hash.
    pub parent_hash: HashValue,
    /// Block timestamp.
    pub timestamp: u64,
    /// Block number.
    pub number: BlockNumber,
    /// Block author.
    pub author: AccountAddress,
    /// Block author auth key.
    pub author_auth_key: Option<AuthenticationKey>,
    /// The transaction accumulator root hash after executing this block.
    pub accumulator_root: HashValue,
    /// The parent block accumulator root hash.
    pub parent_block_accumulator_root: HashValue,
    /// The last transaction state_root of this block after execute.
    pub state_root: HashValue,
    /// Gas used for contracts execution.
    pub gas_used: u64,
    /// Block difficulty
    pub difficulty: U256,
    /// Consensus nonce field.
    pub nonce: u32,
    /// hash for block body
    pub body_hash: HashValue,
    /// The chain id
    pub chain_id: u8,
}
pub enum BlockTransactionsView {
    Hashes(Vec<HashValue>),
    Full(Vec<SignedUserTransactionView>),
}
```

需要注意的是，BlockBodyView 包含了两种，一种是只包含 block 中所有 transaction 的 hash 值，另外一种是完整的 txn 数据。这样客户端在不需要具体 txn 数据时，可以选择性调用。

### Transaction

``` rust
pub struct TransactionView {
    block_hash: HashValue,
    block_number: BlockNumber,
    transaction_hash: HashValue,
    transaction_index: u64,
    block_metadata: Option<BlockMetadataView>,
    user_transaction: Option<SignedUserTransactionView>,
}
pub struct BlockMetadataView {
    /// Parent block hash.
    pub parent_hash: HashValue,
    pub timestamp: u64,
    pub author: AccountAddress,
    pub author_auth_key: Option<AuthenticationKey>,
    pub uncles: u64,
    pub number: BlockNumber,
    pub chain_id: u8,
    pub parent_gas_used: u64,
}
pub struct SignedUserTransactionView {
    /// The raw transaction
    pub raw_txn: RawUserTransactionView,

    /// Public key and signature to authenticate
    pub authenticator: TransactionAuthenticator,
}
pub struct RawUserTransactionView {
    /// Sender's address.
    pub sender: AccountAddress,
    // Sequence number of this transaction corresponding to sender's account.
    pub sequence_number: u64,

    // The transaction payload in scs bytes.
    #[serde(serialize_with = "serialize_binary")]
    pub payload: Vec<u8>,

    // Maximal total gas specified by wallet to spend for this transaction.
    pub max_gas_amount: u64,
    // Maximal price can be paid per gas.
    pub gas_unit_price: u64,
    // The token code for pay transaction gas, Default is STC token code.
    pub gas_token_code: String,
    // Expiration timestamp for this transaction. timestamp is represented
    // as u64 in seconds from Unix Epoch. If storage is queried and
    // the time returned is greater than or equal to this time and this
    // transaction has not been included, you can be certain that it will
    // never be included.
    // A transaction that doesn't expire is represented by a very large value like
    // u64::max_value().
    pub expiration_timestamp_secs: u64,
    pub chain_id: u8,
}
```

这里需要注意两点：

1. TransactionView 将 BlockMedata 类型和 SignedUserTransaction 平铺在一起，减少嵌套，绝大部分情况下，外部调用应该只关心 `user_transaction` 字段。
2. RawUserTransactionView 中，将 payload 存储为 scs 序列化后的 bytes 数据，外部如果需要使用里面的具体数据，可以通过 scs 反序列化成 scs 数据结构。这样可以减少 payload 复杂的数据结构给 jsonrpc 使用端带来的负担。

### TransactionInfo

``` rust
pub struct TransactionInfoView {
    block_hash: HashValue,
    block_number: BlockNumber,
    /// The hash of this transaction.
    transaction_hash: HashValue,
    transaction_index: u64,
    /// The root hash of Sparse Merkle Tree describing the world state at the end of this
    /// transaction.
    state_root_hash: HashValue,

    /// The root hash of Merkle Accumulator storing all events emitted during this transaction.
    event_root_hash: HashValue,

    /// The amount of gas used.
    gas_used: u64,

    /// The vm status. If it is not `Executed`, this will provide the general error class. Execution
    /// failures and Move abort's receive more detailed information. But other errors are generally
    /// categorized with no status code or other information
    status: TransactionVMStatus,
}

#[derive(Clone, Debug, Hash, Eq, PartialEq, Serialize)]
pub enum TransactionVMStatus {
    Executed,
    OutOfGas,
    MoveAbort {
        location: AbortLocation,
        abort_code: u64,
    },
    ExecutionFailure {
        location: AbortLocation,
        function: u16,
        code_offset: u16,
    },
    MiscellaneousError,
}
```

TransactionInfoView 中在 TransactionInfo 的基础上加入了 txn 所在 block 的相关信息。

### Transaction Events

``` rust
pub struct TransactionEventView {
    pub block_hash: Option<HashValue>,
    pub block_number: Option<BlockNumber>,
    pub transaction_hash: Option<HashValue>,
    // txn index in block
    pub transaction_index: Option<u64>,

    pub data: Vec<u8>,
    pub type_tags: TypeTag,
    pub event_key: EventKey,
    pub event_seq_number: u64,
}
```

## 缺点

自定义的 rpc 类型定义大部分和已有的内部结构，在数据字段上是重叠的，显的繁琐和重复。
