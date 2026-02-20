# 2.2 交易与 Gas

## 交易 (Transaction)

交易是以太坊状态改变的唯一方式。每个交易都会被矿工/验证者打包进区块。

### 交易结构

```
交易字段

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  from: address          发送者地址                          │
│  ├── 必须是 EOA                                            │
│  └── 由签名推导，不包含在原始数据中                         │
│                                                             │
│  to: address            接收者地址                          │
│  ├── EOA: 普通转账                                         │
│  ├── CA: 合约调用                                          │
│  └── null: 创建合约                                        │
│                                                             │
│  value: uint256         转账金额 (Wei)                      │
│  └── 可以为 0                                              │
│                                                             │
│  data: bytes            输入数据                            │
│  ├── 转账: 通常为空                                        │
│  ├── 合约调用: 函数签名 + 参数                             │
│  └── 合约创建: 合约字节码                                  │
│                                                             │
│  nonce: uint64          发送者交易计数                      │
│  ├── 防止重放攻击                                          │
│  └── 确保交易顺序                                          │
│                                                             │
│  gasLimit: uint64       Gas 上限                            │
│  └── 交易最大消耗的 Gas                                    │
│                                                             │
│  maxFeePerGas: uint256  每单位 Gas 最大费用                 │
│  maxPriorityFeePerGas: uint256  小费                       │
│  └── EIP-1559 费用机制                                     │
│                                                             │
│  chainId: uint64        链 ID                               │
│  ├── 防止跨链重放                                          │
│  └── 主网: 1, Goerli: 5, Sepolia: 11155111                │
│                                                             │
│  v, r, s: bytes         签名数据                            │
│  └── ECDSA 签名                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 交易类型

```
交易类型

1. 普通转账
   {
     from: "0xalice...",
     to: "0xbob...",
     value: 1000000000000000000,  // 1 ETH
     data: "0x"
   }

2. 合约调用
   {
     from: "0xalice...",
     to: "0xcontract...",
     value: 0,
     data: "0xa9059cbb0000..."  // 函数签名 + 参数
   }

3. 合约部署
   {
     from: "0xalice...",
     to: null,                  // 空
     value: 0,
     data: "0x6080604052..."    // 字节码
   }
```

### 交易生命周期

```
交易生命周期

1. 构造交易
   ┌──────────────────────────────────────┐
   │ 填写: to, value, data, gasLimit 等   │
   └─────────────────┬────────────────────┘
                     │
                     ▼
2. 签名交易
   ┌──────────────────────────────────────┐
   │ 用私钥签名                            │
   │ 生成 v, r, s                         │
   └─────────────────┬────────────────────┘
                     │
                     ▼
3. 广播交易
   ┌──────────────────────────────────────┐
   │ 发送到节点                            │
   │ 进入交易池 (mempool)                  │
   └─────────────────┬────────────────────┘
                     │
                     ▼
4. 等待打包
   ┌──────────────────────────────────────┐
   │ 交易池中的交易等待被选中              │
   │ Gas 费高的优先                        │
   └─────────────────┬────────────────────┘
                     │
                     ▼
5. 执行交易
   ┌──────────────────────────────────────┐
   │ 验证签名、余额、nonce                 │
   │ 执行交易逻辑                          │
   │ 更新状态                              │
   └─────────────────┬────────────────────┘
                     │
                     ▼
6. 确认交易
   ┌──────────────────────────────────────┐
   │ 交易被包含在区块中                    │
   │ 等待更多区块确认                      │
   └──────────────────────────────────────┘
```

### 交易状态

```
交易状态

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Pending    等待被打包                                      │
│     │                                                       │
│     ▼                                                       │
│  Included   已被打包进区块                                  │
│     │                                                       │
│     ├──> Success  执行成功                                  │
│     │    └── 状态已更新                                     │
│     │    └── Gas 费用已扣除                                 │
│     │                                                       │
│     └──> Failed    执行失败                                 │
│          └── 状态回滚                                       │
│          └── Gas 费用已扣除 (不退还)                        │
│                                                             │
│  Replaced   被替换 (同 nonce 更高 Gas)                      │
│     └── 原交易失效                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Gas 机制

