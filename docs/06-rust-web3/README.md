# 第六章：Rust 与 Web3

> 发挥 Rust 优势，深入 Web3 底层

## 本章目标

- 掌握 Solana 程序开发（Rust 智能合约）
- 理解 Substrate 区块链框架
- 学会使用 Alloy 与以太坊交互
- 了解 Foundry 内部实现原理
- 开发链下工具（索引器、机器人）

## 目录

1. [Solana 开发基础](01-solana.md) - 程序结构、账户模型、CPI 调用
2. [Substrate 框架](02-substrate.md) - 构建自定义区块链
3. [Alloy 库使用](03-alloy.md) - Rust 与以太坊交互
4. [Foundry 内部原理](04-foundry-internal.md) - Rust 编写的开发工具
5. [链下工具开发](05-offchain-tools.md) - 索引器、机器人、监控

## 学习时间

预计 7-10 天

## 前置知识

- 第一章：Web3 基础概念
- 第二章：以太坊基础
- 第三章：Solidity 智能合约
- Rust 编程语言基础

## 为什么 Rust 在 Web3 中如此重要

```
┌─────────────────────────────────────────────────────────────┐
│              Rust 在 Web3 生态系统中的地位                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Solana    │    │  Polkadot   │    │   Near      │     │
│  │  (Rust)     │    │  (Rust)     │    │  (Rust)     │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Foundry    │    │   Alloy     │    │   Reth      │     │
│  │  (Rust)     │    │  (Rust)     │    │  (Rust)     │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  Rust 特性完美契合区块链需求：                                │
│  ✓ 内存安全 - 无 GC，确定性执行                              │
│  ✓ 高性能 - 接近 C 的速度                                    │
│  ✓ 并发安全 - 所有权系统防止数据竞争                         │
│  ✓ 跨平台 - 编译为原生代码                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Rust vs Solidity 开发对比

| 维度 | Solidity (EVM) | Rust (Solana) | Rust (Substrate) |
|------|----------------|---------------|------------------|
| 执行环境 | EVM 虚拟机 | BPF 虚拟机 | WASM |
| 状态模型 | 合约存储 | 账户模型 | Storage 区块链状态 |
| 并发 | 顺序执行 | 并行执行 | 并行执行 |
| 开发难度 | 入门简单 | 较高 | 较高 |
| 安全模型 | Gas 限制 | 计算单元限制 | 权重系统 |
| 升级性 | 代理模式 | 可升级程序 | 运行时升级 |

## Rust 在 Web3 中的主要应用场景

### 1. 智能合约/程序开发

```rust
// Solana 程序示例
use anchor_lang::prelude::*;

#[program]
pub mod hello_world {
    use super::*;
    
    pub fn initialize(ctx: Context<Initialize>, data: String) -> Result<()> {
        let my_account = &mut ctx.accounts.my_account;
        my_account.data = data;
        my_account.owner = ctx.accounts.user.key();
        Ok(())
    }
}

#[account]
#[derive(InitSpace)]
pub struct MyAccount {
    pub owner: Pubkey,
    #[max_len(280)]
    pub data: String,
}
```

对比 Solidity：

```solidity
// Solidity 等价合约
contract HelloWorld {
    struct MyAccount {
        address owner;
        string data;
    }
    
    mapping(address => MyAccount) public accounts;
    
    function initialize(string memory data) public {
        accounts[msg.sender].owner = msg.sender;
        accounts[msg.sender].data = data;
    }
}
```

### 2. 基础设施开发

```rust
// 简化的区块处理（类似 Reth）
pub struct BlockProcessor {
    evm: Evm,
}

impl BlockProcessor {
    pub async fn process_block(&self, block: Block) -> Result<Vec<Receipt>> {
        let mut receipts = Vec::new();
        
        for transaction in block.transactions {
            let result = self.evm.execute(transaction)?;
            receipts.push(result.receipt);
        }
        
        Ok(receipts)
    }
}
```

### 3. 链下工具

```rust
// 事件监听器
use alloy::providers::Provider;

async fn listen_events(provider: impl Provider) {
    let filter = Filter::new()
        .address(contract_address)
        .event("Transfer(address,address,uint256)");
    
    let mut stream = provider.subscribe_logs(&filter).await?;
    
    while let Some(log) = stream.next().await {
        println!("Transfer event: {:?}", log);
    }
}
```

## 核心概念对照

```
┌─────────────────────────────────────────────────────────────┐
│            Solidity vs Rust (Solana) 概念对照               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Solidity              Rust (Solana/Anchor)                │
│  ─────────            ────────────────────                 │
│  contract              program (#[program])                │
│  storage               account (#[account])                │
│  mapping               HashMap/PDA                          │
│  msg.sender            ctx.accounts.user                    │
│  require()             require!() / Result<T, Error>       │
│  event                 emit!() / Event struct               │
│  modifier              constraint (#[account(...)])        │
│  inheritance           组合模式                             │
│  interface             Trait                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 开发环境准备

```bash
# Solana 开发
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
solana --version

# Anchor 框架（Solana 开发框架）
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force
avm install latest

# Substrate 开发
rustup update nightly
rustup target add wasm32-unknown-unknown --toolchain nightly

# Alloy（以太坊交互）
cargo add alloy

# Foundry 源码学习
git clone https://github.com/foundry-rs/foundry
cd foundry && cargo build
```

## 学习路径

```
┌─────────────────────────────────────────────────────────────┐
│  阶段一：Alloy（1-2 天）                                      │
│  - 用熟悉的 Rust 操作以太坊                                   │
│  - 理解 RPC 调用、交易构建                                    │
│  - 快速获得成就感                                             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段二：Solana（3-4 天）                                     │
│  - 学习账户模型                                               │
│  - 理解与 EVM 的根本差异                                      │
│  - 编写第一个 Solana 程序                                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段三：Foundry 内部（1-2 天）                                │
│  - 阅读优秀 Rust 代码                                         │
│  - 理解编译器、测试框架原理                                    │
│  - 提升代码能力                                               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段四：链下工具（2-3 天）                                    │
│  - 实战项目开发                                               │
│  - 索引器、机器人                                              │
│  - 综合运用所学知识                                            │
└─────────────────────────────────────────────────────────────┘
```

## 开始学习

[下一节：Solana 开发基础 →](01-solana.md)
