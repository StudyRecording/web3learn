# Hardhat 环境搭建和配置

## 前置要求

### 1. Bun 安装（推荐）

本教程使用 **Bun** 作为 JavaScript/TypeScript 运行时，它比 Node.js 更快、更简洁。

```bash
# 安装 Bun
curl -fsSL https://bun.sh/install | bash

# 验证安装
bun --version
# 期望输出: 1.x.x 或更高
```

> 也可以使用 Node.js 16+，但 Bun 是首选。如需使用 Node.js，可参考附录：
> ```bash
> # 使用 nvm 安装 Node.js（替代方案）
> curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
> nvm install --lts
> ```

```bash
# macOS/Linux
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc  # 或 ~/.zshrc

# 安装 LTS 版本
nvm install --lts
nvm use --lts

# Windows 用户可以使用 nvm-windows
# 下载地址: https://github.com/coreybutler/nvm-windows/releases
```

### 2. 包管理器

Bun 内置包管理器，无需额外安装。与 npm/pnpm/yarn 命令对比：

| 操作 | npm | Bun |
|------|-----|-----|
| 安装依赖 | `npm install` | `bun install` |
| 安装开发依赖 | `npm install --save-dev <pkg>` | `bun add --dev <pkg>` |
| 运行脚本 | `npm run <script>` | `bun run <script>` |
| 执行 CLI | `npx <command>` | `bunx <command>` |

## 创建 Hardhat 项目

### 方式一：交互式创建（推荐新手）

```bash
# 创建项目目录
mkdir my-hardhat-project
cd my-hardhat-project

# 初始化项目
bun init -y

# 安装 Hardhat
bun add --dev hardhat

# 创建 Hardhat 项目
bunx hardhat init
```

你会看到以下选项：

```
888    888                      888 888               888
888    888                      888 888               888
888    888                      888 888               888
8888888888  8888b.  888d888 .d88888 88888b.   .d8888b 88888b.  888  888
888    888     "88b 888P"  d88" 888 888 "88b d88" 888 888 "88b 888  .88P
888    888 .d888888 888    888  888 888  888 888  888 888  888 888 d88"
888    888 888  888 888    Y88b 888 888  888 Y88b 888 888  888 8888888"
888    888  "Y888888 888     "Y88888 888  888  "Y88888 888  888 888  .d888P
                                             888  888
                                             888  888
                                             888  888
Welcome to Hardhat v2.19.0

? What do you want to do? …
❯ Create a JavaScript project
  Create a TypeScript project
  Create an empty hardhat.config.js
  Quit
```

**选择建议**：
- `Create a TypeScript project`：推荐，类型安全，IDE 支持更好
- `Create a JavaScript project`：简单快速，适合快速原型

### 方式二：手动创建（理解项目结构）

```bash
mkdir my-hardhat-project
cd my-hardhat-project
bun init -y
bun add --dev hardhat

# 手动创建目录结构
mkdir -p contracts scripts test

# 创建配置文件
touch hardhat.config.js
```

## 项目结构解析

一个典型的 Hardhat 项目结构：

```
my-hardhat-project/
├── contracts/              # Solidity 合约目录
│   └── Lock.sol           # 示例合约
├── scripts/               # 部署和交互脚本
│   └── deploy.js          # 部署脚本
├── test/                  # 测试文件
│   └── Lock.js            # 测试文件
├── hardhat.config.js      # Hardhat 配置文件
├── package.json           # npm 配置
└── package-lock.json      # 依赖锁定文件
```

### 与 Java 项目对比

| Hardhat | Java | 说明 |
|---------|------|------|
| contracts/ | src/main/java/ | 源代码目录 |
| test/ | src/test/java/ | 测试代码目录 |
| scripts/ | - | 部署脚本，Java 项目通常没有 |
| hardhat.config.js | pom.xml/build.gradle | 项目配置 |
| node_modules/ | target/ | 依赖和编译产物 |

## 配置详解

