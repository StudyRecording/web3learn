# 2.4 以太坊网络

## 网络类型

```
┌─────────────────────────────────────────────────────────────┐
│                   以太坊网络类型                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  主网 (Mainnet)                                             │
│  ├── 真实资产                                               │
│  ├── Chain ID: 1                                            │
│  └── 生产环境                                               │
│                                                             │
│  测试网 (Testnet)                                           │
│  ├── 测试资产 (无价值)                                      │
│  ├── 开发测试用                                             │
│  └── 免费 ETH                                               │
│                                                             │
│  本地开发链                                                  │
│  ├── Hardhat Network                                        │
│  ├── Anvil (Foundry)                                        │
│  └── Ganache                                                │
│                                                             │
│  Layer 2 网络                                               │
│  ├── Arbitrum                                               │
│  ├── Optimism                                               │
│  ├── Base                                                   │
│  └── zkSync                                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 主网 (Mainnet)

```
以太坊主网

基本信息:
├── Chain ID: 1
├── 代币: ETH
├── 区块时间: ~12 秒
├── 共识: PoS (权益证明)
└── 状态: 生产环境

区块浏览器:
└── https://etherscan.io

RPC 端点:
├── https://mainnet.infura.io/v3/YOUR_KEY
├── https://eth-mainnet.alchemyapi.io/v2/YOUR_KEY
└── https://eth.llamarpc.com
```

## 测试网 (Testnet)

### Sepolia (推荐)

```
Sepolia 测试网

基本信息:
├── Chain ID: 11155111
├── 代币: SepoliaETH (无价值)
├── 区块时间: ~12 秒
├── 共识: PoS
└── 状态: 活跃，推荐使用

获取测试 ETH:
├── https://sepoliafaucet.com
├── https://www.alchemy.com/faucets/ethereum-sepolia
└── https://faucet.quicknode.com/ethereum/sepolia

区块浏览器:
└── https://sepolia.etherscan.io

RPC 端点:
├── https://sepolia.infura.io/v3/YOUR_KEY
└── https://eth-sepolia.alchemyapi.io/v2/YOUR_KEY
```

### 其他测试网

| 测试网 | Chain ID | 状态 | 用途 |
|--------|----------|------|------|
| Sepolia | 11155111 | 活跃 | 推荐，PoS |
| Holesky | 17000 | 活跃 | 验证者测试 |
| Goerli | 5 | 弃用中 | 旧测试网 |

## 本地开发链

### Hardhat Network

```javascript
// hardhat.config.js
module.exports = {
  solidity: "0.8.20",
  networks: {
    hardhat: {
      // 本地开发链配置
      chainId: 31337,
      // 自动挖矿模式
      mining: {
        auto: true,
        interval: 0
      },
      // 初始账户
      accounts: {
        count: 20,
        // 每个账户 10000 ETH
        defaultBalance: 10000
      }
    }
  }
};

// 启动本地节点
// npx hardhat node
```

### Anvil (Foundry)

```bash
# 安装 Foundry
curl -L https://foundry.paradigm.xyz | bash
foundryup

# 启动 Anvil
anvil

# 自定义配置
anvil --chain-id 31337 --port 8545 --accounts 10 --balance 10000

# 输出示例:
# Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
# Private Key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

## Layer 2 网络

### 为什么需要 Layer 2

```
以太坊扩展性问题:

Layer 1 限制:
├── TPS: ~15-30 笔/秒
├── Gas 费用: 高峰期可达 $50+
└── 用户体验: 差

Layer 2 解决方案:
├── TPS: 1000-10000+ 笔/秒
├── Gas 费用: 降低 90%+
├── 继承 L1 安全性
└── 更好的用户体验
```

### Arbitrum

```
Arbitrum (Offchain Labs)

类型: Optimistic Rollup
特点:
├── EVM 兼容
├── 低 Gas 费用
├── 安全性继承以太坊
└── 成熟的生态系统

网络信息:
├── 主网 Chain ID: 42161
├── 测试网 (Sepolia) Chain ID: 421614
└── 区块浏览器: https://arbiscan.io

RPC:
├── https://arb1.arbitrum.io/rpc
└── https://arb-mainnet.g.alchemy.com/v2/YOUR_KEY
```

### Optimism

```
Optimism

类型: Optimistic Rollup
特点:
├── EVM 等效
├── 低 Gas 费用
├── OP Stack (可定制)
└── 生态系统增长快

网络信息:
├── 主网 Chain ID: 10
├── 测试网 (Sepolia) Chain ID: 11155420
└── 区块浏览器: https://optimistic.etherscan.io

RPC:
├── https://mainnet.optimism.io
└── https://opt-mainnet.g.alchemy.com/v2/YOUR_KEY
```

