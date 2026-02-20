# 第五章：前端与合约交互

本章将系统介绍如何开发 Web3 前端应用，实现 DApp 与智能合约的完整交互流程。

## 学习目标

- 掌握 ethers.js 和 viem 库的使用
- 理解 wagmi React Hooks 的最佳实践
- 实现多种钱包连接方案
- 完成合约的读写操作和事件监听
- 处理交易的生命周期

## 章节目录

| 章节 | 主题 | 说明 |
|------|------|------|
| [01-ethersjs.md](./01-ethersjs.md) | ethers.js 基础 | Web3 最流行的 JavaScript 库 |
| [02-wagmi.md](./02-wagmi.md) | wagmi + viem | 现代 React DApp 开发方案 |
| [03-wallet-connect.md](./03-wallet-connect.md) | 钱包连接 | MetaMask、WalletConnect 等 |
| [04-contract-interaction.md](./04-contract-interaction.md) | 合约交互 | 读取、写入、事件监听 |
| [05-transaction-management.md](./05-transaction-management.md) | 交易管理 | 签名、发送、确认、错误处理 |
| [06-dapp-example.md](./06-dapp-example.md) | 完整 DApp 示例 | 实战项目 |
| [07-wagmi-v2.md](./07-wagmi-v2.md) | wagmi v2 更新 | TanStack Query、新API |

## 技术栈对比

```
┌─────────────────────────────────────────────────────────────┐
│                    Web3 前端技术栈演进                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  web3.js (2015)      → 早期以太坊 JavaScript 库             │
│        ↓                                                    │
│  ethers.js (2018)    → 更轻量、TypeScript 友好              │
│        ↓                                                    │
│  wagmi + viem (2022) → React Hooks、类型安全、性能优化       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 核心概念对照

对于 Java 开发者，以下是 Web3 前端开发与传统后端开发的对比：

| 概念 | Java 后端 | Web3 前端 |
|------|----------|-----------|
| 客户端 | HTTP Client | Provider (JsonRpcProvider) |
| 认证 | Session/JWT | 钱包签名验证 |
| 数据库 | MySQL/PostgreSQL | 智能合约存储 |
| API 调用 | REST/RPC | 合约方法调用 |
| 事务 | @Transactional | 交易 (Transaction) |
| 事件监听 | 消息队列/Event | 合约 Event 监听 |
| 异步处理 | CompletableFuture | Promise/async-await |

## 前置要求

```json
{
  "dependencies": {
    "ethers": "^6.9.0",
    "viem": "^2.0.0",
    "wagmi": "^2.0.0",
    "@tanstack/react-query": "^5.0.0",
    "@wagmi/connectors": "^4.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "^18.2.0"
  }
}
```

## 开发环境准备

```bash
# 创建项目
bun create vite my-dapp --template react-ts

# 安装依赖
cd my-dapp
bun add ethers viem wagmi @tanstack/react-query @wagmi/connectors
```

## 学习建议

1. **先理解原理**：ethers.js 是基础，理解 Provider、Signer、Contract 的概念
2. **掌握工具**：wagmi 提供了更好的开发体验，适合 React 项目
3. **实践为主**：每个章节都有可运行的代码示例
4. **安全意识**：前端与私钥、签名相关的安全最佳实践

## 参考资源

- [ethers.js 官方文档](https://docs.ethers.org/v6/)
- [viem 官方文档](https://viem.sh/)
- [wagmi 官方文档](https://wagmi.sh/)
- [MetaMask 文档](https://docs.metamask.io/)
