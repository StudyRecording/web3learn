# 1.4 Web3 技术栈

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Web3 技术栈全景                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      应用层 (Application)                     │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │  DeFi   │ │   NFT   │ │   DAO   │ │  GameFi │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      前端层 (Frontend)                        │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │ React   │ │ ethers.js│ │  wagmi  │ │ RainbowKit│          │   │
│  │  │  Vue    │ │ web3.js │ │viem     │ │ ConnectKit│           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      SDK/框架层 (Frameworks)                  │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │ Hardhat │ │ Foundry │ │ Scaffold│ │thirdweb │           │   │
│  │  │ Truffle │ │ Brownie │ │  -ETH   │ │         │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      智能合约层 (Contracts)                   │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │Solidity │ │  Vyper  │ │  Rust   │ │ Move    │           │   │
│  │  │ (EVM)   │ │  (EVM)  │ │(Solana) │ │(Aptos)  │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      节点/索引层 (Infrastructure)             │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │ Infura  │ │ Alchemy │ │The Graph│ │ Moralis │           │   │
│  │  │ QuickNode│ │ ankr   │ │(索引)   │ │(BaaS)   │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      区块链层 (Blockchain)                    │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │Ethereum │ │ Solana  │ │ Polygon │ │Arbitrum │           │   │
│  │  │  (L1)   │ │  (L1)   │ │  (L2)   │ │  (L2)   │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      存储层 (Storage)                         │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │  IPFS   │ │ Arweave │ │Filecoin │ │   Swarm │           │   │
│  │  │(分布式) │ │(永久)   │ │(激励)   │ │(以太坊) │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 智能合约开发语言

### Solidity (以太坊/EVM)

```solidity
// Solidity - 最流行的智能合约语言

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleStorage {
    uint256 private value;
    
    event ValueChanged(uint256 newValue);
    
    function setValue(uint256 _value) public {
        value = _value;
        emit ValueChanged(_value);
    }
    
    function getValue() public view returns (uint256) {
        return value;
    }
}

// 特点:
// - 类似 JavaScript 的语法
// - 静态类型
// - 支持 inheritance, libraries
// - EVM 专用
```

### Rust (Solana/Substrate)

```rust
// Rust - 用于 Solana 和 Substrate 开发

// Solana 程序示例
use solana_program::{
    account_info::AccountInfo,
    entrypoint,
    msg,
    pubkey::Pubkey,
};

entrypoint!(process_instruction);

pub fn process_instruction(
    _program_id: &Pubkey,
    _accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> entrypoint::ProgramResult {
    msg!("Hello, Solana!");
    Ok(())
}

// 特点:
// - 内存安全
// - 高性能
// - Solana/Substrate 专用
// - 学习曲线较陡
```

### Vyper (以太坊/EVM)

```python
# Vyper - Python 风格的智能合约语言

# @version ^0.3.0

value: uint256

@external
def setValue(_value: uint256):
    self.value = _value

@external
@view
def getValue() -> uint256:
    return self.value

# 特点:
# - Python 语法
# - 更安全（限制性更强）
# - 无限循环、递归等危险特性
// - 适合安全关键的合约
```

## 开发框架

### Hardhat (JavaScript/TypeScript)

```javascript
// Hardhat - 最流行的以太坊开发框架

// hardhat.config.js
module.exports = {
  solidity: "0.8.20",
  networks: {
    sepolia: {
      url: process.env.SEPOLIA_RPC_URL,
      accounts: [process.env.PRIVATE_KEY]
    }
  }
};

// 测试文件
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleStorage", function () {
  it("Should store and retrieve value", async function () {
    const Storage = await ethers.getContractFactory("SimpleStorage");
    const storage = await Storage.deploy();
    
    await storage.setValue(42);
    expect(await storage.getValue()).to.equal(42);
  });
});
```

### Foundry (Rust)

```rust
// Foundry - 用 Rust 编写，速度极快

// src/SimpleStorage.sol
contract SimpleStorage {
    uint256 public value;
    function setValue(uint256 _value) external {
        value = _value;
    }
}

// test/SimpleStorage.t.sol
contract SimpleStorageTest is Test {
    SimpleStorage storage;
    
    function setUp() public {
        storage = new SimpleStorage();
    }
    
    function testSetValue() public {
        storage.setValue(42);
        assertEq(storage.value(), 42);
    }
}

// 特点:
// - 编译、测试速度快 10-100 倍
// - Solidity 编写测试
// - 内置模糊测试
// - 现在开发者的首选
```

## 前端库

### ethers.js

```javascript
// ethers.js - 以太坊交互库

import { ethers } from 'ethers';

// 连接钱包
const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
const address = await signer.getAddress();

// 连接合约
const contractAddress = '0x...';
const abi = [...];
const contract = new ethers.Contract(contractAddress, abi, signer);

// 读取数据
const value = await contract.getValue();

// 写入数据（需要签名）
const tx = await contract.setValue(42);
await tx.wait(); // 等待确认

// 监听事件
contract.on('ValueChanged', (newValue) => {
  console.log('New value:', newValue);
});
```

### wagmi + viem (现代推荐)