### 什么是 Gas

```
Gas 是以太坊的计算单位

类比:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  汽车行驶:                                                   │
│  ├── 距离 = Gas 数量 (消耗的计算量)                          │
│  ├── 油价 = Gas Price (每单位 Gas 的价格)                    │
│  └── 总费用 = 距离 × 油价                                    │
│                                                             │
│  以太坊:                                                     │
│  ├── 计算 = Gas Used (消耗的计算量)                          │
│  ├── 价格 = Gas Price (每单位 Gas 的价格)                    │
│  └── 总费用 = Gas Used × Gas Price                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

为什么需要 Gas?
├── 防止无限循环
├── 激励矿工/验证者
├── 防止垃圾交易
└── 为计算定价
```

### Gas 单位

```
Gas 单位

1 Gwei = 10^9 Wei = 0.000000001 ETH
1 ETH  = 10^9 Gwei = 10^18 Wei

示例:
Gas Price = 20 Gwei
Gas Used  = 21,000 (普通转账)
交易费用 = 21,000 × 20 Gwei = 420,000 Gwei
         = 0.00042 ETH

如果 ETH = $2,000:
交易费用 = $0.84
```

### EIP-1559 费用模型

```
EIP-1559 (2021年8月实施)

旧模型:
total_fee = gas_used × gas_price


新模型:
                    base_fee (基础费用)
total_fee = gas_used × ──────────────────────
                    priority_fee (小费)

组成部分:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Base Fee (基础费用)                                        │
│  ├── 由网络自动调整                                         │
│  ├── 根据区块拥堵程度                                       │
│  ├── 被销毁 (不归矿工)                                      │
│  └── 目标: 每个区块 50% 满                                  │
│                                                             │
│  Priority Fee (小费)                                        │
│  ├── 用户设置                                               │
│  ├── 归验证者所有                                           │
│  └── 激励快速打包                                           │
│                                                             │
│  Max Fee Per Gas (最大费用)                                 │
│  ├── 用户愿意支付的最高单价                                 │
│  └── max_fee >= base_fee + priority_fee                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Base Fee 调整机制:
- 区块 > 50% 满: base_fee 上涨 12.5%
- 区块 < 50% 满: base_fee 下降 12.5%
- 最大变化: 每区块 ±12.5%
```

### Gas Limit vs Gas Used

```
Gas Limit vs Gas Used

Gas Limit: 用户设置的最大 Gas
Gas Used:  实际消耗的 Gas

示例:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  用户设置 Gas Limit = 100,000                               │
│  实际 Gas Used = 50,000                                     │
│                                                             │
│  结果:                                                       │
│  ├── 消耗 50,000 Gas                                        │
│  ├── 剩余 50,000 Gas 退还                                   │
│  └── 只扣实际消耗的费用                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Gas 不足:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  用户设置 Gas Limit = 50,000                                │
│  实际需要 Gas = 100,000                                     │
│                                                             │
│  结果:                                                       │
│  ├── 执行到 50,000 时停止                                   │
│  ├── 状态回滚                                               │
│  ├── 50,000 Gas 费用不退还                                  │
│  └── 报错: "out of gas"                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Gas 消耗示例

```
常见操作的 Gas 消耗

操作                          Gas (大约)
─────────────────────────────────────────
普通 ETH 转账                 21,000
ERC-20 代币转账               65,000
ERC-20 授权 (approve)         46,000
Uniswap 代币兑换              150,000
铸造 NFT                      100,000+
部署合约                      数百万

存储操作 Gas:
─────────────────────────────────────────
从存储读取                    2,100 (冷) / 100 (热)
写入存储 (从零到非零)         20,000
写入存储 (修改)               5,000
清除存储 (变为零)             退款 15,000

