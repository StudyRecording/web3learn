# Hardhat 开发流程

本章将详细介绍使用 Hardhat 进行智能合约开发的完整流程，包括编译、测试、部署和调试。

## 编译合约

### 基础编译

```bash
npx hardhat compile
```

输出示例：

```
Compiling 2 files with 0.8.24
Compilation finished successfully
```

编译产物存储在 `artifacts/` 目录：

```
artifacts/
├── build-info/
│   └── 12345.json          # 编译元数据
├── contracts/
│   └── SimpleToken.sol/
│       ├── SimpleToken.dbg.json
│       └── SimpleToken.json    # ABI + Bytecode
└── hardhat.config.js.map
```

### 编译配置

```javascript
// hardhat.config.js
module.exports = {
  solidity: {
    version: "0.8.24",
    settings: {
      // 优化器配置
      optimizer: {
        enabled: true,
        runs: 200,  // 优化执行次数 vs 部署大小
      },
      // IR 编译器（用于复杂合约）
      viaIR: false,
      // EVM 版本
      evmVersion: "paris",  // paris, shanghai, cancun
      // 输出配置
      outputSelection: {
        "*": {
          "*": ["abi", "evm.bytecode", "evm.deployedBytecode"]
        }
      }
    }
  }
};
```

**优化器 runs 参数选择**：

| 场景 | runs 值 | 说明 |
|------|---------|------|
| 部署成本低 | 1 | 最小 bytecode，运行 gas 高 |
| 均衡 | 200 | 默认值，平衡 |
| 频繁调用 | 1000+ | 较大 bytecode，运行 gas 低 |

### 多版本编译

当项目依赖不同版本的库时：

```javascript
// hardhat.config.js
module.exports = {
  solidity: {
    compilers: [
      {
        version: "0.8.24",
        settings: {
          optimizer: { enabled: true, runs: 200 }
        }
      },
      {
        version: "0.7.6",
        settings: {
          optimizer: { enabled: true, runs: 200 }
        }
      },
      {
        version: "0.6.12",
        settings: {}
      }
    ],
    // 版本覆盖
    overrides: {
      "contracts/legacy/OldContract.sol": {
        version: "0.5.17",
        settings: {}
      }
    }
  }
};
```

### 查看合约大小

```bash
npx hardhat size-contracts
```

输出：

```
 ·---------------------------|---------------------------|------------------·
 |  Contract Name            ·  Size (KB)                ·  % of Limit      ·
 ·---------------------------|---------------------------|------------------·
 |  SimpleToken              ·  2.34                     ·  11.7 %          ·
 |  ComplexDeFi              ·  18.5                     ·  92.5 %          ·
 ·---------------------------|---------------------------|------------------·
```

> 注意：合约大小限制为 24KB，超过会导致部署失败。

## 测试合约

### 测试框架

Hardhat 使用 Mocha + Chai 作为测试框架（类似 Java 的 JUnit）：

```bash
npm install --save-dev chai @nomicfoundation/hardhat-chai-matchers
```

### 基础测试示例

