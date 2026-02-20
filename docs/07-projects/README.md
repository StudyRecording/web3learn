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
bunx hardhat init

# 安装核心依赖
bun add --dev hardhat @nomicfoundation/hardhat-toolbox
bun add --dev @openzeppelin/contracts
bun add --dev chai ethers

# 安装前端依赖
bun add next react react-dom
bun add wagmi viem @rainbow-me/rainbowkit
bun add tailwindcss postcss autoprefixer
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

## 测试网部署完整流程

### 1. 准备工作

```bash
# 1. 获取测试网 ETH（Sepolia）
# 访问：https://sepoliafaucet.com 或 https://www.alchemy.com/faucets/ethereum-sepolia
# 需要输入你的钱包地址

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env 文件，填入你的配置
```

### 2. 编译合约

```bash
# 清理并重新编译
bunx hardhat clean
bunx hardhat compile

# 检查合约大小
bunx hardhat size-contracts
```

### 3. 本地测试

```bash
# 启动本地节点（终端1）
bunx hardhat node

# 运行测试（终端2）
bunx hardhat test --network localhost

# 本地部署测试
bunx hardhat run scripts/deploy.js --network localhost
```

### 4. 部署到测试网

```bash
# 方式一：使用 Hardhat 脚本
bunx hardhat run scripts/deploy.js --network sepolia

# 方式二：使用 Foundry（如果使用 Foundry）
# forge script script/Deploy.s.sol --rpc-url $SEPOLIA_RPC_URL --broadcast --verify
```

**部署脚本示例：**

```javascript
// scripts/deploy.js
const hre = require("hardhat");

async function main() {
  const [deployer] = await hre.ethers.getSigners();
  console.log("Deploying contracts with account:", deployer.address);
  console.log("Account balance:", (await deployer.provider.getBalance(deployer.address)).toString());

  const Token = await hre.ethers.getContractFactory("MyToken");
  const token = await Token.deploy("My Token", "MTK", 1000000);
  
  await token.waitForDeployment();
  
  console.log("Token deployed to:", await token.getAddress());
  
  // 保存部署信息
  const fs = require('fs');
  const deployments = {
    network: hre.network.name,
    token: await token.getAddress(),
    deployer: deployer.address,
    timestamp: new Date().toISOString()
  };
  fs.writeFileSync('deployments.json', JSON.stringify(deployments, null, 2));
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 5. 验证合约

```bash
# 使用 Hardhat 验证
bunx hardhat verify --network sepolia <CONTRACT_ADDRESS> "My Token" "MTK" "1000000"

# 或者使用 Foundry
# forge verify-contract --chain-id 11155111 --etherscan-api-key $ETHERSCAN_API_KEY <ADDRESS> <CONTRACT_PATH>
```

### 6. 前端配置

```typescript
// src/config/contracts.ts
export const CONTRACTS = {
  sepolia: {
    token: '0x...', // 你的合约地址
  },
  localhost: {
    token: '0x...', // 本地部署地址
  }
};

// 根据网络获取地址
export const getContractAddress = (chainId: number) => {
  switch (chainId) {
    case 11155111: // Sepolia
      return CONTRACTS.sepolia;
    case 31337: // Hardhat Local
      return CONTRACTS.localhost;
    default:
      throw new Error('Unsupported network');
  }
};
```

### 7. 部署检查清单

- [ ] 合约已在本地测试通过
- [ ] 钱包有足够的测试网 ETH（至少 0.1 ETH）
- [ ] 环境变量配置正确
- [ ] 部署脚本已更新构造函数参数
- [ ] 合约已成功部署并验证
- [ ] 前端配置已更新合约地址
- [ ] 在测试网上进行了功能测试

### 8. 常见部署问题

**问题1：insufficient funds**
```
Error: insufficient funds for intrinsic transaction cost
```
解决：从水龙头获取更多测试网 ETH

**问题2：nonce too low**
```
Error: nonce too low
```
解决：等待前一个交易确认，或重置 Metamask 账户 nonce

**问题3：gas limit**
```
Error: transaction underpriced
```
解决：在 hardhat.config.js 中增加 gasPrice

```javascript
networks: {
  sepolia: {
    url: process.env.SEPOLIA_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
    gasPrice: 20000000000 // 20 gwei
  }
}
```

## 下一步

从 [ERC-20 代币项目](./01-token.md) 开始，逐步掌握 Web3 全栈开发技能。
