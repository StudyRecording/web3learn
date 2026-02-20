# 第七章：实战项目

## 本章目标

通过四个完整的实战项目，将前面学习的知识融会贯通，掌握 Web3 全栈开发的完整流程。

## 项目列表

| 项目 | 难度 | 技术栈 | 学习重点 |
|------|------|--------|----------|
| [ERC-20 代币](./01-token.md) | ⭐⭐ | Solidity + Hardhat + React | 代币标准、铸造/转账、授权机制 |
| [NFT 市场](./02-nft.md) | ⭐⭐⭐ | Solidity + OpenZeppelin + Next.js | NFT 标准、元数据、交易市场 |
| [简易 DEX](./03-defi.md) | ⭐⭐⭐⭐ | Solidity + Uniswap V2 思想 | AMM、流动性池、价格计算 |
| [DAO 治理](./04-dao.md) | ⭐⭐⭐⭐ | Solidity + 治理模式 | 提案机制、投票权重、链上执行 |
| [进阶项目](./05-advance.md) | ⭐⭐⭐⭐⭐ | 综合 | 跨链、Layer2、隐私计算 |

## 项目架构概览

每个项目都包含以下完整内容：

```
project/
├── contracts/          # Solidity 合约
│   └── *.sol
├── test/              # 测试用例
│   └── *.test.ts
├── scripts/           # 部署脚本
│   └── deploy.ts
├── frontend/          # 前端应用
│   ├── src/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── utils/
│   └── package.json
└── README.md
```

## 技术栈说明

### 后端（区块链层）
- **Solidity**: 智能合约开发语言
- **Hardhat**: 开发环境、测试框架
- **OpenZeppelin**: 安全的合约库
- **Ethers.js**: 区块链交互库

### 前端（用户界面）
- **React/Next.js**: 前端框架
- **TailwindCSS**: 样式框架
- **Wagmi/Viem**: React Web3 钩子库
- **RainbowKit**: 钱包连接组件

### 工具链
- **TypeScript**: 类型安全
- **ESLint/Prettier**: 代码规范
- **Git**: 版本控制

## 学习建议

### 对于 Java 开发者
1. **测试驱动开发**: 像写 JUnit 一样写合约测试
2. **设计模式**: 关注合约的可升级性、代理模式
3. **异常处理**: Solidity 使用 `require`、`revert`，类似 Java 的异常
4. **状态管理**: 区块链即数据库，合约即服务

### 对于 Rust 学习者
1. **所有权模型**: Solidity 没有所有权概念，但要注意存储布局
2. **Gas 优化**: 类似 Rust 的性能优化思维
3. **安全性**: Rust 的安全理念同样适用于合约开发

## 项目依赖安装

```bash
# 创建项目目录
mkdir web3-projects && cd web3-projects

# 初始化 Hardhat 项目
npx hardhat init

# 安装核心依赖
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npm install --save-dev @openzeppelin/contracts
npm install --save-dev chai ethers

# 安装前端依赖
npm install next react react-dom
npm install wagmi viem @rainbow-me/rainbowkit
npm install tailwindcss postcss autoprefixer
```

## 环境配置

```env
# .env 文件
SEPOLIA_RPC_URL=https://sepolia.infura.io/v3/YOUR_KEY
PRIVATE_KEY=your_private_key
ETHERSCAN_API_KEY=your_api_key
```

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

module.exports = {
  solidity: {
    version: "0.8.20",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200
      }
    }
  },
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]
    },
    localhost: {
      url: "http://127.0.0.1:8545"
    }
  }
};
```

## 下一步

从 [ERC-20 代币项目](./01-token.md) 开始，逐步掌握 Web3 全栈开发技能。