```javascript
// test/SimpleToken.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleToken", function () {
  let token;
  let owner;
  let addr1;
  let addr2;

  // 每个测试前部署新合约
  beforeEach(async function () {
    [owner, addr1, addr2] = await ethers.getSigners();
    
    const Token = await ethers.getContractFactory("SimpleToken");
    token = await Token.deploy("MyToken", "MTK", 1000000);
    await token.waitForDeployment();
  });

  describe("Deployment", function () {
    it("Should set the right name and symbol", async function () {
      expect(await token.name()).to.equal("MyToken");
      expect(await token.symbol()).to.equal("MTK");
    });

    it("Should assign the total supply to the owner", async function () {
      const ownerBalance = await token.balanceOf(owner.address);
      expect(await token.totalSupply()).to.equal(ownerBalance);
    });

    it("Should set the right decimals", async function () {
      expect(await token.decimals()).to.equal(18);
    });
  });

  describe("Transactions", function () {
    it("Should transfer tokens between accounts", async function () {
      // 从 owner 转账到 addr1
      await token.transfer(addr1.address, 100);
      
      const addr1Balance = await token.balanceOf(addr1.address);
      expect(addr1Balance).to.equal(100);

      const ownerBalance = await token.balanceOf(owner.address);
      expect(ownerBalance).to.equal(
        (await token.totalSupply()) - 100n
      );
    });

    it("Should fail if sender doesn't have enough balance", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);
      
      // 尝试从 addr1 转账（余额为 0）
      await expect(
        token.connect(addr1).transfer(owner.address, 1)
      ).to.be.revertedWith("Insufficient balance");

      // 余额应该不变
      expect(await token.balanceOf(owner.address)).to.equal(initialOwnerBalance);
    });

    it("Should emit Transfer event", async function () {
      await expect(token.transfer(addr1.address, 100))
        .to.emit(token, "Transfer")
        .withArgs(owner.address, addr1.address, 100);
    });

    it("Should update balances after multiple transfers", async function () {
      const initialOwnerBalance = await token.balanceOf(owner.address);

      // 多次转账
      await token.transfer(addr1.address, 100);
      await token.transfer(addr2.address, 50);

      const finalOwnerBalance = await token.balanceOf(owner.address);
      expect(finalOwnerBalance).to.equal(initialOwnerBalance - 150n);

      expect(await token.balanceOf(addr1.address)).to.equal(100);
      expect(await token.balanceOf(addr2.address)).to.equal(50);
    });
  });

  describe("Approve and TransferFrom", function () {
    it("Should approve spending", async function () {
      await token.approve(addr1.address, 100);
      
      expect(
        await token.allowance(owner.address, addr1.address)
      ).to.equal(100);
    });

    it("Should emit Approval event", async function () {
      await expect(token.approve(addr1.address, 100))
        .to.emit(token, "Approval")
        .withArgs(owner.address, addr1.address, 100);
    });

    it("Should transfer from approved account", async function () {
      await token.approve(addr1.address, 100);
      
      await token.connect(addr1).transferFrom(
        owner.address,
        addr2.address,
        50
      );

      expect(await token.balanceOf(addr2.address)).to.equal(50);
      expect(
        await token.allowance(owner.address, addr1.address)
      ).to.equal(50);
    });

    it("Should fail transfer from without approval", async function () {
      await expect(
        token.connect(addr1).transferFrom(owner.address, addr2.address, 50)
      ).to.be.revertedWith("Insufficient allowance");
    });
  });
});
```

### 常用断言

```javascript
const { expect } = require("chai");

// 相等性
expect(value).to.equal(100);
expect(value).to.not.equal(50);

// 大小比较
expect(balance).to.be.above(100);
expect(balance).to.be.below(1000);
expect(balance).to.be.at.least(0);

// 类型检查
expect(address).to.be.a("string");
expect(balance).to.be.a("bigint");

// 包含
expect(array).to.include(element);
expect(string).to.contain("substring");

// 事件
await expect(tx).to.emit(contract, "EventName")
  .withArgs(arg1, arg2, arg3);

// Revert
await expect(tx).to.be.reverted;
await expect(tx).to.be.revertedWith("Error message");
await expect(tx).to.be.revertedWithCustomError(contract, "CustomError");

// 事件（自定义错误）
await expect(tx)
  .to.emit(contract, "CustomError")
  .withArgs(owner.address, 100);

// Gas 消耗
const tx = await contract.someFunction();
const receipt = await tx.wait();
console.log(`Gas used: ${receipt.gasUsed}`);
```

### 网络辅助函数

