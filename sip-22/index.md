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

NFT implementation and extension protocol

<!--more-->

## Motivation

In smart contracts, Tokens are used to represent fungible digital resources, while NFTs are needed to represent non-fungible resources. in Move, any instance of a type that cannot be copied and dropped can be considered a non-fungible resource and is an NFT. But NFTs need a standard for uniform presentation and collecting and transferring , so the design of this standard.

## Objectives

To provide a common, extensible NFT standard, and providing a basic implementation of NFT-related operations.

## Type Definition

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

NFT is a Move type that supports store ability, but not copy and drop, and contains some basic meta-information: 

1. creator: address of the NFT creator.
2. id: the unique id under this NFT type. 
3. base_meta: basic generic metadata information, mainly used to define how to display NFT. 
4. type_meta: developer-defined metadata, also used to mark the type of NFT. metadata is not a resource, it represent information, so it supports copy + store + drop.
5. body: NFT contains resources, can be used to embed other resources.

If you think of NFT as a box, NFT itself defines the box's ownership, unique number, and presentation, and NFTBody is the gem encapsulated in the box. Presentation is defined by Metadata.

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

Metadata defines the basic information needed to display the NFT, name, image and description. If there is other information that needs to be extended, it can be defined in the type_meta. Image has two fields to represent, `image` indicates the image url, `image_data` can be directly stored in the image binary data, the client use the non-empty field in `image` and `image_data` to display the NFT.

In addition, some type of NFT will use the same image for all instance, in this case, the `image` and `image_data` in the NFT metadata can both be empty, the client  using the metadata in the NFTTypeInfo.


```rust
 /// The info of NFT type
struct NFTTypeInfo<NFTMeta: copy + store + drop, NFTTypeInfoExt: copy + store + drop> has key, store {
        counter: u64,
        meta: Metadata,
        info: NFTTypeInfoExt,
        mint_events: Event::EventHandle<MintEvent<NFTMeta>>,
}
```

NFTTypeInfo is used to maintain a counter for the NFT id, as well as the global metata of the NFT type, each NFT type needs to be registered in the registry first.

## Method Definition

Each NFT type needs to be registered first, and the registration requires the NFT's  NFTMeta type and a global metadata for that type.

```rust
public fun register<NFTMeta: copy + store + drop, NFTTypeInfoExt: copy + store + drop>(sender: &signer, info: NFTTypeInfoExt, meta: Metadata)
```

After registration, three capabilities representing permissions will be move to the `sender`'s account.

1. MintCapability : used to mint the type of NFT
2. BurnCapability : used to burn the type of NFT
3. UpdateCapability: used to update the type of NFT's metadata

These three permissions correspond to three methods:

```rust
/// Mint NFT，return NFT's instance
public fun mint_with_cap<NFTMeta: copy + store + drop, NFTBody: store, Info: copy + store + drop>(creator: address, cap: &mut MintCapability<NFTMeta>, base_meta: Metadata, type_meta: NFTMeta, body: NFTBody): NFT<NFTMeta, NFTBody>

/// Burn NFT，return NFTBody in the NFT
public fun burn_with_cap<NFTMeta: copy + store + drop, NFTBody: store>(cap: &mut BurnCapability<NFTMeta>, nft: NFT<NFTMeta, NFTBody>): NFTBody 

/// Update NFT's metadata
public fun update_meta_with_cap<NFTMeta: copy + store + drop, NFTBody: store>(cap: &mut UpdateCapability<NFTMeta>, nft: &mut NFT<NFTMeta, NFTBody>, base_meta: Metadata, type_meta: NFTMeta)
```

The above lists the basic methods related to NFT, and NFT how to store, how to transfer, this is not the NFT module itself care about things, is the function of NFTGallery.

## NFTGallery 

NFTGallery module provides the basic functions that users use to collect and store NFT. It contains the following methods:

```rust

  /// Init a NFTGallery to accept NFT<NFTMeta, NFTBody> for `sender`
public fun accept<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer);

/// Transfer NFT from `sender` to `receiver`
public fun transfer<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer, id: u64, receiver: address);

/// Get the all NFT info
public fun get_nft_infos<NFTMeta: copy + store + drop, NFTBody: store>(owner: address): vector<NFT::NFTInfo<NFTMeta>>;

/// Deposit nft to `sender` NFTGallery
public fun deposit<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer, nft: NFT<NFTMeta, NFTBody>);

/// Deposit nft to `receiver` NFTGallery
public fun deposit_to<NFTMeta: copy + store + drop, NFTBody: store>(receiver: address, nft: NFT<NFTMeta, NFTBody>);

/// Withdraw one nft of NFTMeta from `sender`, caller should ensure at least one NFT in the Gallery.
public fun withdraw_one<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer): NFT<NFTMeta, NFTBody>;

/// Withdraw nft of NFTMeta and id from `sender`
public fun withdraw<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer, id: u64): Option<NFT<NFTMeta, NFTBody>>;

```

NFTGallery provides a generic space for storing and querying NFT, of course, developers can also design their own NFT storage module.

## Extensions

### Custom Metadata

If the developer needs to add a custom metadata, you can define it in the NFTMeta type, for example, to define a video NFT, you need to add a video url.

```rust
struct VideoNFT has copy, store, drop {
  video_url: vector<u8>,
}
struct VideoNFTBody has store{}
```

The actual NFT data format is equivalent to.

```rust
struct NFT{
  creator: address,
  id: u64,
  base_meta: Metadata,
  type_meta: VideoNFT,
  body: VideoNFTBody,
}
```

### Nested NFTBody

If the developer wants to embed additional resources in the NFT, this can be done by way of a custom Body, for example, in the VideoNFTBody above where we wants to embed some Token: 