### 基础配置

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: "0.8.24",
};
```

### 完整配置示例

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("@nomicfoundation/hardhat-verify");
require("hardhat-gas-reporter");
require("solidity-coverage");
require("hardhat-deploy");
require("dotenv").config();

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  // Solidity 编译器配置
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: {
        enabled: true,
        runs: 200,  // 优化次数，影响 gas
      },
      viaIR: true,  // 启用 IR 编译器，用于复杂合约
    },
  },

  // 多版本编译器配置
  // solidity: {
  //   compilers: [
  //     {
  //       version: "0.8.24",
  //       settings: {
  //         optimizer: {
  //           enabled: true,
  //           runs: 200,
  //         },
  //       },
  //     },
  //     {
  //       version: "0.7.6",
  //       settings: {},
  //     },
  //   ],
  // },

  // 网络配置
  networks: {
    // 本地测试网络（Hardhat Network）
    hardhat: {
      chainId: 31337,
      // 模拟主网分叉
      // forking: {
      //   url: process.env.MAINNET_RPC_URL,
      //   blockNumber: 15000000,
      // },
    },

    // 本地节点（hardhat node）
    localhost: {
      url: "http://127.0.0.1:8545",
      chainId: 31337,
    },

    // Sepolia 测试网
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "https://rpc.sepolia.org",
      accounts: process.env.PRIVATE_KEY 
        ? [process.env.PRIVATE_KEY]
        : [],
      chainId: 11155111,
    },

    // 主网
    mainnet: {
      url: process.env.MAINNET_RPC_URL || "https://eth.llamarpc.com",
      accounts: process.env.PRIVATE_KEY 
        ? [process.env.PRIVATE_KEY]
        : [],
      chainId: 1,
    },

    // Polygon
    polygon: {
      url: process.env.POLYGON_RPC_URL || "https://polygon-rpc.com",
      accounts: process.env.PRIVATE_KEY 
        ? [process.env.PRIVATE_KEY]
        : [],
      chainId: 137,
    },

    // Arbitrum
    arbitrum: {
      url: process.env.ARBITRUM_RPC_URL || "https://arb1.arbitrum.io/rpc",
      accounts: process.env.PRIVATE_KEY 
        ? [process.env.PRIVATE_KEY]
        : [],
      chainId: 42161,
    },
  },

  // Etherscan 验证配置
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY,
      sepolia: process.env.ETHERSCAN_API_KEY,
      polygon: process.env.POLYGONSCAN_API_KEY,
      arbitrumOne: process.env.ARBISCAN_API_KEY,
    },
  },

  // Gas 报告配置
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_API_KEY,
    outputFile: "gas-report.txt",
    noColors: true,
  },

  // 测试覆盖率配置
  // solidity-coverage 会自动使用

  // 路径配置
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts",
  },

  // Mocha 测试配置
  mocha: {
    timeout: 40000,  // 测试超时时间
  },
};
```

### TypeScript 配置

如果选择 TypeScript 项目，会自动生成 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "declaration": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["./scripts", "./test", "./hardhat.config.ts"],
  "files": ["./hardhat.config.ts"]
}
```

## 环境变量管理

创建 `.env` 文件存储敏感信息：

```bash
# .env
# 私钥（切勿提交到 git！）
PRIVATE_KEY=your_private_key_here

# RPC 节点 URL
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/your-api-key
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/your-api-key
POLYGON_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/your-api-key

# API Keys
ETHERSCAN_API_KEY=your_etherscan_api_key
POLYGONSCAN_API_KEY=your_polygonscan_api_key
COINMARKETCAP_API_KEY=your_coinmarketcap_api_key

# Gas 报告开关
REPORT_GAS=true
```

`.gitignore` 必须包含：

```gitignore
# .gitignore
node_modules
.env
cache
artifacts
coverage
coverage.json
typechain
typechain-types

# Hardhat files
cache
artifacts

# others
.idea
.vscode
*.log
```

安装 dotenv：

```bash
bun add dotenv
```

## 常用插件

### 核心插件

```bash
# Hardhat Toolbox（包含常用工具的套件）
bun add --dev @nomicfoundation/hardhat-toolbox

