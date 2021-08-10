---
sip: 22
title: "[SIP22] NFT"
author: "@jolestar"
sip_type: Standard
category: Contract
status: Alpha
created: 2021-08-10
weight: 22
---

## NFT

NFT 的实现和扩展协议

<!--more-->

## 动机

在智能合约中，Token 用来表达可拆分的数字资源，而要表达不可拆分的资源，就需要 NFT。在 Move 中，任何一个不可 copy 和 drop 的类型实例，都可以认为是一个不可拆分的资源，是一个 NFT。但 NFT 需要一种统一展示的标准，以及 NFT 的收集和转移方法，所以设计本标准。

## 目标

提供一种通用的，可扩展的标准，同时提供 NFT 相关的基本操作实现。

## 类型定义

```rust

struct NFT<NFTMeta: copy + store + drop, NFTBody: store> has store {
        /// The creator of NFT
        creator: address,
        /// The unique id of NFT under NFTMeta type
        id: u64,
        /// The metadata of NFT
        base_meta: Metadata,
        /// The extension metadata of NFT
        type_meta: NFTMeta,
        /// The body of NFT, NFT is a box for NFTBody
        body: NFTBody,
}
```

NFT 是 Move 中的一种资源，它支持 store ability，但不可 copy 以及 drop，包含一些基本的元信息：

1. creator: NFT 创建者的 address。
2. id: 该 NFT 类型下的唯一 id。
3. base_meta: 基础的通用 metadata 信息，主要用来表达如何展示 NFT。
4. type_meta: 开发者自定义的 metadata，同时用来标记 NFT 的类型。Metadata 不是资源，它表达信息，所以支持 copy + store + drop。
5. body: NFT 包含的资源，可以用来嵌入其他的资源。

如果把 NFT 识为一个箱子，NFT 本身定义了这个箱子的归属，唯一编号，以及展示方式，而 NFTBody 就是箱子中封装的珠宝。展示方式通过 Metadata 来表示。

```rust
struct Metadata has copy, store, drop {
        /// NFT name's utf8 bytes.
        name: vector<u8>,
        /// Image link, such as ipfs://xxxx
        image: vector<u8>,
        /// Image bytes data, image or image_data can not empty for both.
        image_data: vector<u8>,
        /// NFT description utf8 bytes.
        description: vector<u8>,
}
```

Metadata 定义了 NFT 展示所需要的基本信息，名称，图片，描述。如果有其他需要扩展的信息，可以定义在 type_meta 中。图片有两个字段表达，`image` 表示图片地址，`image_data` 可以直接保存图片的二进制信息，客户端展示的时候，使用 `image` 和 `image_data` 中不为空的那个字段。

另外，有的 NFT 的所有实例会使用同一个图片，这种情况下，NFT metadata 中的 `image` 和 `image_data` 可以都为空，客户端展示的时候使用 NFTType Info 中的 metadata。

```rust
 /// The info of NFT type
struct NFTTypeInfo<NFTMeta: copy + store + drop, NFTTypeInfoExt: copy + store + drop> has key, store {
        counter: u64,
        meta: Metadata,
        info: NFTTypeInfoExt,
        mint_events: Event::EventHandle<MintEvent<NFTMeta>>,
}
```

NFTTypeInfo 用于维护 NFT id 的计数器，以及该 NFT 类型的全局 metata，每一种 NFT 类型需要先在注册中心注册。

## 方法定义

每种 NFT 的类型需要先注册，注册时需要 NFT 类型的全局 metadata。

```rust
public fun register<NFTMeta: copy + store + drop, NFTTypeInfoExt: copy + store + drop>(sender: &signer, info: NFTTypeInfoExt, meta: Metadata)
```

注册后 `sender`  账号下会被写入三个权限：

1. MintCapability : 用于铸造该类型的 NFT
2. BurnCapability：用于烧毁该类型的 NFT
3. UpdateCapability：用于更新该类型的 NFT metadata

这三个权限对应三个方法：

```rust
/// 铸造 NFT，返回 NFT 的实例
public fun mint_with_cap<NFTMeta: copy + store + drop, NFTBody: store, Info: copy + store + drop>(creator: address, _cap: &mut MintCapability<NFTMeta>, base_meta: Metadata, type_meta: NFTMeta, body: NFTBody): NFT<NFTMeta, NFTBody>

///烧毁 NFT，返回 NFT 内部嵌套的 NFTBody
public fun burn_with_cap<NFTMeta: copy + store + drop, NFTBody: store>(_cap: &mut BurnCapability<NFTMeta>, nft: NFT<NFTMeta, NFTBody>): NFTBody 

///更新 NFT 的 metadata
public fun update_meta_with_cap<NFTMeta: copy + store + drop, NFTBody: store>(_cap: &mut UpdateCapability<NFTMeta>, nft: &mut NFT<NFTMeta, NFTBody>, base_meta: Metadata, type_meta: NFTMeta)
```

上面列举了 NFT 相关的基本方法，而 NFT 如何存储，如何转让，这个不是 NFT 模块本身关心的事情，是 NFTGallery 的功能。

## NFT 陈列室（NFTGallery） 

NFTGallery 模块提供了用户用来收集和存储 NFT 的基本功能。主要包含以下方法：

```rust
/// 准备 NFTGallery 去接受类型为 NFT<NFTMeta, NFTBody> 的 NFT，用户每接受一种新的 NFT，都需要调用这个方法初始化。
public fun accept<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer)

/// 将 id 为参数 `id` 的 NFT 从 `sender` 转给 `receiver`
public fun transfer<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer, id: u64, receiver: address)

/// 获取 `ownder` 的类型为 NFTMeta 的所有 NFT info, 返回 NFTInfo 列表
public fun get_nft_infos<NFTMeta: copy + store + drop, NFTBody: store>(owner: address):vector<NFT::NFTInfo<NFTMeta>>
   
/// 将 `nft` 存放到 `sender` 的 NFTGallery 中
public fun deposit<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer, nft: NFT<NFTMeta, NFTBody>)

/// 将 `nft` 存放到 `receiver` 的 NFTGallery
public fun deposit_to<NFTMeta: copy + store + drop, NFTBody: store>(receiver: address, nft: NFT<NFTMeta, NFTBody>)

/// 从 `sender` 的 NFTGallery 中取一个类型为 NFTMeta NFT
public fun withdraw_one<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer): NFT<NFTMeta, NFTBody>

/// 从 `sender` 的 NFTGallery 中取一个类型为 NFTMeta，id 为参数 `id` 的 NFT
public fun withdraw<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer, id: u64): Option<NFT<NFTMeta, NFTBody>>
```

NFTGallery 提供了一个空间用来存储和查询 NFT，当然，开发者也可以自行设计 NFT 的存储模块。

## 扩展方式

### 自定义 Metadata

TODO

### 嵌套 NFTBody

TODO

### 自定义存储和转让逻辑

IdentifierNFT

## 使用场景示例

1. NFT 游戏道具
2. NFT 作为会员身份
3. NFT 作为购物凭证
4. 通过 MerkleTree 证明来分发 NFT

