# 4.7 Hardhat 3 新特性

## Hardhat 3 概述

Hardhat 3 是 Nomic Foundation 发布的重大更新，带来了许多新功能和改进。

```
┌─────────────────────────────────────────────────────────────┐
│                   Hardhat 3 主要变化                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Solidity 测试支持                                       │
│     └── 类似 Foundry，可用 Solidity 编写测试                │
│                                                             │
│  2. 多链支持                                                │
│     └── 可配置多个模拟网络（Ethereum、OP Mainnet等）        │
│                                                             │
│  3. ESM-first                                               │
│     └── 配置文件必须使用 ES Modules                         │
│                                                             │
│  4. 网络管理器                                              │
│     └── 可同时管理多个网络连接                              │
│                                                             │
│  5. 测试运行器插件                                          │
│     └── 可选择 Mocha 或 Node.js 内置测试运行器              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 安装和初始化

### 创建新项目

```bash
# 创建项目目录
mkdir my-hardhat3-project
cd my-hardhat3-project

# 初始化 npm
npm init -y

# 安装 Hardhat
bun add --dev hardhat

# 初始化 Hardhat 3 项目
bunx hardhat init
```

### 选择项目类型

```
? What do you want to do? 
❯ Create a JavaScript project
  Create a TypeScript project
  Create a TypeScript project (with Viem)  ← 推荐选择
  Create an empty hardhat.config.js
```

## ESM-first 配置

### 配置文件变化

```javascript
// hardhat.config.js (Hardhat 2 - CommonJS)
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.20",
};

// hardhat.config.js (Hardhat 3 - ESM)
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";

const config: HardhatUserConfig = {
  solidity: "0.8.28",
};

export default config;
```

### package.json 配置

```json
{
  "name": "my-hardhat3-project",
  "type": "module",  // 必须添加
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "test:solidity": "hardhat test --typecheck",
    "deploy": "hardhat run scripts/deploy.ts"
  },
  "devDependencies": {
    "hardhat": "^3.0.0",
    "@nomicfoundation/hardhat-toolbox": "^5.0.0",
    "@nomicfoundation/hardhat-chai-matchers": "^2.0.0",
    "@nomicfoundation/hardhat-ethers": "^3.0.0",
    "@typechain/hardhat": "^9.0.0",
    "@typechain/ethers-v6": "^0.5.0",
    "ethers": "^6.0.0",
    "typescript": "^5.0.0"
  }
}
```

## Solidity 测试

Hardhat 3 支持使用 Solidity 编写测试，类似于 Foundry。

### 项目结构

```
my-hardhat3-project/
├── contracts/
│   └── Counter.sol
├── test/
│   ├── Counter.t.sol       # Solidity 测试
│   └── Counter.ts           # TypeScript 测试（可选）
├── scripts/
│   └── deploy.ts
└── hardhat.config.ts
```

### 编写 Solidity 测试

```solidity
// test/Counter.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "hardhat/console.sol";
import "../contracts/Counter.sol";

contract CounterTest {
    Counter public counter;
    
    function setUp() public {
        counter = new Counter();
    }
    
    function test_InitialCount() public view {
        require(counter.count() == 0, "Initial count should be 0");
    }
    
    function test_Increment() public {
        counter.increment();
        require(counter.count() == 1, "Count should be 1");
    }
    
    function testFuzz_Increment(uint256 times) public {
        vm.assume(times < 100);  // 假设条件
        
        for (uint256 i = 0; i < times; i++) {
            counter.increment();
        }
        
        require(counter.count() == times, "Count mismatch");
    }
    
    // 测试失败情况
    function testFail_DecrementBelowZero() public {
        counter.decrement();  // 应该失败
    }
}
```

### 运行测试

```bash
# 运行所有测试
bunx hardhat test

# 运行特定测试文件
bunx hardhat test test/Counter.t.sol

# 运行带类型检查
bunx hardhat test --typecheck

# 详细输出
bunx hardhat test -vvvv
```

## 多链支持

Hardhat 3 可以配置多个模拟网络，每个可以有不同的链类型。

### 配置多链网络

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";

const config: HardhatUserConfig = {
  solidity: "0.8.28",
  networks: {
    // 本地开发网络
    hardhat: {
      chainType: "l1",  // 以太坊 L1
    },
    // OP 主网模拟
    hardhatOp: {
      chainType: "optimism",
      url: "http://127.0.0.1:8545",
    },
    // 测试网配置
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
    // Arbitrum 测试网
    arbitrumSepolia: {
      url: process.env.ARBITRUM_SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
};

export default config;
```

