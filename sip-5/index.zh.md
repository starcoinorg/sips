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

所以这个提案中建议引入国库 (Treasury) 机制，为链的可持续的生态系统打好基础。
 
## 方案

1. STC 总量恒定。
2. 创世块中预挖一部分 n% 的 STC 作为国库的基础生态基金。
3. 通过 PoW 分发的 Token，按时间线性在 Y 年内挖完。
4. 每个块挖出的 STC，可根据出块时间计算得出，其中 t% 奖励给矿工，其他的注入国库。t% 这个比例可通过 DAO 治理调整。
5. 挖完之后，矿工的奖励直接从国库提取，依然沿用原来的计算方式。如果国库中已经没有余额，则没有奖励。
6. 未来链上的一些收益，比如短地址拍卖，可直接注入国库。
7. 国库的资金支配通过 DAO 治理实现。


## 经济模型解释

当前的 PoW 公链，PoW 既是保证安全的一种共识机制，也是 Token 的分发策略。PoW 链的经济模型中，Token 先分发给矿工，然后再流转到其他生态。

但这种模型对于 BTC 这样的以价值存储为目标的链来说，是可以形成生态闭环的，但对于智能合约链来说，Token 的价值依赖于链上的生态的繁荣，所以 Token 分发策略中应该向上层的生态应用倾斜，同时需要通过生态应用分发 Token，而不单纯通过矿工分发。

从长远来看，链上的基础生态收益，最后归入到国库中，国库资金如果最后可以覆盖未来的研发投入以及矿工奖励，说明链的经济模型实现了自举。


## 技术方案

定义一个链上的 Treasury, 这个 Treasury 是一个通用的模块，支持任意 Token，所以第三方 Token 也可以使用国库机制。 


```rust
module Treasury{
    
    struct Treasury<TokenType> has store,key{
        token: Token<TokenType>,
        mint_key: LinearTimeMintKey<TokenType>,
    }

    
    public fun deposit<TokenType>(treasury_addr: address, token: Token<TokenType>) {
        // deposit token to Treasury at treasury_addr
    }

    public fun deposit_mint_key<TokenType>(treasury_addr: address, mint_key: LinearTimeMintKey<TokenType>) {
        // deposit mint_key to Treasury at treasury_addr
    }

}
```

在当前的 DAO 系统中新增一种 TreasuryProposal, 通过 DAO 投票来决定国库资金的支配。

```rust

struct TreasuryProposal{
     /// the receiver of tokens.
    receiver: address,
    /// how many tokens to withdraw from treasury.
    amount: u128,
    /// time lock period, the token is time lokced.
    period: Option<u64>
}

```

调整区块奖励的计算方式：

原来的计算方式：

```
block_reward = base_block_reward * (block_time_target/base_block_time_target)
```

调整为

```
block_issue = (total_stc * percent_of_pow_issue)/y_year_in_millsecond  * block_time_target;

block_reward = base_block_reward * (block_time_target/base_block_time_target)

miner_reward = min(block_issue, block_reward);
treasury_reward = block_issue - miner_reward; 
```

原来 ConsensusConfig 中的 base_block_reward 设置项，不再直接决定区块奖励，而是受到总的发行数量的限制。

## 升级流程

1. 按照 stdlib 的升级模式进行兼容性升级，total_stc 这个值静态定义在 module 代码中，不新增 OnChainConfig 选项，升级后区块奖励已经有变化。
2. 将已经分配的生态基金 LinearTimeMintKey 转入到国库中。