```javascript
// wagmi - React Hooks for Ethereum
// viem - 轻量级以太坊接口

import { useAccount, useConnect, useDisconnect } from 'wagmi';
import { useContractRead, useContractWrite } from 'wagmi';

function App() {
  // 钱包连接
  const { address, isConnected } = useAccount();
  const { connect, connectors } = useConnect();
  const { disconnect } = useDisconnect();
  
  // 读取合约
  const { data: value } = useContractRead({
    address: '0x...',
    abi: contractAbi,
    functionName: 'getValue',
  });
  
  // 写入合约
  const { write } = useContractWrite({
    address: '0x...',
    abi: contractAbi,
    functionName: 'setValue',
  });
  
  return (
    <div>
      {isConnected ? (
        <button onClick={() => disconnect()}>
          Disconnect {address}
        </button>
      ) : (
        connectors.map((connector) => (
          <button key={connector.id} onClick={() => connect({ connector })}>
            Connect {connector.name}
          </button>
        ))
      )}
    </div>
  );
}
```

## 区块链节点服务

### 为什么需要节点服务

```
直接运行节点的问题:
├── 需要大量存储空间 (以太坊全节点 > 1TB)
├── 同步时间长 (数天到数周)
├── 需要稳定带宽
└── 维护成本高

解决方案: 使用节点服务提供商
```

### 主要提供商

```
节点服务提供商

1. Infura
   ├── 最老牌
   ├── 免费额度充足
   └── 支持 Ethereum, IPFS

2. Alchemy
   ├── 开发者工具丰富
   ├── 免费额度大
   └── 支持多链

3. QuickNode
   ├── 高性能
   ├── 企业级
   └── 多链支持

4. Ankr
   ├── 去中心化
   ├── 支持多链
   └── 价格合理
```

## 数据索引

### 为什么需要索引

```
区块链数据查询的问题:

问题: 获取用户的所有交易
├── 需要扫描所有区块
├── 耗时极长
└── 前端无法直接使用

解决方案: 使用索引服务
├── 预先索引链上数据
├── 提供 GraphQL API
└── 快速查询
```

### The Graph

```graphql
# The Graph - 去中心化索引协议

# 定义子图 (Subgraph)
# subgraph.yaml
dataSources:
  - kind: ethereum
    name: UniswapV3Pool
    network: mainnet
    source:
      address: "0x..."
      abi: Pool
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Swap
      abis:
        - name: Pool
          file: ./abis/Pool.json
      eventHandlers:
        - event: Swap(...)
          handler: handleSwap

# 查询
query {
  swaps(
    where: { origin: "0xuserAddress" }
    orderBy: timestamp
    orderDirection: desc
    first: 10
  ) {
    id
    amountIn
    amountOut
    timestamp
  }
}
```

## 去中心化存储

### 为什么需要去中心化存储

```
链上存储的问题:
├── 成本极高 (以太坊: ~$10,000/MB)
├── 不适合存储大文件
└── 只适合存储关键数据

解决方案: 去中心化存储
├── 存储哈希在链上
├── 实际文件在去中心化网络
└── 成本低，可靠性高
```

### IPFS

```
IPFS (InterPlanetary File System)

工作原理:
┌─────────────────────────────────────────┐
│                                         │
│  文件 ──分块──> 多个数据块              │
│            │                            │
│            ▼                            │
│         计算哈希                         │
│            │                            │
│            ▼                            │
│    分布式网络存储                        │
│            │                            │
│            ▼                            │
│  通过哈希获取文件                        │
│                                         │
└─────────────────────────────────────────┘

使用示例:
const { create } = require('ipfs-http-client');
const ipfs = create();

// 上传文件
const result = await ipfs.add(file);
console.log(result.cid.toString()); // QmXxx...

// 下载文件
const chunks = [];
for await (const chunk of ipfs.cat(cid)) {
  chunks.push(chunk);
}
```

### Arweave

```
Arweave - 永久存储

特点:
├── 一次付费，永久存储
├── 适合 NFT 元数据
├── 内容可验证
└── 抗审查

使用场景:
├── NFT 元数据和图片
├── 历史数据存档
└── 去中心化网站
```

## 测试网络

### 为什么需要测试网

```
开发流程:
├── 本地开发 ──> 本地测试链 (Hardhat/Anvil)
├── 测试网部署 ──> 公共测试网 (Sepolia)
└── 主网部署 ──> 以太坊主网

测试网特点:
├── 代币无真实价值
├── 可免费获取
├── 与主网功能相同
└── 用于开发和测试
```

### 主要测试网

| 测试网 | 说明 | 获取测试币 |
|--------|------|-----------|
| Sepolia | 以太坊官方测试网 | https://sepoliafaucet.com |
| Goerli | 即将弃用 | - |
| Mumbai | Polygon 测试网 | https://faucet.polygon.technology |
| Fuji | Avalanche 测试网 | https://faucet.avax.network |

## 开发工具汇总

```
┌─────────────────────────────────────────────────────────────────┐
│                     Web3 开发工具箱                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  钱包                                                           │
│  ├── MetaMask (浏览器插件)                                      │
│  ├── Rabby (更安全的替代品)                                     │
│  └── Ledger (硬件钱包)                                          │
│                                                                 │
│  IDE                                                            │
│  ├── Remix (在线 IDE)                                           │
│  ├── VS Code + Solidity 插件                                    │
│  └── Rust Analyzer (Rust 开发)                                  │
│                                                                 │
│  区块链浏览器                                                    │
│  ├── Etherscan (以太坊)                                         │
│  ├── Polygonscan (Polygon)                                      │
│  └── Solscan (Solana)                                           │
│                                                                 │
│  测试/安全                                                      │
│  ├── Foundry (模糊测试)                                         │
│  ├── Slither (静态分析)                                         │
│  └── Mythril (符号执行)                                         │
│                                                                 │
│  监控/调试                                                      │
│  ├── Tenderly (交易模拟)                                        │
│  └── OpenZeppelin Defender                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 下一步

[下一节：Web2 vs Web3 开发对比 →](05-web2-vs-web3.md)