```javascript
const { time, loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

describe("Time-based Tests", function () {
  it("Should unlock after time period", async function () {
    const lock = await Lock.deploy(oneWeekInSec);
    
    // 快进时间
    await time.increase(oneWeekInSec);
    
    // 或者快进到指定时间戳
    await time.increaseTo(timestamp);
    
    // 获取当前区块时间戳
    const latestTime = await time.latest();
    
    // 现在可以解锁
    await expect(lock.withdraw()).to.not.be.reverted;
  });

  it("Should mine multiple blocks", async function () {
    // 挖掘 5 个区块
    await network.provider.send("hardhat_mine", [5]);
    
    // 挖掘到指定区块
    await network.provider.send("hardhat_mine", [10]);
  });

  it("Should set balance", async function () {
    // 设置账户余额
    await network.provider.send("hardhat_setBalance", [
      addr1.address,
      "0x1000000000000000000"  // 1 ETH
    ]);
  });

  it("Should impersonate account", async function () {
    // 模拟其他账户（如主网合约）
    await network.provider.request({
      method: "hardhat_impersonateAccount",
      params: [targetAddress]
    });
    
    const signer = await ethers.getSigner(targetAddress);
    // 使用 signer 执行操作
    
    await network.provider.request({
      method: "hardhat_stopImpersonatingAccount",
      params: [targetAddress]
    });
  });
});
```

### Fixture 模式

使用 `loadFixture` 提高测试效率，避免每次都重新部署：

```javascript
const { loadFixture } = require("@nomicfoundation/hardhat-network-helpers");

async function deployTokenFixture() {
  const [owner, addr1, addr2] = await ethers.getSigners();
  
  const Token = await ethers.getContractFactory("SimpleToken");
  const token = await Token.deploy("Test", "TST", 1000000);
  await token.waitForDeployment();
  
  // 返回所有需要的状态
  return { token, owner, addr1, addr2 };
}

describe("SimpleToken with Fixture", function () {
  it("Should use fixture", async function () {
    const { token, owner, addr1 } = await loadFixture(deployTokenFixture);
    
    // fixture 会在测试间自动重置状态
    expect(await token.balanceOf(owner.address)).to.equal(
      await token.totalSupply()
    );
  });
});
```

### 运行测试

```bash
# 运行所有测试
npx hardhat test

# 运行特定文件
npx hardhat test test/SimpleToken.js

# 运行匹配的测试
npx hardhat test --grep "transfer"

# 显示 console.log 输出
npx hardhat test --verbose

# 测试覆盖率
npx hardhat coverage
```

## 部署合约

### 基础部署脚本

```javascript
// scripts/deploy.js
const hre = require("hardhat");

async function main() {
  console.log("开始部署...");

  const Token = await hre.ethers.getContractFactory("SimpleToken");
  
  // 部署合约
  const token = await Token.deploy(
    "My Token",
    "MTK",
    1000000
  );

  // 等待部署完成
  await token.waitForDeployment();

  const address = await token.getAddress();
  console.log(`SimpleToken 部署到: ${address}`);

  // 部署多个合约
  const NFT = await hre.ethers.getContractFactory("MyNFT");
  const nft = await NFT.deploy("My NFT", "MNFT");
  await nft.waitForDeployment();
  console.log(`MyNFT 部署到: ${await nft.getAddress()}`);

  // 验证部署参数
  console.log("验证部署:");
  console.log(`  名称: ${await token.name()}`);
  console.log(`  符号: ${await token.symbol()}`);
  console.log(`  总量: ${await token.totalSupply()}`);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 使用 hardhat-deploy 插件

安装插件：

```bash
npm install --save-dev hardhat-deploy
```

创建部署配置：

```javascript
// deployments/localhost/deploy.js
module.exports = async ({ getNamedAccounts, deployments }) => {
  const { deploy, log } = deployments;
  const { deployer } = await getNamedAccounts();

  log("部署 SimpleToken...");

  const token = await deploy("SimpleToken", {
    from: deployer,
    args: ["My Token", "MTK", "1000000"],
    log: true,
    waitConfirmations: 1,  // 本地网络
  });

  log(`SimpleToken 部署到 ${token.address}`);
  
  // 部署依赖合约
  const exchange = await deploy("Exchange", {
    from: deployer,
    args: [token.address],
    log: true,
  });

  log(`Exchange 部署到 ${exchange.address}`);
};