# 包含：
# - @nomicfoundation/hardhat-ethers: Ethers.js 集成
# - @nomicfoundation/hardhat-chai-matchers: Chai 断言扩展
# - @nomicfoundation/hardhat-network-helpers: 网络辅助函数
# - @nomicfoundation/hardhat-verify: 合约验证
# - hardhat-gas-reporter: Gas 报告
# - solidity-coverage: 代码覆盖率
```

### 其他实用插件

```bash
# Gas 报告
bun add --dev hardhat-gas-reporter

# 代码覆盖率
bun add --dev solidity-coverage

# 合约验证
bun add --dev @nomicfoundation/hardhat-verify

# 部署管理
bun add --dev hardhat-deploy

# 可升级合约
bun add --dev @openzeppelin/hardhat-upgrades

# Prettier 格式化
bun add --dev prettier prettier-plugin-solidity

# Solhint 静态分析
bun add --dev solhint
```

## 验证安装

创建一个简单的合约来验证环境：

```solidity
// contracts/SimpleStorage.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract SimpleStorage {
    uint256 private storedData;

    event ValueChanged(uint256 newValue);

    function set(uint256 x) public {
        storedData = x;
        emit ValueChanged(x);
    }

    function get() public view returns (uint256) {
        return storedData;
    }
}
```

编译合约：

```bash
bunx hardhat compile
```

输出：

```
Compiling 1 file with 0.8.24
Compilation finished successfully
```

## 常用命令

```bash
# 编译合约
bunx hardhat compile

# 清理编译产物
bunx hardhat clean

# 运行测试
bunx hardhat test

# 运行特定测试文件
bunx hardhat test test/SimpleStorage.js

# 启动本地节点
bunx hardhat node

# 运行脚本
bunx hardhat run scripts/deploy.js

# 部署到指定网络
bunx hardhat run scripts/deploy.js --network sepolia

# 查看账户
bunx hardhat accounts

# 检查合约大小
bunx hardhat size-contracts

# 生成类型定义（TypeScript）
bunx hardhat typechain

# 验证合约
bunx hardhat verify --network sepolia <CONTRACT_ADDRESS> <CONSTRUCTOR_ARGS>

# 帮助
bunx hardhat help
```

## VSCode 集成

### 推荐扩展

1. **Solidity** (Nomic Foundation) - 语法高亮和智能提示
2. **Hardhat Solidity** - Hardhat 特定支持
3. **ESLint** - JavaScript/TypeScript 检查
4. **Prettier** - 代码格式化

### 配置文件

```json
// .vscode/settings.json
{
  "solidity.compileUsingRemoteVersion": "v0.8.24",
  "solidity.packageDefaultDependenciesContractsDirectory": "contracts",
  "solidity.packageDefaultDependenciesDirectory": "node_modules",
  "editor.formatOnSave": true,
  "[solidity]": {
    "editor.defaultFormatter": "NomicFoundation.hardhat-solidity"
  },
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

```json
// .vscode/extensions.json
{
  "recommendations": [
    "NomicFoundation.hardhat-solidity",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode"
  ]
}
```

## 故障排除

### 常见问题

1. **编译器版本不匹配**
```
Error: Source file requires different compiler version
```
解决：检查 `hardhat.config.js` 中的 `solidity` 版本设置

2. **依赖安装失败**
```bash
# 清除缓存
rm -rf node_modules bun.lockb
bun install

# 或者使用 Node.js 缓存清理（如果使用 npm）
# npm cache clean --force
# rm -rf node_modules package-lock.json
# npm install
```

3. **端口被占用（启动本地节点）**
```bash
# 查找占用 8545 端口的进程
lsof -i :8545  # macOS/Linux
netstat -ano | findstr :8545  # Windows

# 终止进程或使用其他端口
bunx hardhat node --port 8546
```

4. **内存不足**
```bash
# 增加 Bun 内存限制
export BUN_JSC_forceRAMSize=4294967296  # 4GB
# 或使用 Node.js 内存限制（如果使用 Node）
# export NODE_OPTIONS="--max-old-space-size=4096"
```

## 下一步

环境搭建完成后，继续学习 [Hardhat 开发流程](./02-hardhat-development.md)，了解如何编译、测试和部署智能合约。