### Base

```
Base (Coinbase)

类型: Optimistic Rollup (OP Stack)
特点:
├── Coinbase 支持
├── 低费用
├── EVM 兼容
└── 快速增长

网络信息:
├── 主网 Chain ID: 8453
├── 测试网 Chain ID: 84532
└── 区块浏览器: https://basescan.org

RPC:
└── https://mainnet.base.org
```

### Polygon

```
Polygon PoS

类型: 侧链 (有独立共识)
特点:
├── EVM 兼容
├── 极低费用
├── 快速确认
└── 独立验证者集

网络信息:
├── 主网 Chain ID: 137
├── 测试网 (Mumbai) Chain ID: 80001
└── 区块浏览器: https://polygonscan.com

RPC:
├── https://polygon-rpc.com
└── https://polygon-mainnet.g.alchemy.com/v2/YOUR_KEY
```

### Layer 2 对比

| 特性 | Arbitrum | Optimism | Base | Polygon |
|------|----------|----------|------|---------|
| 类型 | Optimistic Rollup | Optimistic Rollup | Optimistic Rollup | 侧链 |
| TPS | 4000+ | 2000+ | 2000+ | 7000+ |
| 费用 | 低 | 低 | 很低 | 极低 |
| 安全性 | 高 | 高 | 高 | 中 |
| EVM 兼容 | 是 | 是 | 是 | 是 |

## 网络配置

### MetaMask 添加网络

```javascript
// 网络配置参数
const networks = {
  sepolia: {
    chainId: '0xaa36a7',  // 11155111 十六进制
    chainName: 'Sepolia',
    nativeCurrency: {
      name: 'SepoliaETH',
      symbol: 'ETH',
      decimals: 18
    },
    rpcUrls: ['https://sepolia.infura.io/v3/YOUR_KEY'],
    blockExplorerUrls: ['https://sepolia.etherscan.io']
  },
  
  arbitrum: {
    chainId: '0xa4b1',  // 42161
    chainName: 'Arbitrum One',
    nativeCurrency: {
      name: 'Ether',
      symbol: 'ETH',
      decimals: 18
    },
    rpcUrls: ['https://arb1.arbitrum.io/rpc'],
    blockExplorerUrls: ['https://arbiscan.io']
  }
};

// 添加网络
async function addNetwork(network) {
  await window.ethereum.request({
    method: 'wallet_addEthereumChain',
    params: [networks[network]]
  });
}
```

### 切换网络

```javascript
// 切换到指定网络
async function switchNetwork(chainId) {
  try {
    await window.ethereum.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: chainId }]
    });
  } catch (switchError) {
    // 如果网络未添加，添加它
    if (switchError.code === 4902) {
      await addNetwork(chainId);
    }
  }
}

// 使用示例
await switchNetwork('0xaa36a7');  // Sepolia
await switchNetwork('0x1');       // Mainnet
await switchNetwork('0xa4b1');    // Arbitrum
```

## 跨链桥

### 什么是跨链桥

```
跨链桥 (Bridge)

功能: 在不同链之间转移资产

工作原理:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  以太坊主网                     Arbitrum                    │
│  ┌─────────┐                   ┌─────────┐                 │
│  │ 100 ETH │──锁定──> 桥 <──铸造──│ 100 ETH│                 │
│  │         │           │           │(L2 ETH)│               │
│  └─────────┘           └───────────└─────────┘               │
│                                                             │
│  从 L1 到 L2: 锁定 L1 资产，在 L2 铸造等量资产              │
│  从 L2 到 L1: 销毁 L2 资产，在 L1 释放锁定资产              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 常用跨链桥

```
官方桥:
├── Arbitrum Bridge: https://bridge.arbitrum.io
├── Optimism Bridge: https://app.optimism.io/bridge
└── Base Bridge: https://bridge.base.org

第三方桥:
├── LayerZero (Stargate)
├── Multichain
├── Hop Protocol
└── Across Protocol

注意:
├── 官方桥安全性最高
├── 第三方桥速度快但风险较高
└── 大额资产使用官方桥
```

## 开发流程

```
推荐开发流程

1. 本地开发
   └── 使用 Hardhat/Anvil 本地链

2. 单元测试
   └── 本地环境测试

3. 测试网部署
   └── 部署到 Sepolia
   └── 前端集成测试

4. 审计
   └── 安全审计

5. 主网部署
   └── 先部署合约
   └── 验证合约代码
   └── 前端上线

6. Layer 2 部署
   └── 选择合适的 L2
   └── 部署和验证
```

## 下一步

[下一节：以太坊生态系统 →](05-ecosystem.md)