module.exports.tags = ["all", "token"];
```

```javascript
// hardhat.config.js
require("hardhat-deploy");

module.exports = {
  namedAccounts: {
    deployer: {
      default: 0,  // 第一个账户
      1: 0,        // 主网
      5: 0,        // Goerli
    },
  },
};
```

部署命令：

```bash
# 本地部署
npx hardhat deploy

# 部署到测试网
npx hardhat deploy --network sepolia

# 部署特定标签
npx hardhat deploy --tags token

# 导出部署地址
npx hardhat export --export deployments.json
```

### 带 constructor 参数的部署

```javascript
// scripts/deployWithArgs.js
async function main() {
  const [deployer] = await ethers.getSigners();
  
  console.log("部署账户:", deployer.address);
  console.log("账户余额:", (await ethers.provider.getBalance(deployer.address)).toString());

  const Token = await ethers.getContractFactory("SimpleToken");
  
  // 带 JSON 参数部署
  const constructorArgs = [
    "My Token",
    "MTK",
    ethers.parseUnits("1000000", 18)  // 使用 ethers 解析大数
  ];

  // 估算 gas
  const deployTx = await Token.getDeployTransaction(...constructorArgs);
  const estimatedGas = await ethers.provider.estimateGas(deployTx);
  console.log("预估 Gas:", estimatedGas.toString());

  // 部署
  const token = await Token.deploy(...constructorArgs, {
    gasLimit: estimatedGas * 12n / 10n  // 增加 20% 余量
  });

  await token.waitForDeployment();
  console.log("部署成功:", await token.getAddress());
}

main().catch(console.error);
```

### 验证合约

部署到测试网或主网后，需要在区块浏览器上验证：

```bash
# 自动验证（需要在 hardhat.config.js 配置 etherscan）
npx hardhat run scripts/deploy.js --network sepolia

# 手动验证
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> "My Token" "MTK" "1000000"

# 验证代理合约
npx hardhat verify --network sepolia --contract contracts/Implementation.sol:Implementation <PROXY_ADDRESS>
```

验证配置：

```javascript
// hardhat.config.js
module.exports = {
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY,
      sepolia: process.env.ETHERSCAN_API_KEY,
      polygon: process.env.POLYGONSCAN_API_KEY,
    },
  },
};
```

## 调试技巧

### console.log 调试

```solidity
// contracts/DebugExample.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "hardhat/console.sol";

contract DebugExample {
    function debugLog(uint256 value) public view {
        console.log("Value is:", value);
        console.log("Block timestamp:", block.timestamp);
        console.log("Sender:", msg.sender);
        
        // 格式化输出
        console.log("Formatted: %s has %d tokens", msg.sender, value);
    }
}
```

测试时查看输出：

```bash
npx hardhat test --verbose
```

### 使用 Hardhat Network 辅助调试

```javascript
const { ethers } = require("hardhat");

it("Debug transaction", async function () {
  const tx = await contract.someFunction();
  const receipt = await tx.wait();
  
  // 查看事件
  console.log("Events:", receipt.logs);
  
  // 查看详细信息
  console.log("Transaction:", {
    hash: tx.hash,
    from: tx.from,
    to: tx.to,
    gasLimit: tx.gasLimit.toString(),
    data: tx.data
  });
  
  // 查看内部调用
  await network.provider.send("hardhat_setLoggingEnabled", [true]);
});
```

### Stack Trace

Hardhat 提供详细的错误堆栈跟踪：

```javascript
// 测试失败时会显示详细的错误信息
await expect(contract.transfer(addr1, 1000))
  .to.be.revertedWith("Insufficient balance");

