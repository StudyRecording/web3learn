# 部署流程和脚本

本章介绍智能合约的部署流程，包括本地测试、测试网部署、主网部署以及验证合约。

## 部署概述

智能合约部署是一个不可逆的过程，需要谨慎操作：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   本地开发    │ ──→ │   测试网     │ ──→ │    主网      │
│  (Anvil/     │     │  (Sepolia/   │     │  (Ethereum/  │
│   Hardhat)   │     │   Goerli)    │     │   L2s)       │
└──────────────┘     └──────────────┘     └──────────────┘
      ↓                     ↓                    ↓
  快速迭代             团队测试             生产环境
  调试验证             社区测试             真实资金
```

## 部署前检查清单

```markdown
## 技术检查
- [ ] 所有测试通过
- [ ] 测试覆盖率 > 90%
- [ ] 无编译警告
- [ ] Gas 优化完成
- [ ] 合约大小 < 24KB

## 安全检查
- [ ] 代码审计（内部/外部）
- [ ] 无已知漏洞
- [ ] 权限设置正确
- [ ] 管理员密钥安全存储

## 运营准备
- [ ] 多签钱包配置完成
- [ ] 紧急暂停机制就绪
- [ ] 升级路径规划（如需要）
- [ ] 监控和告警配置
```

## Hardhat 部署

### 基础部署脚本

```javascript
// scripts/deploy.js
const hre = require("hardhat");

