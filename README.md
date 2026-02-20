# Web3 开发学习教程

> 面向有编程经验的开发者，从零开始学习 Web3 开发

## 适用人群

- 有后端开发经验（如 Java、Rust）
- 了解前端基础（HTML/CSS/JavaScript）
- 想要系统学习 Web3 开发

## 学习路线

```
┌─────────────────────────────────────────────────────────────────┐
│                        Web3 学习路线图                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第一阶段：基础入门                                                │
│  ┌──────────────┐    ┌──────────────┐                          │
│  │ 01 Web3基础  │ -> │ 02 以太坊基础 │                          │
│  └──────────────┘    └──────────────┘                          │
│                                                                 │
│  第二阶段：智能合约开发                                            │
│  ┌──────────────┐    ┌──────────────┐                          │
│  │ 03 Solidity  │ -> │ 04 开发框架  │                          │
│  └──────────────┘    └──────────────┘                          │
│                                                                 │
│  第三阶段：应用开发                                                │
│  ┌──────────────┐    ┌──────────────┐                          │
│  │ 05 前端集成  │ -> │ 06 Rust+Web3 │                          │
│  └──────────────┘    └──────────────┘                          │
│                                                                 │
│  第四阶段：实战项目                                                │
│  ┌──────────────┐                                               │
│  │ 07 实战项目  │                                               │
│  └──────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 目录结构

```
WEB3/
├── docs/
│   ├── 01-web3-basics/          # 第一章：Web3基础概念
│   ├── 02-ethereum-fundamentals/ # 第二章：以太坊基础
│   ├── 03-solidity/             # 第三章：Solidity智能合约开发
│   ├── 04-frameworks/           # 第四章：开发框架
│   ├── 05-frontend-integration/ # 第五章：前端与合约交互
│   ├── 06-rust-web3/            # 第六章：Rust与Web3
│   ├── 07-projects/             # 第七章：实战项目
│   ├── 08-tools-and-resources/  # 附录：工具与资源
│   └── 09-faq/                  # 附录：常见问题FAQ
├── examples/                    # 示例代码
└── README.md                    # 本文件
```

## 章节概览

### 第一阶段：基础入门

| 章节                                                  | 内容                      | 预计时间 |
| --------------------------------------------------- | ----------------------- | ---- |
| [01 Web3基础概念](docs/01-web3-basics/README.md)        | 区块链原理、Web2 vs Web3、核心概念 | 2-3天 |
| [02 以太坊基础](docs/02-ethereum-fundamentals/README.md) | 账户、交易、Gas、EVM、共识机制      | 3-4天 |

### 第二阶段：智能合约开发

| 章节                                          | 内容                    | 预计时间 |
| ------------------------------------------- | --------------------- | ---- |
| [03 Solidity开发](docs/03-solidity/README.md) | 语法、数据类型、合约结构、安全实践     | 1-2周 |
| [04 开发框架](docs/04-frameworks/README.md)     | Hardhat、Foundry、测试、部署 | 1周   |

### 第三阶段：应用开发

| 章节 | 内容 | 预计时间 |
|------|------|----------|
| [05 前端集成](docs/05-frontend-integration/README.md) | ethers.js、wagmi、钱包连接 | 1周 |
| [06 Rust与Web3](docs/06-rust-web3/README.md) | Substrate、Solana、链开发 | 1-2周 |

### 第四阶段：实战项目

| 章节 | 内容 | 预计时间 |
|------|------|----------|
| [07 实战项目](docs/07-projects/README.md) | DeFi、NFT市场、DAO等完整项目 | 2-3周 |

## 学习建议

### 对于 Java 开发者

1. **理解不可变性**：智能合约部署后不可修改，需要不同的设计模式
2. **Gas 优化**：类似 Java 的性能优化，但关乎真金白银
3. **事件驱动**：合约事件类似 Java 的事件/监听器模式
4. **安全第一**：合约漏洞可能导致巨额损失

### 对于 Rust 开发者

1. **Substrate 开发**：用 Rust 构建自定义区块链
2. **Solana 开发**：Rust 是 Solana 的主要开发语言
3. **工具开发**：大量 Web3 基础设施用 Rust 编写

## 开发环境准备

```bash
# Bun (推荐 v1.0+)
curl -fsSL https://bun.sh/install | bash
bun --version

# Solidity 编译器
bun add -g solc

# Hardhat
bun add -g hardhat

# Foundry (Rust 编写的快速工具)
curl -L https://foundry.paradigm.xyz | bash
foundryup

# Rust (如果还没安装)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## 学习资源

- [以太坊官方文档](https://ethereum.org/zh/developers/)
- [Solidity 官方文档](https://docs.soliditylang.org/)
- [Foundry Book](https://book.getfoundry.sh/)
- [Hardhat 文档](https://hardhat.org/docs)

## 开始学习

准备好了吗？从 [第一章：Web3基础概念](docs/01-web3-basics/README.md) 开始吧！
