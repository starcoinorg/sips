---
sip: 3
title: "[SIP4] 通过链上治理实现特性开关"
author: Starcoin Core Dev
type: paper
category: Core
status: Draft
created: 2021-04-11
weight: 4
---

# 通过链上治理实现特性开关

## 动机

区块链上的协议如果需要升级，并且无法与旧的版本兼容，也是大家常说的『硬分叉』升级，通常的做法是通过高度设置一个特性开关(feature flag)，约定好在某个高度激活该升级。但这种方式面临几个挑战：

1. 代码要合并进主分支的时候，就需要确定一个激活的高度，如果社区对是否支持该特性，以及激活时间不能达成一致，则会导致该特性的代码一直不能合并进入主代码分支，无法进入版本发布流程。
2. 代码合并进入主分支并发布版本后，到激活高度这段时间，是留给矿工（或者验证者）的"投票"时间，矿工通过是否升级来进行投票。如果大多数算力的矿工决定不升级，则会导致链分叉为两条。

按照 Starcoin 白皮书中的治理设计原则『技术创造可能，社区决定取舍』，我们对链上的协议升级做出了改进，通过链上的特性开关机制和 DAO 来实现不兼容性协议升级。


## 技术方案

定义一个链上的 FeatureFlag，并提供判断方法来检测开关是否打开。


```rust
module FeatureFlag{
    
    struct FeatureFlag has store{
        sip: u64,
        /// This feature's active time 
        active_time: Option<u64>,    
    }

    
    public fun is_active(account: address, sip: u64): bool {
        // check the feature flag is active by onchain time.
    }

}
```

在当前的 DAO 系统中新增一种 FeatureFlagProposal, 通过 DAO 投票来决定激活高度。

```rust

struct FeatureFlagProposal{
    sip: u64, 
    active_time: u64,
}

```

在链的代码框架中提供简便的方法来根据链的状态判断某个 FeatureFlag 是否激活。

## 升级流程

1. 用户或者社区开发者提出需要升级的特性需求，并提交 SIP 提案。
2. 核心开发者从技术角度评估该 SIP 并决定是否接受。
3. 接受的 SIP 进入开发流程，开发测试完成后，需要合并到主代码分支的时候，核心开发者只需要通过以下角度进行评估：
   * 实现的特性是否和 SIP 的目标相符。
   * 新的逻辑是否被特性开关控制，是否会影响旧的逻辑。
   * 是否满足技术性指标：比如性能，可维护性等。
4. 合并之后，新的版本中就会携带改特性的逻辑，但出于未激活状态，矿工正常升级即可。
5. 发起 FeatureFlagProposal, 呼吁社区进行投票，决定是否激活改特性， 以及激活的时间。
6. 投票通过，等到链上时间到达激活时间，特性开关生效，全网启用新特性。
7. 投票未通过，该特性将一直处于未激活状态，等待必要的时候清理。


## 期望效果

加速功能以及版本的迭代速度，让争议后置处理。实现『技术创造可能，社区决定取舍』的目标。