### 链类型支持

```typescript
// 支持的链类型
type ChainType = "l1" | "optimism" | "arbitrum" | "generic";

// L1 - 以太坊主网
networks: {
  hardhat: {
    chainType: "l1",  // 默认
  }
}

// Optimism
networks: {
  hardhatOp: {
    chainType: "optimism",
  }
}

// 通用（兼容 Hardhat 2 行为）
networks: {
  custom: {
    chainType: "generic",
  }
}
```

## 网络管理器

Hardhat 3 允许在任务中同时管理多个网络连接。

### 使用示例

```typescript
// scripts/multi-chain-deploy.ts
import { ethers } from "hardhat";

async function main() {
  // 获取多个网络连接
  const sepoliaProvider = new ethers.JsonRpcProvider(process.env.SEPOLIA_RPC_URL);
  const arbitrumProvider = new ethers.JsonRpcProvider(process.env.ARBITRUM_RPC_URL);
  
  // 在多个链上部署
  const Counter = await ethers.getContractFactory("Counter");
  
  console.log("Deploying to Sepolia...");
  const counterSepolia = await Counter.connect(sepoliaProvider.getSigner()).deploy();
  console.log("Sepolia address:", await counterSepolia.getAddress());
  
  console.log("Deploying to Arbitrum...");
  const counterArbitrum = await Counter.connect(arbitrumProvider.getSigner()).deploy();
  console.log("Arbitrum address:", await counterArbitrum.getAddress());
}

main().catch(console.error);
```

## 测试运行器插件

Hardhat 3 的测试运行器是插件化的，可以选择不同的运行器。

### 配置测试运行器

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
// 或单独引入
import "@nomicfoundation/hardhat-mocha";
// 或使用 Node.js 内置测试
import "@nomicfoundation/hardhat-node-test";

const config: HardhatUserConfig = {
  solidity: "0.8.28",
  mocha: {
    timeout: 40000,
  },
};

export default config;
```

## Hardhat 2 vs Hardhat 3 对比

| 特性 | Hardhat 2 | Hardhat 3 |
|------|-----------|-----------|
| Solidity 测试 | ❌ | ✅ |
| 多链支持 | ❌ 单一网络 | ✅ 多网络配置 |
| 模块系统 | CommonJS | ESM-first |
| 网络连接 | 单连接 | 多连接管理 |
| 测试运行器 | 内置 Mocha | 可选择插件 |
| Viem 支持 | 需手动配置 | 开箱即用 |
| TypeScript | 可选 | 默认支持 |

## 迁移指南

### 从 Hardhat 2 迁移

```bash
# 1. 更新 package.json
bun add --dev hardhat@latest @nomicfoundation/hardhat-toolbox@latest

# 2. 添加 ESM 支持
# 在 package.json 中添加
{
  "type": "module"
}

# 3. 转换配置文件
# 将 hardhat.config.js 重命名为 hardhat.config.ts
# 更新为 ESM 语法

# 4. 更新脚本和测试
# 将 require() 改为 import
# 将 module.exports 改为 export default
```

### 配置文件迁移示例

```javascript
// Hardhat 2 (旧)
const { ethers } = require("hardhat");

module.exports = {
  solidity: "0.8.20",
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY],
    },
  },
};

// Hardhat 3 (新)
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";

const config: HardhatUserConfig = {
  solidity: "0.8.28",
  networks: {
    hardhat: {
      chainType: "l1",
    },
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL || "",
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
    },
  },
};

export default config;
```

## 最佳实践

### 推荐的项目结构

```
my-project/
├── contracts/
│   ├── interfaces/
│   ├── libraries/
│   └── MyContract.sol
├── test/
│   ├── solidity/           # Solidity 测试
│   │   └── MyContract.t.sol
│   └── typescript/         # TypeScript 集成测试
│       └── MyContract.integration.ts
├── scripts/
│   ├── deploy.ts
│   └── upgrade.ts
├── ignition/
│   └── modules/
│       └── MyContract.ts
├── hardhat.config.ts
├── package.json
└── tsconfig.json
```

### TypeScript 配置

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Node",
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "declaration": true,
    "resolveJsonModule": true,
    "types": ["hardhat/config", "node"]
  },
  "include": ["./scripts", "./test", "./hardhat.config.ts"],
  "exclude": ["node_modules"]
}
```

## 下一步

[返回目录](README.md)
