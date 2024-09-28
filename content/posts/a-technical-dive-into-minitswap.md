---
title: "A Technical Dive into Minitswap"
date: "2024-09-26"
summary: "Exploring how Initia's Minitswap DEX works under the hood." 
description: "Exploring how Initia's Minitswap DEX works under the hood."
toc: true
readTime: true
math: true
tags: ["blockchain", "initia", "cosmossdk"]
showTags: false
hideBackToTop: false
---

## Introduction

## Terminology

### INIT Token Variants

As part of this article, we will be mentioning 3 main variants of the INIT token:

- $INIT$: The original variant of the INIT token on the Initia L1
- $opINIT_m$: The variant of the INIT token that's bridged over the OPinit Bridge to a Minitia $m$
- $ibcOPINIT_m$: The variant of the INIT token that's bridged over the OPinit Bridge to a Minitia $m$ then back down to the Initia L1 via IBC

## Minitswap Pools

Unlike how pools on normal DEXes like Uniswap work, each individual pool on Minitswap are not its own smart contract, but instead [resources](https://aptos.dev/en/network/blockchain/resources) within the same Minitswap module.

Through this design, each virtual pool is still isolated from the others, can have varying and isolated states and parameters not affected by other pools, while still all utilizing the same underlying INIT liquidity.

However, although all virtual pools utilizes the same INIT liquidity as its base, each pool define its own virtual `pool_size` variable that is used to derive the pool's internal liquidity, swap pricing, and other parameters. This separation between "physical" and "virtual" liquidity have the following benefits:

- **independent liquidity management**: each pool can manage its own virtual liquidity without affecting the others
- **better pool parameter management**: more dynamic and granular control over factors such as pool depth, parameters, and other states without strictly being bound by physical liquidity constraints.
- **more customizable pool mechanisms**: not having to interact with physical asset transfers enables complex pool mechanisms to be implemented more easily

The physical INIT (and later ibcOPINIT once swaps start occurring) is then mainly used for transferring the user-desired output asset during swaps.

### Imbalance & Recovery

#### Pool Imbalance

When a new virtual pool is created on Minitswap, the pool's virtual INIT (`init_pool_amount`) and ibcOPINIT (`ibc_op_init_pool_amount`) balances are set to the input `pool_size` value. At this point, the pool's virtual INIT and ibcOPINIT balances are equal, and most swaps gives the desired 1:1 INIT:ibcOPINIT exchange rate. And since there's physically only INIT in the pool, only ibcOPINIT → INIT swaps are possible. As each such swap occurs:

- The virtual pool's INIT balance (`init_pool_amount`) decreases as INIT is transferred out of the pool
- The virtual pool's ibcOPINIT balance (`ibc_op_init_pool_amount`) increases as ibcOPINIT is transferred into the pool

After a period of time, the `init_pool_amount` and `ibc_op_init_pool_amount` gradually becomes out of sync. As more INIT is transferred out of the pool, this cause the relative value of INIT to ibcOPINIT to increase, causing the swap rate to deviate from the desired 1:1 ratio and the pool to become imbalanced.

We define the imbalance as follows:

```js
let imbalance =
    bigdecimal::from_ratio_u64(
        pool.peg_keeper_owned_ibc_op_init_balance
            + pool.ibc_op_init_pool_amount - pool.pool_size,
        pool.pool_size
    );
```

Thus, as more ibcOPINIT → INIT swaps occur, the imbalance increases. We then will be using this imbalance value as the basis for calculating the pool's recovery.

#### Pool Recovery

Minitswap uses a recovery mechanism to bring the pool back to a balanced state. To determine when recovery should occur, we first define two variables; **current ratio** and **fully recovered ratio**.

The **current ratio** ($R_c$) defines the current imbalance of the pool, calculated as follows:

$$
R_{c} = \frac{B_{ibc}}{B_{init} + B_{ibc}}
$$

where:

- $B_{init}$ is the virtual balance of INIT in the pool
- $B_{ibc}$ is the virtual balance of ibcOPINIT in the pool

```
let current_ratio =
    bigdecimal::from_ratio_u64(
        pool.ibc_op_init_pool_amount,
        pool.init_pool_amount + pool.ibc_op_init_pool_amount
    );
```

The **fully recovered ratio** ($R_{fr}$) then defines the target imbalance between INIT and ibcOPINIT in the pool.

$$
R_{fr} = 0.5 + (R_{max} - 0.5) \cdot \frac{(fI)^3}{(1 + |fI|^3)}
$$

where

- $R_{max}$ is the maximum recover ratio, which is 0.9
- $f$ is the flexibility parameter
- $I$ is the imbalance of the pool

### Peg Keeper

### In-House Arbitrage

## Architecture

The Minitswap DEX itself consists of 2 main components:

1. The Move module that houses the Minitswap logic and stores the user-deposited INIT token liquidity
2. The various virtual INIT-ibcOPINIT pools for each Minitia

This design allows for each Minitia to have independent virtual pools with its own separated pool state and parameters, while still all utilizing a unified underlying INIT liquidity.

Let's explore each of these components in detail.

### Minitswap Module

The Minitswap module is precompile Move module on the Initia L1. The module is responsible

- Holding and tracking the INIT liquidity deposited by users
- Storing the global configurations, parameters, states used by all virtual pools
- Defining the overall logic and functionality of the DEX

### Virtual Pools

When each Minitia is whitelisted, a virtual pool is created through the [`create_pool`](https://github.com/initia-labs/movevm/blob/main/precompile/modules/initia_stdlib/sources/minitswap.move#L1003-L1089) function on Minitswap.

Each virtual pool then contains the following key information:

```java
struct VirtualPool has key {
    /// Z. Virtual pool size
    pool_size: u64,
    /// V. Recover velocity. Real recover amount = Vt
    recover_velocity: BigDecimal,
    /// R_max max recover ratio
    max_ratio: BigDecimal,
    /// f. Flexibility
    recover_param: BigDecimal,
    /// Virtual pool amount of INIT
    init_pool_amount: u64,
    /// Virtual pool amount of ibc_op_INIT
    ibc_op_init_pool_amount: u64,
    /// last recovered timestamp
    last_recovered_timestamp: u64,
    /// INIT balance of peg keeper (negative value)
    virtual_init_balance: u64,
    /// ibc op INIT balance of peg keeper
    virtual_ibc_op_init_balance: u64,
    /// ibc op INIT balance of peg keeper which also include unprocessed arb_batch state.
    peg_keeper_owned_ibc_op_init_balance: u64,
    /// ANN
    ann: u64,
    /// Is pool in active
    active: bool,

    // in house arb configs

    /// op bridge id
    op_bridge_id: u64,
    /// ibc channel
    ibc_channel: String,
    /// hook contract
    hook_contract: String,
    /// ongoing in house arb info
    arb_batch_map: Table<vector<u8>, ArbInfo>
    
    //...other variables
}
```

- `pool_size`