```rust
struct VideoNFTBody has store{
  token: Token<STC>,
}
```

### Custom transfer

There are NFT application scenarios, NFT transfer is restricted, such as as membership credentials. In this case, a custom NFT storage is needed to implement a custom transfer method. The NFT stored in the NFTGallery is fully controlled by the user, and the NFT developer cannot restrict its use and transfer.

Take IdentifierNFT as an example, IdentifierNFT is a kind of NFT container, which guarantees that each user can only have one NFT of the same type and cannot be transferred by the user after the NFT developer grants the user an NFT, which is generally used in user identity-related NFT scenarios, such as medals of honor.

```rust
/// IdentifierNFT contains an Option<NFT>, which is empty by default, which is equivalent to a box that can hold NFT 
struct IdentifierNFT<NFTMeta: copy + store + drop, NFTBody: store> has key {
        nft: Option<NFT<NFTMeta, NFTBody>>,
}

/// The user uses the `accept` method to initialize an empty IdentifierNFT under his account 
public fun accept<NFTMeta: copy + store + drop, NFTBody: store>(sender: &signer) {
  move_to(sender, IdentifierNFT<NFTMeta, NFTBody> {
    nft: Option::none(),
  });
}

/// The developer grants the nft to the receiver by MintCapability and embeds the nft into the IdentifierNFT 
public fun grant_to<NFTMeta: copy + store + drop, NFTBody: store>(_cap: &mut MintCapability<NFTMeta>, receiver: address, nft: NFT<NFTMeta, NFTBody>) acquires IdentifierNFT {
     let id_nft = borrow_global_mut<IdentifierNFT<NFTMeta, NFTBody>>(receiver);
     Option::fill(&mut id_nft.nft, nft);
}

/// Developers can also take out the NFT in the `owner` IdentifierNFT by BurnCapability 
public fun revoke<NFTMeta: copy + store + drop, NFTBody: store>(_cap: &mut BurnCapability<NFTMeta>, owner: address): NFT<NFTMeta, NFTBody>  acquires IdentifierNFT {
     let id_nft = move_from<IdentifierNFT<NFTMeta, NFTBody>>(owner);
     let IdentifierNFT { nft } = id_nft;
     Option::destroy_some(nft)
}
```

In the above scenario, the NFTMeta definition and the registered developer can programmatically define the transfer logic of the NFT (of course, it is possible to disallow the transfer).

## Example usage scenarios

### NFT game props

One advantage of using the NFT in Move to define game props is that you can define multiple NFT types in the same Module, and you can also define the combination of props, such as: 

```rust
/// The player combines two L2 cards into a new L3 card   
struct L1Card has store {}
struct L2Card has store {
     first: L1Card,
     second: L1Card,
 }
```
For a more detailed example see [nft_card.move](https://github.com/starcoinorg/starcoin/blob/master/vm/functional-tests/tests/testsuite/nft/nft_card.move)

### NFT as membership

A user pays the membership fee to become a member, the membership fee is locked in the NFT. When the user performs a membership action or quit, the membership is checked for expiration and the membership fee is deducted according to the join time. For a more detailed example, see [identifier_nft.move](https://github.com/starcoinorg/starcoin/blob/master/vm/functional-tests/tests/testsuite/nft/identifier_nft.move)

### NFT as shopping credentials

Users directly mint NFTs by paying Token and then exchange NFTs for physical goods. Users also can trade NFTs between themselves. for a more detailed example,see [nft_boxminer.move](https://github.com/starcoinorg/starcoin/blob/master/vm/functional-tests/tests/testsuite/nft/nft_boxminer.move)

### Distribute NFTs via MerkleTree proofs

The NFT developer defines the NFT and submits the MerkleTree root on the chain, and the user takes the NFT proof and mint it by themselves.

Starcoin's GenesisNFT is minted in this way. GenesisNFT is distributed to the addresses of accounts that participated in Barnard network mining before the starcoin mainnet was launched. 

The parent_hash(0x0f2fdd39d11dc3d25f21d05078783d476ff98ca4035320e5932bb3938af0e827) of the main network's genesis block points to the block on the Barnard network with a height of 310,000. All addresses on the Barnard network that have minted blocks prior to this height can mint a GenesisNFT. 

For details, see [GenesisNFT.move](https://github.com/starcoinorg/starcoin/blob/master/vm/stdlib/modules/GenesisNFT.move) [MerkleNFT.move](https://github.com/starcoinorg/starcoin/blob/master/vm/stdlib/modules/MerkleNFT.move)

## Differences between Starcoin NFT standard and ERC721/ERC1155

1. ERC721/ERC1155 is Interface and does not define the implementation, extensibility is achieved through different implementations. The Starcoin NFT standard contains implementations of data types and basic operations, and extensibility is achieved through upper-level combinations. 
2. By default, Starcoin NFT, similar to ERC721, is not fungible. However, third parties can extend the splitting and merging logic to achieve the purpose of ERC1155.
3. The NFT of ERC721/ERC1155 can only be moved within the contract, and cannot be moved from the contract to another contract, so the agreement combination on the NFT is very difficult. Thanks to the type feature of Move, the NFT in Starcoin can be moved between different contracts, and other contracts can define new types to encapsulate the NFT and expand new transfer logic (such as auctions). This brings great convenience to the protocol design on the NFT, and can combine a lot of gameplay. 
4. ERC721/ERC1155 distinguishes NFT types by contract address. To implement multiple NFTs, multiple contracts need to be deployed. If there are many types of NFTs, contract calls will be very complicated. 
5. Starcoin's NFT is stored in the user's state space, and all of the user's NFTs can be displayed by listing the resources of the user's state space, including NFTs embedded in other resource. This brings great convenience to ecological tools, such as displaying NFTs in wallets or block explorer, or the auction market. 