// 输出示例：
// ContractSimpleToken
//   transfer
//     ✕ Should fail when insufficient balance
//   Error: VM Exception while processing transaction: reverted with reason string 'Insufficient balance'
//     at SimpleToken.transfer (contracts/SimpleToken.sol:45)
//     at processTicksAndRejections (internal/process/task_queues.js:95:5)
```

### 调试存储槽

```javascript
const { ethers } = require("hardhat");

// 读取存储槽
const slot0 = await ethers.provider.getStorage(contractAddress, 0);
console.log("Slot 0:", slot0);

// 读取映射
const balanceSlot = ethers.solidityPackedKeccak256(
  ["uint256", "uint256"],
  [accountAddress, 1]  // balanceOf 在 slot 1
);
const balance = await ethers.provider.getStorage(contractAddress, balanceSlot);
console.log("Balance:", ethers.getBigInt(balance));
```

## 完整示例项目

### 项目结构

```
my-hardhat-project/
├── contracts/
│   ├── SimpleToken.sol
│   └── interfaces/
│       └── IERC20.sol
├── scripts/
│   ├── deploy.js
│   └── interact.js
├── test/
│   ├── SimpleToken.js
│   └── utils.js
├── deployments/
│   └── sepolia/
├── hardhat.config.js
├── package.json
├── .env
└── .gitignore
```

### 交互脚本

```javascript
// scripts/interact.js
const hre = require("hardhat");

async function main() {
  const contractAddress = "0x...";  // 已部署的合约地址
  
  const Token = await hre.ethers.getContractFactory("SimpleToken");
  const token = Token.attach(contractAddress);

  // 读取数据
  const name = await token.name();
  const symbol = await token.symbol();
  const totalSupply = await token.totalSupply();
  
  console.log("Token Info:");
  console.log(`  Name: ${name}`);
  console.log(`  Symbol: ${symbol}`);
  console.log(`  Total Supply: ${hre.ethers.formatUnits(totalSupply, 18)}`);

  // 调用函数
  const [signer] = await hre.ethers.getSigners();
  const balance = await token.balanceOf(signer.address);
  console.log(`  My Balance: ${hre.ethers.formatUnits(balance, 18)}`);

  // 执行交易
  const recipient = "0x...";
  const amount = hre.ethers.parseUnits("100", 18);
  
  const tx = await token.transfer(recipient, amount);
  console.log("Transaction hash:", tx.hash);
  
  const receipt = await tx.wait();
  console.log("Transaction confirmed in block:", receipt.blockNumber);
}

main().catch(console.error);
```

## package.json 脚本配置

```json
{
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "test:coverage": "hardhat coverage",
    "node": "hardhat node",
    "deploy:local": "hardhat run scripts/deploy.js --network localhost",
    "deploy:sepolia": "hardhat run scripts/deploy.js --network sepolia",
    "deploy:mainnet": "hardhat run scripts/deploy.js --network mainnet",
    "verify": "hardhat verify --network",
    "clean": "hardhat clean",
    "format": "prettier --write .",
    "lint": "solhint 'contracts/**/*.sol'"
  }
}
```

## 最佳实践

1. **使用 TypeScript**：类型安全，IDE 支持更好
2. **分离测试文件**：每个合约一个测试文件
3. **使用 Fixture**：提高测试速度
4. **测试覆盖率**：保持 90% 以上
5. **Gas 优化**：使用 gas reporter 监控
6. **环境隔离**：不同网络使用不同配置
7. **私钥管理**：永远不要硬编码私钥
8. **版本控制**：锁定依赖版本

## 下一步

现在你已经掌握了 Hardhat 的核心开发流程，继续学习 [Foundry 环境搭建](./03-foundry-setup.md)，体验 Rust 带来的极速开发体验。
