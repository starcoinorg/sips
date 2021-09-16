---
sip: 23
title: "[SIP23] 预言机"
author: "@jolestar"
sip_type: Standard
category: Contract
status: Alpha
created: 2021-08-11
weight: 23
---

## 预言机

预言机（Oracle） 的实现和扩展协议

<!--more-->

## 动机

智能合约中，通常通过预言机来获取外部数据。预言机属于基础设施之一。

## 目标

提供一种通用的，可扩展的预言机标准以及基本的读写方法，让各种预言机的协议以及数据提供方可以共享统一的基础库以及数据格式。

## 类型以及方法定义

```rust
 struct DataRecord<ValueT: copy+store+drop> has copy, store, drop {
        ///The data version
        version: u64,
        ///The record value
        value: ValueT,
        ///Update timestamp millisecond
        updated_at: u64,
}
```

DataRecord 代表一条预言机更新记录，包含了该记录的版本号，更新时间以及数据 ValueT。 ValueT 是一个泛型参数，它可能是一个价格，也可能是个哈希，也可能是一个复杂的结构，这个由预言机开发者来定义。


```rust
struct OracleInfo<OracleT: copy+store+drop, Info: copy+store+drop> has key {
        ///The datasource counter
        counter: u64,
        ///Ext info
        info: Info,
}

public fun register_oracle<OracleT: copy+store+drop, Info: copy+store+drop>(sender: &signer, info: Info);
```

预言机开发者需要先定义一种类型 OracleT，来标记这个预言机，同时可以提供一些扩展信息的说明（Info），并通过 `register_oracle` 方法来注册预言机。


```rust
 struct DataSource<OracleT: copy+store+drop, ValueT: copy+store+drop> has key {
      /// the id of data source of ValueT
      id: u64,
      /// the data version counter.
      counter: u64,
      update_events: 	Event::EventHandle<OracleUpdateEvent<OracleT, ValueT>>,
}

public fun init_data_source<OracleT:  copy+store+drop, Info: copy+store+drop, ValueT: copy+store+drop>(sender: &signer, init_value: ValueT);
```

预言机数据提供方需要通过 `init_data_source` 方法来初始化预言机数据源。

```rust
struct OracleFeed<OracleT: copy+store+drop, ValueT: copy+store+drop> has key {
        record: DataRecord<ValueT>,
 }
```
每个数据提供源的账号下，会有一个 `OracleFeed` 结构来维护 `OracleT` 和 `DataRecord` 的关系。  


```rust
struct OracleUpdateEvent<OracleT: copy+store+drop, ValueT: copy+store+drop> has copy,store,drop {
        source_id: u64,
        record: DataRecord<ValueT>,
}

public fun update<OracleT: copy+store+drop, ValueT: copy+store+drop>(sender: &signer, value: ValueT); 

```

预言机数据提供方可以通过 update 方法来更新自己账号下的 `OracleFeed`, 每次更新都会触发一个 `OracleUpdateEvent` 事件。

```kotlin
public fun read<OracleT:copy+store+drop, ValueT: copy+store+drop>(ds_addr: address): ValueT
```

任何人都可以通过 `read` 方法来读取预言机的值，读取时需要指定数据源的账号地址。

## 扩展方式

### 自定义更新策略

默认情况下，每个数据源提供方可直接更新自己账号下预言机数据，但如果想通过合约更新策略，比如定义一种去中心化的数据更新协议，可以先通过 

```kotlin
public fun remove_update_capability<OracleT:copy+store+drop>(sender: &signer):UpdateCapability<OracleT>;
```
方法将自己账号下的 `UpdateCapability<OracleT>` 取出来，然后重新定义数据结构，将 Capability 锁在新的合约中，然后在该合约中定义更新策略，通过调用

```kotlin
public fun update_with_cap<OracleT: copy+store+drop, ValueT: copy+store+drop>(cap: &mut UpdateCapability<OracleT>, value: ValueT);
```

方法来更新数据。


### 自定义数据类型

比如 Token 的价格数据，通过 u128 来表示， 预言机中的 `ValueT` 就是 u128。同时可以在扩展信息中记录了价格的精度 `scaling_factor`，方便使用方换算。

```rust
struct PriceOracleInfo has copy,store,drop{
  scaling_factor: u128,
}

public fun register_oracle<OracleT: copy+store+drop>(sender: &signer, precision: u8){
  let scaling_factor = Math::pow(10, (precision as u64));
  Oracle::register_oracle<OracleT, PriceOracleInfo>(sender, PriceOracleInfo{
    scaling_factor,
  });
}

public fun init_data_source<OracleT: copy+store+drop>(sender: &signer, init_value: u128){
  Oracle::init_data_source<OracleT, PriceOracleInfo, u128>(sender, init_value);
}
```

### 自定义数据聚合策略

任何人都可以定义一种聚合策略，来从多个数据源提供方筛选和聚合数据。

比如下面的 `latest_price_average_aggregator` 方法，接受多个数据源地址，会将每个数据源地址下的数据读取出来，并通过 `updated_in` 的更新时间限制进行过滤，然后求平均值。

```kotlin
public fun latest_price_average_aggregator<OracleT: copy+store+drop>(addrs: &vector<address>, updated_in: u64): u128 {
  let len = Vector::length(addrs);
  let price_records = PriceOracle::read_records<OracleT>(addrs);
  let prices = Vector::empty();
  let i = 0;
  let expect_updated_after = Timestamp::now_milliseconds() - updated_in;
  while (i < len){
    let record = Vector::pop_back(&mut price_records);
    let (_version, price, updated_at) = Oracle::unpack_record(record);
    if (updated_at >= expect_updated_after) {
      Vector::push_back(&mut prices, price);
    };
    i = i + 1;
  };
  // if all price data not match the update_in filter, abort.
  assert(!Vector::is_empty(&prices), Errors::invalid_state(ERR_NO_PRICE_DATA_AVIABLE));
  Math::avg(&prices)
}
```

## 总结

通过 Move 定义的预言机协议有以下优点：

1. 扩展性强，无论是数据结构，还是数据汇报更新协议，还是聚合协议，都可以在当前协议之上扩展出来。
2. Onchain 和 Offchain 数据结构一致。合约中读取到的数据结构和通过链下的 RPC 接口获取到的 Resource 结构一致，方便解析使用。

尚未解决的点：当前的协议只支持了更新和读取最新版本的数据，历史版本的数据未在合约中保存。如何保存以及聚合历史数据，比较难提供通用的方法，需要根据具体的场景进行设计。当然，如果在实践中发现了比较通用的方法，也可以沉淀到系统合约的预言机库中。