注意: 存储操作是最贵的！
```

## 交易实战

### 发送交易 (ethers.js)

```javascript
import { ethers } from 'ethers';

// 连接钱包
const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

// 1. 普通转账
const tx1 = await signer.sendTransaction({
  to: '0xRecipientAddress...',
  value: ethers.parseEther('0.1'),  // 0.1 ETH
});

console.log('交易哈希:', tx1.hash);
await tx1.wait();  // 等待确认
console.log('交易确认');

// 2. 调用合约
const contract = new ethers.Contract(address, abi, signer);

// 读取数据 (不消耗 Gas)
const balance = await contract.balanceOf(signer.address);

// 写入数据 (消耗 Gas)
const tx2 = await contract.transfer(
  '0xRecipientAddress...',
  ethers.parseUnits('100', 18)  // 100 代币
);
await tx2.wait();

// 3. 自定义 Gas
const tx3 = await signer.sendTransaction({
  to: '0xRecipientAddress...',
  value: ethers.parseEther('0.1'),
  gasLimit: 21000,
  maxFeePerGas: ethers.parseUnits('20', 'gwei'),
  maxPriorityFeePerGas: ethers.parseUnits('2', 'gwei'),
});
```

### 查询交易

```javascript
// 查询交易详情
const tx = await provider.getTransaction(txHash);

console.log('发送者:', tx.from);
console.log('接收者:', tx.to);
console.log('金额:', ethers.formatEther(tx.value));
console.log('Gas Limit:', tx.gasLimit);
console.log('区块号:', tx.blockNumber);

// 查询交易回执
const receipt = await provider.getTransactionReceipt(txHash);

console.log('状态:', receipt.status === 1 ? '成功' : '失败');
console.log('Gas Used:', receipt.gasUsed);
console.log('区块号:', receipt.blockNumber);

// 计算交易费用
const fee = receipt.gasUsed * receipt.gasPrice;
console.log('交易费用:', ethers.formatEther(fee), 'ETH');
```

### 监听交易

```javascript
// 监听待确认交易
provider.on('pending', (txHash) => {
  console.log('新交易:', txHash);
});

// 监听区块
provider.on('block', (blockNumber) => {
  console.log('新区块:', blockNumber);
});

// 等待交易确认
const receipt = await provider.waitForTransaction(txHash, 3);  // 等待3个确认
```

## Gas 优化技巧

### 1. 选择合适的时机

```
Gas 价格波动:

高 Gas 时段 (避免):
├── 美国 9:00-17:00 EST (北京时间 22:00-6:00)
├── 热门 NFT 发售
└── DeFi 空投领取

低 Gas 时段 (推荐):
├── 周末
├── 美国深夜
└── 假期

工具:
├── https://etherscan.io/gastracker
├── https://gasnow.org
└── https://ethgasstation.info
```

### 2. 使用 Layer 2

```
Layer 2 费用对比 (2024-2025 数据)

网络          转账费用      交易费用
─────────────────────────────────────
Ethereum L1   $0.5-5       $1-50
Arbitrum      $0.01-0.1    $0.1-1
Optimism      $0.01-0.1    $0.1-1
Polygon       $0.001-0.01  $0.01-0.1
Base          $0.001-0.01  $0.01-0.1

Layer 2 节省 90-99% 的 Gas 费用
```

### 3. 批量操作

```solidity
// 单独操作: 需要多笔交易
function approveAndCall() external {
    // 每次操作都需要一笔交易
}

// 批量操作: 一笔交易完成多个操作
function batchTransfer(
    address[] calldata recipients,
    uint256[] calldata amounts
) external {
    require(recipients.length == amounts.length, "Length mismatch");
    
    for (uint256 i = 0; i < recipients.length; i++) {
        _transfer(msg.sender, recipients[i], amounts[i]);
    }
}

// 节省: 基础 Gas + 多笔交易的额外开销
```

## 下一步

[下一节：EVM 虚拟机 →](03-evm.md)
