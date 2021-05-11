---
sip: 3
title: "[SIP4] 引入国库机制，让链的发展可持续"
author: "@jolestar"
type: paper
category: Core
status: Draft
created: 2021-04-13
weight: 5
---

# 引入国库机制，让链的发展可持续

## 背景与动机

国库是一个和 DAO 关联起来的资金库，可以通过治理机制支配其中的资金。当前 Starcoin 的经济模型中，预留了一部分固定的生态基金的 Token 额度，用于发展生态，但固定额度的生态基金有可持续问题。另外通过 PoW 分发的 Token，如果分发完之后，链如何运转，也尚未有明确答案。

所以这个提案中建议引入国库 (Treasury) 机制，为链的可持续的生态系统打好基础，同时也给 DAO 系统补上了不可或缺的一环。
 
## 方案

1. STC 总量恒定，在创世块中一次性铸造出来锁到国库中，并销毁铸造能力。
2. 国库中的的 STC 支取需要通过链上的治理机制进行决策。
3. PoW 矿工奖励也从国库中提取，出块奖励的基础额度通过链上治理机进行调整。
4. 如果 STC 国库没有余额，则出块无奖励。
5. 链的生态k可以创造一些收益，并注入国库。


## 经济模型解释

当前的 PoW 公链，PoW 既是保证安全的一种共识机制，也是 Token 的分发策略。PoW 链的经济模型中，Token 先分发给矿工，然后再流转到其他生态。

但这种模型对于 BTC 这样的以价值存储为目标的链来说，是可以形成生态闭环的，但对于智能合约链来说，Token 的价值依赖于链上的生态的繁荣，所以 Token 分发策略中应该向上层的生态应用倾斜，同时需要通过生态应用分发 Token，而不单纯通过矿工分发。

从长远来看，链上的基础生态收益，最后归入到国库中，国库资金如果最后可以覆盖未来的研发投入以及矿工奖励，说明链的经济模型实现了自举。


## 技术方案

定义一个链上的 Treasury, 这个 Treasury 是一个通用的模块，支持任意 Token，所以第三方 Token 也可以使用国库机制。 只有 Token 但 issuer 可以创造 Treasury. 


```rust
module Treasury{
    
    struct Treasury<TokenT> has store,key {
        balance: Token<TokenT>,
        /// event handle for treasury withdraw event
        withdraw_events: Event::EventHandle<WithdrawEvent>,
        /// event handle for treasury deposit event
        deposit_events: Event::EventHandle<DepositEvent>,
    }

     public fun deposit<TokenT:store>(token: Token<TokenT>) {
        // deposit token to Treasury 
    }


}
```

在当前的 DAO 系统中新增一种 TreasuryProposal, 通过 DAO 投票来决定国库资金的支配。

```rust
module TreasuryWithdrawDaoProposal{
     /// WithdrawToken request.
    struct WithdrawToken has copy, drop, store {
        /// the receiver of withdraw tokens.
        receiver: address,
        /// how many tokens to mint.
        amount: u128,
        /// How long in milliseconds does it take for the token to be released
        period: u64,
    }
}
```

投票通过后，receiver 可以收到一个 LinearWithdrawCapability, 可以通过这个 capability 从 Treasury 中通过线性释放的方式提款。 

```rust
module Treasury{
    

    /// A linear time withdraw capability which can withdraw token from Treasury in a period by time-based linear release.
    struct LinearWithdrawCapability<TokenT> has key, store {
        /// The total amount of tokens that can be withdrawn by this capability
        total: u128,
        /// The amount of tokens that have been withdrawn by this capability
        withdraw: u128,
        /// The time-based linear release start time, timestamp in seconds.
        start_time: u64,
        ///  The time-based linear release period in seconds
        period: u64
    }

     /// Withdraw tokens with given `LinearWithdrawCapability`.
    public fun withdraw_with_linear_capability<TokenT: store>(cap: &mut LinearWithdrawCapability<TokenT>): Token<TokenT> {
       
    }
}
```

## 升级方案

1. Stdlib upgrade package 的 init_script 中需要初始化 Treasury 并销毁 genesis account 下的 MintCapability。
2. 需要修改当前的区块奖励获取方式，不再通过铸币的方式进行区块奖励。