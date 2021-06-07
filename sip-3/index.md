---
sip: 3
title: "[SIP3] Token Economics"
author: Starcoin Foundation
type: paper
category: Core
status: Beta
created: 2020-11-14
weight: 3
---

# Starcoin Economic White Paper

STC is the native token of the Starcoin network, with a total constant issuance of 3,185,136,000.

Currently, it is mainly used for:

1. Paying the GAS fee of the transaction.
2. Paying the state space fee (after the future state billing mechanism is activated).
3. The voting of on-chain governance.

In the future, as the on-chain ecosystem becomes mature, STC will have more and more application scenarios.

## The Genesis Token Distribution

For Starcoin, all Tokens are first-class citizens on the chain, so STC also conforms to the Token specification of Starcoin. After each type of Token is generated, the issuing account will get the seigniorage, and only the accounts with the seigniorage can mint the Token.

The genesis account 0x1 is the issuance account for STC, which is only used for the generation transaction and the meta transaction execution of the blocks (similar to the coinbase transactions for Bitcoin). In the generation transaction, the genesis account will mint all STCs at once and deposit them in the treasury, and then revoke the seigniorage to ensure that no more STCs can be issued.

Then the following operations will be performed:

* 159,256,800 STCs (5%) are withdrawn and transferred to the Starcoin Foundation account (0xA550C18) and will be distributed to early investors.
* A linear withdrawal right with a 3-year release period of 255,129,390 STCs (8%) will be issued to the Starcoin Foundation for ecological construction.
* A linear withdrawal right with 3-year release period of 222,641,010 STCs (7%) will be issued to the Starcoin Foundation for core project development.
* The withdrawal rights of the Treasury will be locked in the DAO, and future withdrawals from the Treasury can only be performed through on-chain governance.

Note: The linear withdrawal right is only a credential of ownership, and Tokens are not involved in the transfer. The owners can periodically withdraw the Tokens from the Treasury that are linearly released by owning the withdrawal rights.

##Block Reward

The block rewards in Starcoin follow the model of linear release in time, and the rewards of each block are calculated by the following equation:

```
block_reward = base_block_reward/base_block_time_target * current_epoch.block_time_target
```

During initialization, base_block_reward is 10 STCs, and base_block_time_target is 10 seconds, which means that one STC is released in each second. If the target block generation time of the current Epoch is 15 seconds, the reward for each block of the current Epoch is 15 STCs.

Among them, base_block_reward and base_block_time_target are on-chain configurations and can be adjusted through on-chain governance. The block target time (block_time_target) of each Epoch will be dynamically adjusted according to the uncle block rate of the previous Epoch (for specific adjustment methods, please refer to the technical white paper).

In order to encourage miners to report uncle blocks in the network, Starcoin will give additional rewards to each miner that has the uncle block header (note: miners who dig out the uncle block are not rewarded).

```
uncle_reward = block_reward * base_reward_per_uncle_percent * uncle_count
```

Among them, base_reward_per_uncle_percent is the on-chain configuration, which is initialized to 10% and can be adjusted through on-chain governance. uncle_count is the number of uncle headers included in the current block with a maximum value of 2. In other words, if a block contains 2 uncle blocks, you can get an additional 20% reward.

<img src="./images/starcoin_block_reward.png" alt="block reward" style="zoom:100%;" align=center />

The reward of each block is extracted from the Treasury by the genesis account, and sent to the miner who offered the block, delayed by N blocks to the account, and the initial value is 7 blocks. If the balance in the Treasury is 0, there is no block reward for this block.

## On-chain Governance

Starcoin has a built-in DAO contract. In the generation transaction, the genesis account will initialize a DAO and set the STC as the governance Token. STC holders can participate in the governance of the following governance items:


1. On-chain configuration changes, such as the aforementioned block reward parameters.
2. The changes of DAO-related parameters.
3. System contract upgrade in Stdlib.
4. Application for withdrawal rights from the Treasury.

For voting, one STC accounts for one vote. When the number of affirmative votes exceeds the minimum number of votes in the DAO and is greater than the number of negative votes, the proposal is accepted. The minimum number of votes is calculated by the multiplication of both the circulating STC amount and the DAO threshold ratio (the initial threshold is 4%). The initial voting period is 7 days. All participating STCs in the voting period will remain locked in the proposal until the end of the voting.

Note: the circulating STC amount = total minted STC amount - Treasury balance. In other words, if the linear withdrawal right is not used, it will not be counted towards actual circulation.

## Treasury Withdrawal Rights

Treasury withdrawal rights are an STC distribution mechanism in addition to miner rewards. Any one can initiate proposals and submit it to the ecological project construction party to apply for withdrawal rights. The following is the initiation process:

1. Anyone can initiate a proposal. The proposal needs to specify the recipient of linear withdrawal rights, the total number of Tokens in the application, and the period of linear release.
2. Community members may support or oppose the proposal via STC voting.
3. After the proposal is passed, the receiver needs to execute the proposal on the chain to obtain linear withdrawal rights.
4. The recipient periodically withdraws funds from the treasury through linear withdrawal rights.
   
## Economic Model Bootstrapping

<img src="./images/starcoin_ecosystem.png" alt="ecosystem" style="zoom:100%;" align=center />

In the current PoW-based public chain, PoW is not only a security-wise consensus mechanism, but also a Token distribution strategy. In the economic model of the PoW chain, Tokens are firstly distributed to miners and then transferred to other ecosystems.

However, for chains such as BTC that target value storage, this model can form an ecological closed loop. But for smart contract chains, the value of Tokens depends on the prosperity of the on-chain ecology, so the Token distribution strategy should be leaning toward the upper-level ecological applications, and at the same time, Tokens need to be distributed through ecological construction, not simply through miners.

In the long run, the basic on-chain ecological benefits are finally included in the Treasury. If the funds in the Treasury can finally cover future R&D investment and miner rewards, it means that the economic model of the chain has bootstrapped itself up.