async function main() {
  const [deployer] = await hre.ethers.getSigners();
  
  console.log("部署账户:", deployer.address);
  console.log("账户余额:", hre.ethers.formatEther(
    await hre.ethers.provider.getBalance(deployer.address)
  ));

  // 部署合约
  const Token = await hre.ethers.getContractFactory("SimpleToken");
  console.log("正在部署 SimpleToken...");
  
  const token = await Token.deploy(
    "My Token",
    "MTK",
    hre.ethers.parseUnits("1000000", 18)
  );
  
  await token.waitForDeployment();
  const address = await token.getAddress();
  
  console.log("SimpleToken 部署到:", address);
  
  // 验证部署
  console.log("\n验证部署:");
  console.log("  名称:", await token.name());
  console.log("  符号:", await token.symbol());
  console.log("  总量:", hre.ethers.formatUnits(await token.totalSupply(), 18));
  
  return address;
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

### 使用 hardhat-deploy

```javascript
// deployments/sepolia/01-deploy-token.js
module.exports = async ({ getNamedAccounts, deployments }) => {
  const { deploy, execute, read } = deployments;
  const { deployer, treasury } = await getNamedAccounts();
  
  // 部署代币
  const token = await deploy("SimpleToken", {
    from: deployer,
    args: ["My Token", "MTK", "1000000"],
    log: true,
    waitConfirmations: 5,
  });
  
  console.log("Token 部署到:", token.address);
  
  // 部署交易所
  const exchange = await deploy("Exchange", {
    from: deployer,
    args: [token.address],
    log: true,
    waitConfirmations: 5,
    libraries: {
      Math: mathLib.address,
    },
  });
  
  console.log("Exchange 部署到:", exchange.address);
  
  // 执行初始化操作
  await execute(
    "SimpleToken",
    { from: deployer },
    "transfer",
    treasury,
    ethers.parseUnits("100000", 18)
  );
  
  // 验证
  const balance = await read("SimpleToken", "balanceOf", treasury);
  console.log("Treasury 余额:", ethers.formatUnits(balance, 18));
};

module.exports.tags = ["all", "token"];
module.exports.dependencies = ["math"];  // 依赖其他部署
```

### 多网络部署

```javascript
// scripts/deploy-all.js
const hre = require("hardhat");

async function main() {
  const network = hre.network.name;
  console.log(`正在部署到 ${network}...`);
  
  // 网络特定配置
  const configs = {
    localhost: {
      confirmations: 1,
      verify: false,
    },
    sepolia: {
      confirmations: 5,
      verify: true,
    },
    mainnet: {
      confirmations: 10,
      verify: true,
    },
  };
  
  const config = configs[network] || configs.localhost;
  
  // 部署逻辑
  const contracts = await deployContracts(config);
  
  // 保存部署信息
  await saveDeploymentInfo(network, contracts);
  
  // 验证合约
  if (config.verify) {
    await verifyContracts(contracts);
  }
}

async function deployContracts(config) {
  const contracts = {};
  
  // 部署代币
  const Token = await hre.ethers.getContractFactory("SimpleToken");
  contracts.token = await Token.deploy("My Token", "MTK", 1000000);
  await contracts.token.waitForDeployment(config.confirmations);
  
  // 部署其他合约...
  
  return contracts;
}

async function saveDeploymentInfo(network, contracts) {
  const fs = require("fs");
  const path = require("path");
  
  const deployInfo = {
    network,
    timestamp: new Date().toISOString(),
    contracts: {},
  };
  
  for (const [name, contract] of Object.entries(contracts)) {
    deployInfo.contracts[name] = {
      address: await contract.getAddress(),
      abi: contract.interface.formatJson(),
    };
  }
  
  const dir = path.join(__dirname, "..", "deployments", network);
  if (!fs.existsSync(dir)) {
    fs.mkdirSync(dir, { recursive: true });
  }
  
  fs.writeFileSync(
    path.join(dir, "deployment.json"),
    JSON.stringify(deployInfo, null, 2)
  );
}

async function verifyContracts(contracts) {
  console.log("\n验证合约...");
  
  for (const [name, contract] of Object.entries(contracts)) {
    console.log(`验证 ${name}...`);
    try {
      await hre.run("verify:verify", {
        address: await contract.getAddress(),
        constructorArguments: contract.constructorArgs || [],
      });
    } catch (e) {
      console.log(`验证失败 (${name}):`, e.message);
    }
  }
}

main().catch(console.error);
```

### 部署命令

```bash
# 本地部署
bunx hardhat run scripts/deploy.js

# 部署到本地节点
bunx hardhat node  # 终端 1
bunx hardhat run scripts/deploy.js --network localhost  # 终端 2

# 部署到测试网
bunx hardhat run scripts/deploy.js --network sepolia

# 部署到主网
bunx hardhat run scripts/deploy.js --network mainnet

# 验证合约
bunx hardhat verify --network sepolia <ADDRESS> <CONSTRUCTOR_ARGS>
```

## Foundry 部署

### 基础部署脚本

```solidity
// script/Deploy.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/SimpleToken.sol";

contract DeployScript is Script {
    function run() external returns (SimpleToken) {
        // 从环境变量读取私钥
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        // 开始广播交易
        vm.startBroadcast(deployerPrivateKey);
        
        SimpleToken token = new SimpleToken(
            "My Token",
            "MTK",
            1_000_000 * 10 ** 18
        );
        
        vm.stopBroadcast();
        
        console.log("Token deployed at:", address(token));
        
        return token;
    }
}
```

### 复杂部署脚本

```solidity
// script/DeployDeFi.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/Token.sol";
import "../src/Vault.sol";
import "../src/Router.sol";

contract DeployDeFi is Script {
    // 配置结构
    struct Config {
        address admin;
        address treasury;
        uint256 initialSupply;
    }
    
    function run() external {
        // 读取配置
        Config memory config = readConfig();
        
        // 获取部署者
        uint256 privateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(privateKey);
        
        console.log("Deployer:", deployer);
        console.log("Balance:", deployer.balance);
        
        vm.startBroadcast(privateKey);
        
        // 部署代币
        Token token = deployToken(config);
        
        // 部署金库
        Vault vault = deployVault(token, config);
        
        // 部署路由
        Router router = deployRouter(token, vault);
        
        // 配置权限
        configurePermissions(token, vault, router, config);
        
        vm.stopBroadcast();
        
        // 输出部署摘要
        printSummary(token, vault, router);
    }
    
    function readConfig() internal view returns (Config memory) {
        return Config({
            admin: vm.envAddress("ADMIN_ADDRESS"),
            treasury: vm.envAddress("TREASURY_ADDRESS"),
            initialSupply: vm.envUint("INITIAL_SUPPLY")
        });
    }
    
    function deployToken(Config memory config) internal returns (Token) {
        Token token = new Token(
            "DeFi Token",
            "DFT",
            config.initialSupply
        );
        console.log("Token:", address(token));
        return token;
    }
    
    function deployVault(
        Token token,
        Config memory config
    ) internal returns (Vault) {
        Vault vault = new Vault(
            address(token),
            config.treasury
        );
        console.log("Vault:", address(vault));
        return vault;
    }
    
    function deployRouter(
        Token token,
        Vault vault
    ) internal returns (Router) {
        Router router = new Router(
            address(token),
            address(vault)
        );
        console.log("Router:", address(router));
        return router;
    }
    
    function configurePermissions(
        Token token,
        Vault vault,
        Router router,
        Config memory config
    ) internal {
        // 转移管理员权限
        token.transferOwnership(config.admin);
        vault.transferOwnership(config.admin);
        router.transferOwnership(config.admin);
        
        // 设置 minter
        token.setMinter(address(vault), true);
        
        // 转移代币到金库
        token.transfer(address(vault), config.initialSupply / 10);
    }
    
    function printSummary(
        Token token,
        Vault vault,
        Router router
    ) internal view {
        console.log("\n=== Deployment Summary ===");
        console.log("Chain ID:", block.chainid);
        console.log("Token:", address(token));
        console.log("Vault:", address(vault));
        console.log("Router:", address(router));
        console.log("=========================\n");
    }
}
```

### 部署命令

```bash
# 模拟部署（不发送交易）
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# 部署到测试网
forge script script/Deploy.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --private-key $PRIVATE_KEY \
    --broadcast \
    --verify \
    --etherscan-api-key $ETHERSCAN_API_KEY

# 指定链 ID
forge script script/Deploy.s.sol \
    --rpc-url $RPC_URL \
    --private-key $PRIVATE_KEY \
    --chain-id 11155111 \
    --broadcast

# 使用 keystore 文件
forge script script/Deploy.s.sol \
    --rpc-url $RPC_URL \
    --account keystore.json \
    --sender 0x... \
    --broadcast

# 使用硬件钱包
forge script script/Deploy.s.sol \
    --rpc-url $RPC_URL \
    --ledger \
    --sender 0x... \
    --broadcast

# 估算 gas
forge script script/Deploy.s.sol \
    --rpc-url $RPC_URL \
    --private-key $PRIVATE_KEY
```

### 部署记录

Foundry 在 `broadcast/` 目录保存部署记录：

```bash
broadcast/
└── Deploy.s.sol/
    └── 11155111/
        ├── run-1700000000.json
        └── run-latest.json
```

记录内容：

```json
{
  "transactions": [
    {
      "hash": "0x...",
      "contractAddress": "0x...",
      "transactionType": "CREATE",
      "contractName": "SimpleToken",
      "args": ["My Token", "MTK", "1000000000000000000000000"]
    }
  ],
  "timestamp": 1700000000,
  "chain": 11155111
}
```

## 合约验证

### Etherscan 验证

```bash
# Hardhat
bunx hardhat verify --network sepolia <ADDRESS> "My Token" "MTK" "1000000"

# Foundry
forge verify-contract <ADDRESS> SimpleToken \
    --chain-id 11155111 \
    --watch \
    --constructor-args $(cast abi-encode "constructor(string,string,uint256)" "My Token" "MTK" 1000000)

# 使用 verify-check
forge verify-check <GUID> --chain-id 11155111
```

### 多链验证

```javascript
// scripts/verify.js
const hre = require("hardhat");

async function main() {
  const deployments = {
    ethereum: {
      address: "0x...",
      constructorArgs: ["My Token", "MTK", 1000000],
    },
    polygon: {
      address: "0x...",
      constructorArgs: ["My Token", "MTK", 1000000],
    },
    arbitrum: {
      address: "0x...",
      constructorArgs: ["My Token", "MTK", 1000000],
    },
  };
  
  for (const [network, deployment] of Object.entries(deployments)) {
    console.log(`验证 ${network}...`);
    try {
      await hre.run("verify:verify", {
        address: deployment.address,
        constructorArguments: deployment.constructorArgs,
      });
      console.log(`${network} 验证成功`);
    } catch (e) {
      console.log(`${network} 验证失败:`, e.message);
    }
  }
}

main().catch(console.error);
```

## 部署后操作

### 转移所有权

```javascript
// scripts/transfer-ownership.js
async function main() {
  const token = await ethers.getContractAt("SimpleToken", TOKEN_ADDRESS);
  const multisig = "0x...";  // 多签钱包地址
  
  // 转移所有权
  const tx = await token.transferOwnership(multisig);
  await tx.wait();
  
  console.log("所有权已转移到:", multisig);
  
  // 验证
  const owner = await token.owner();
  console.log("当前所有者:", owner);
}
```

### 初始化设置

```solidity
// script/Initialize.s.sol
contract InitializeScript is Script {
    function run() external {
        address token = vm.envAddress("TOKEN_ADDRESS");
        address treasury = vm.envAddress("TREASURY_ADDRESS");
        
        vm.startBroadcast();
        
        // 添加 minter
        Token(token).setMinter(treasury, true);
        
        // 设置费用
        Token(token).setFeeRate(100);  // 1%
        
        vm.stopBroadcast();
    }
}
```

### 验证部署

```javascript
// scripts/verify-deployment.js
async function main() {
  const token = await ethers.getContractAt("SimpleToken", TOKEN_ADDRESS);
  
  console.log("验证部署:");
  console.log("  名称:", await token.name());
  console.log("  符号:", await token.symbol());
  console.log("  总量:", ethers.formatUnits(await token.totalSupply(), 18));
  console.log("  所有者:", await token.owner());
  
  // 验证代码是否验证
  const code = await ethers.provider.getCode(TOKEN_ADDRESS);
  console.log("  代码大小:", code.length, "bytes");
  
  if (code === "0x") {
    throw new Error("合约未部署!");
  }
}
```

## 部署到 L2

### Arbitrum 部署

```javascript
// hardhat.config.js
networks: {
  arbitrum: {
    url: process.env.ARBITRUM_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
    gasPrice: 100000000,  // 0.1 Gwei
  },
  arbitrumGoerli: {
    url: process.env.ARBITRUM_GOERLI_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
  },
}
```

```bash
bunx hardhat run scripts/deploy.js --network arbitrum
```

### Optimism 部署

```javascript
networks: {
  optimism: {
    url: process.env.OPTIMISM_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
  },
  optimismGoerli: {
    url: process.env.OPTIMISM_GOERLI_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
  },
}
```

### Polygon 部署

```javascript
networks: {
  polygon: {
    url: process.env.POLYGON_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
    gasPrice: 30000000000,  // 30 Gwei
  },
  polygonMumbai: {
    url: process.env.MUMBAI_RPC_URL,
    accounts: [process.env.PRIVATE_KEY],
  },
}
```

## 生产部署流程

### 完整部署流程

```bash
# 1. 准备环境
cp .env.example .env
# 编辑 .env 文件，填入正确的配置

# 2. 运行所有测试
forge test -vvv

# 3. 检查合约大小
forge build --sizes

# 4. 估算部署成本
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# 5. 部署到测试网
forge script script/Deploy.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --private-key $TEST_PRIVATE_KEY \
    --broadcast \
    --verify

# 6. 在测试网验证功能
# ...

# 7. 部署到主网
forge script script/Deploy.s.sol \
    --rpc-url $MAINNET_RPC_URL \
    --private-key $PRIVATE_KEY \
    --broadcast \
    --verify \
    --slow  # 减慢发送速度

# 8. 转移所有权到多签
forge script script/TransferOwnership.s.sol \
    --rpc-url $MAINNET_RPC_URL \
    --private-key $PRIVATE_KEY \
    --broadcast

# 9. 验证最终状态
cast call $TOKEN_ADDRESS "owner()" --rpc-url $MAINNET_RPC_URL
```

### 部署检查脚本

```solidity
// script/CheckDeployment.s.sol
contract CheckDeployment is Script {
    function run() external view {
        address token = vm.envAddress("TOKEN_ADDRESS");
        
        // 检查合约是否部署
        uint256 codeSize;
        assembly {
            codeSize := extcodesize(token)
        }
        require(codeSize > 0, "Contract not deployed");
        
        // 检查所有者
        address owner = Token(token).owner();
        console.log("Owner:", owner);
        
        // 检查总供应量
        uint256 supply = Token(token).totalSupply();
        console.log("Total supply:", supply);
        
        // 检查暂停状态
        bool paused = Token(token).paused();
        console.log("Paused:", paused);
        
        console.log("\nDeployment check passed!");
    }
}
```

## 最佳实践

1. **使用多签钱包**：所有管理员操作使用多签
2. **分阶段部署**：先部署核心合约，再部署周边合约
3. **时间锁**：敏感操作使用时间锁
4. **测试网先行**：在多个测试网充分测试
5. **审计**：主网部署前进行安全审计
6. **监控**：部署后持续监控合约状态
7. **应急预案**：准备好紧急暂停和升级方案
8. **文档**：记录所有部署参数和步骤

## 常见问题

### Gas 估算失败

```bash
# 手动指定 gas limit
forge script script/Deploy.s.sol \
    --rpc-url $RPC_URL \
    --private-key $PRIVATE_KEY \
    --broadcast \
    --with-gas-price 20000000000 \
    --gas-limit 5000000
```

### Nonce 问题

```bash
# 指定 nonce
forge script script/Deploy.s.sol \
    --rpc-url $RPC_URL \
    --private-key $PRIVATE_KEY \
    --broadcast \
    --nonce 10
```

### 验证失败

```bash
# 强制验证
forge verify-contract <ADDRESS> SimpleToken \
    --chain-id 1 \
    --watch \
    --retries 10 \
    --delay 5
```

## 总结

部署是智能合约开发的最后一步，也是最关键的一步。遵循最佳实践，做好充分测试和准备，确保部署顺利完成。

恭喜你完成了开发框架章节的学习！你已经掌握了 Hardhat 和 Foundry 两大框架的使用方法，可以开始构建自己的 Web3 应用了。
